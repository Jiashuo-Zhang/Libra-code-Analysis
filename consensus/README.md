# consensus

* Libra使用libraBFT共识算法,它基于BFT，是Hotstuff的一种变种，为了理解算法的大致流程，我们强烈推荐阅读这一部分的Libra官方的[代码说明文档](https://github.com/libra/libra/blob/master/consensus/README.md) 如果想进一步了解理论细节，可以阅读提出LibraBFT的论文[LibraBFT-paper](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/libra-consensus-state-machine-replication-in-the-libra-blockchain.pdf)。

## 目录

```
consensus
├── src
│   └── chained_bft                # Implementation of the LibraBFT protocol
│       ├── block_storage          # In-memory storage of blocks and related data structures
│       ├── consensus_types        # Consensus data types (i.e. quorum certificates)
│       ├── consensusdb            # Database interaction to persist consensus data for safety and liveness
│       ├── liveness               # Pacemaker, proposer, and other liveness related code
│       ├── safety                 # Safety (voting) rules
│       └── test_utils             # Mock implementations that are used for testing only
└── state_synchronizer             # Synchronization between validators to catch up on committed state
```

## 模块：

* **Txn_manager**:是consensus模块与mempool模块的接口，主要用于
  * 当自己是leader的时候pull出一个块来进行propose
  * 有block被commit之后，将这个block中的txns都从mempool中删除
* **State_Computer**: 是consensus模块与execution模块的接口，主要用于：
  * commit block
  * execute block
  * synchronize state
* **BlockStore**:负责维护proposal blocks, block execution, votes, quorum certificates, and persistent storage等等数据结构。这一部分支持并发，可以并发的提供服务。
* **EventProcessor**：LibraBFT本质还是事件驱动型的，这一部分就负责处理各种各样的事件，为每个事件都写了处理函数。  (e.g., process_new_round, process_proposal, process_vote). 
* **Pacemaker**：顾名思义，负责LibraBFT的liveness。他会检测超时的状况并进入一个新的round ，当他是当前round的leader时，这一部分还负责propose出一个block。
* **SafetyRules**：用于保证共识地安全性。Libra有两个vote的rule，这个模块会确保投票时遵守这两个规则。

**因为consensus的实现非常复杂，我决定从一个consensus provider的启动流程开始，比较high level的分析这些代码的功能，因此下面将只分析最核心的代码和流程，这些代码中使用的函数们一般都在更底层的代码中实现，但是我们在这里只是把他们当作API来看待 ,以后如果有必要会尽可能补上这些最底层的实现分析**

#### 启动：在consensus_providers.rs中实现

```rust
//这里实现了一个trait，所有实现了start和stop函数的struct都可以看作是一个consensus模块，事实上，在chained_bft_consensus_provider中，就是给struct ChainedBftProvider实现了这个trait，凭借这个来创造了一个基于LibraBFT的consensus模块。
//这个文件还提供了三个函数，用来创建与mempool，execution，storage进行通信三个客户端，用于与这三个部分进行交互。

/// Public interface to a consensus protocol.
pub trait ConsensusProvider {
    /// Spawns new threads, starts the consensus operations (retrieve txns, consensus protocol,
    /// execute txns, commit txns, update txn status in the mempool, etc).
    /// The function returns after consensus has recovered its initial state,
    /// and has established the required connections (e.g., to mempool and
    /// executor).
    fn start(&mut self) -> Result<()>;

    /// Stop the consensus operations. The function returns after graceful shutdown.
    fn stop(&mut self);
}


/// Helper function to create a ConsensusProvider based on configuration
////这里实际上就是new了一个libraBFT（chainedBFTProvider），然后建立了几个client
pub fn make_consensus_provider(
    node_config: &NodeConfig,
    network_sender: ConsensusNetworkSender,
    network_receiver: ConsensusNetworkEvents,
) -> Box<dyn ConsensusProvider> {
    Box::new(ChainedBftProvider::new(
        node_config,
        network_sender,
        network_receiver,
        create_mempool_client(node_config),
        create_execution_client(node_config),
    ))
}
/// Create a mempool client assuming the mempool is running on localhost
fn create_mempool_client(config: &NodeConfig) -> Arc<MempoolClient> {
    let port = config.mempool.mempool_service_port;
    let connection_str = format!("localhost:{}", port);

    let env = Arc::new(EnvBuilder::new().name_prefix("grpc-con-mem-").build());
    Arc::new(MempoolClient::new(
        ChannelBuilder::new(env).connect(&connection_str),
    ))
}

/// Create an execution client assuming the mempool is running on localhost
fn create_execution_client(config: &NodeConfig) -> Arc<ExecutionClient> {
    let connection_str = format!("localhost:{}", config.execution.port);

    let env = Arc::new(EnvBuilder::new().name_prefix("grpc-con-exe-").build());
    Arc::new(ExecutionClient::new(
        ChannelBuilder::new(env).connect(&connection_str),
    ))
}

/// Create a storage read client based on the config
pub fn create_storage_read_client(config: &NodeConfig) -> Arc<dyn StorageRead> {
    let env = Arc::new(EnvBuilder::new().name_prefix("grpc-con-sto-").build());
    Arc::new(StorageReadServiceClient::new(
        env,
        &config.storage.address,
        config.storage.port,
    ))
}
```



```rust
//一个完整的ChainedBftProvider包括以下几个部分，包括3个client和一个SMR模块
//事实上，开启一个ChainedBftProvider就是用其他3个client去start了一个smr

/// Supports the implementation of ConsensusProvider using LibraBFT.
pub struct ChainedBftProvider {
    smr: ChainedBftSMR<Vec<SignedTransaction>, Author>,
    mempool_client: Arc<MempoolClient>,
    execution_client: Arc<ExecutionClient>,
    synchronizer_client: Arc<StateSynchronizer>,
}

//上面提到的实现接口就是这个，为ChainedBftProvider实现了一个start和stop
//可以看到start（）创建了txn_manager（用mempool_client)来实现pull block和commit txns功能
//state_computer则是用execution_client和synchronizer_client实现的，这一个功能主要是在statr_computrt.rs中实现的，后面会介绍
//最后用上面的这两个实现了最终的smr，并且start，smr的功能我们会在chained_bft_smr中具体说
impl ConsensusProvider for ChainedBftProvider {
    fn start(&mut self) -> Result<()> {
        let txn_manager = Arc::new(MempoolProxy::new(self.mempool_client.clone()));
        let state_computer = Arc::new(ExecutionProxy::new(
            self.execution_client.clone(),
            self.synchronizer_client.clone(),
        ));
        debug!("Starting consensus provider.");
        self.smr.start(txn_manager, state_computer)
    }

    fn stop(&mut self) {
        self.smr.stop();
        debug!("Consensus provider stopped.");
    }
}

```

## 总结：

**前半部分我们根据官方的文档了解算法的大致流程和consensus模块的主要组成，后半部分我们从最顶层的如何开启一个SMR入手，简要介绍了ChainedBftProvider的结构和各部分的作用，后面我们将分别介绍每一部分，并介绍这几部分是如何紧密联系在一起的。**

