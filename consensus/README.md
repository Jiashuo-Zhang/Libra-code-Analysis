# consensus

* Libra使用libraBFT共识算法，LibraBFT有hotstuff的特性，它不只是单纯的BFT，也不是像以太坊比特币等基于区块链的共识算法，Libra遵循 contiguous 3-chain commit rule

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

* **State computer.rs** 建立了和excution modular 的链接
* **consensus _provider.rs**：Public interface to a consensus protocolv
* **txn_manager.rs** :Proxy interface to mempool，包含了commit和pull
* **chained_bft_consensus_providers.rs**:ChainedBftProvider，实现了initialize_setup和choose_leader，算是用LibraBPT实现了consensus provider的接口

## 流程

**因为consensus的实现非常复杂，我决定从一个consensus provider的启动流程开始，比较high level的分析这些代码的功能，因此下面将只分析最核心的代码和流程，这些代码中使用的函数们一般都在更底层的代码中实现，但是我们在这里将不会分析他们的实现，而只是把他们当作提供好的API来看待 ，事实上不会影响对源码的理解，以后如果有必要会尽可能补上这些最底层的实现**

#### 启动：在consensus_providers.rs中实现

```rust
//这里实现了一个trait，所有实现了start和stop函数的struct都可以看作是一个consensus模块，事实上，在chained_bft_consensus_provider中，就是给struct ChainedBftProvider实现了这个trait，凭借这个来创造了一个基于LibraBFT的consensus模块。
//这个文件还提供了三个函数，用来创建与mempool，execution，storage进行通信三个客户端

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
//最后用上面的这两个实现了最终的smr，并且startl了，他的功能我们会在chained_bft_smr中具体说
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

