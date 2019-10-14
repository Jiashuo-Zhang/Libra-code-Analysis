# Libra-code-Analysis

**这个项目旨在帮助研究者们理解Libra的基础框架和了解Rust语言。我们分别描述了The Libra Blockchain的核心组成部分，并聚焦于他们是如何相互联系，组成一个具有完整功能的系统。我们在现有官方文档和注释的基础上，关注于最为核心的框架，自顶向下地逐步深入，并最终完成代码实现层面的分析。同时，对一些繁琐晦涩的部分，我们只关心他们的作用与实现方法，而不深究其具体实现。我们希望这个项目能为希望了解Libra底层实现的研究者们提供一部分帮助，同时填补Libra 中文代码分析这一领域的空白。**

## 背景知识：

**我们的分析建立在了解区块链基础概念（如BFT共识、智能合约、联盟链等）与Rust简单语法的基础上，另外，我们强烈推荐阅读一部分Libra官方文档——[Life-of-a-transaction](https://developers.libra.org/docs/life-of-a-transaction)，这可以帮助阅读者快速了解Libra的各个部分的大致功能以及互相之间的联系，后续的分析以这一文档为主线，逐步深入。**

## 目录：

* **[AC](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/AC/AC.md)**
* **[VM_validator](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/VM_validator/VM_validator.md)**
* **[Mempool](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/mempool)**
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/mempool/README.md)
  * [Mempool_core_mempool](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/mempool/mempool_core_mempool.md)
  * [Mempool_service](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/mempool/mempool_service.md)
  * [Shared_mempool](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/mempool/shared_mempool.md)
  * [runtime](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/mempool/runtime.md)
* **[Consensus](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/consensus)**
  * [LibraBFT-paper](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/libra-consensus-state-machine-replication-in-the-libra-blockchain.pdf)
  * [SMR](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/consensus/SMR)
    * [block_store](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/EventProcessor.md)
    * [SMR](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/SMR.md)
    * [EventProcessor](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/EventProcessor.md)
    * [Sync_nanager](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/sync_manager.md)
    * [Safety_rules](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/safety_rules.md)
    * [Pacemaker](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/consensus/SMR/Pacemaker.md)
* **[Execution](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Execution)**
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Execution/README.md)
  * [Execution_service](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Execution/execution_service%26client.md)
  * [Executor](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Execution/Executor)
    * [block_tree](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Execution/Executor/block_tree.md)
    * [blocks](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Execution/Executor/blocks.md)
* **[Storage](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/storage)**
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/storage/README.md)
  * [accumulator](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/storage/accumulator.md)
  * [Storage_service & client](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/storage/storage_service%26client.md)
  * [Event_store](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/storage/Event_store.md)
  * [Libradb](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/storage/Libradb.md)
* **[Network](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Network)**
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Network/README.md)
* **[Language](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Language)**
  * [Move paper](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Language/libra-move-a-language-with-programmable-resources.pdf)
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Language/REDAME.md)
  * [VM](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Language/VM)
    * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Language/VM/README.md)
    * [VM_runtime](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Language/VM/VM_runtime)
      * [Process_txn](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Language/VM/VM_runtime/process_txn)
      * [Txn_excutors](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/Language/VM/VM_runtime/txn_executor)
      * [VM_runtime_README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/Language/VM/VM_runtime/VM_runtime_README.md)
* **[Client](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/tree/master/client)**
  * [README](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/client/README.md)
  * [client_proxy](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/client/client_proxy.md)
  * [grpc_client](https://github.com/Jiashuo-Zhang/Libra-code-Analysis/blob/master/client/grpc_client.md)

## 说明：

**由于精力与水平有限，整个项目还有很多待完成的工作，我们随时欢迎新的贡献者的加入，共同为Libra的中文代码分析作出贡献。同时，阅读过程中如果发现错误，欢迎联系我们勘误。**

## 参考资料：

* Libra 源码：<https://github.com/libra/libra>

* Libra 开发者官网：<https://developers.libra.org/>

  





