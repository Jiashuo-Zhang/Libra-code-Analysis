# Libra-code-Analysis

**这个项目旨在帮助研究者们理解Libra的基础结构和了解Rust语言。我们分别描述了The Libra Blockchain的核心组成部分，并聚焦于他们是如何相互联系，组成一个具有完整功能的系统。我们在现有官方文档的基础上，关注于最为核心的框架，自顶向下地逐步深入，并最终完成代码实现层面的分析。同时，对一些繁琐晦涩的部分，我们只关心他们的作用与实现方法，而不深究其具体实现。我们希望这个项目能为希望了解Libra底层实现的研究者们提供帮助，同时填补Libra 代码分析这一领域的空白。**

## 目录：

* **[Overview](https://developers.libra.org/docs/life-of-a-transaction)**
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

## 参考资料：

* Libra 源码：<https://github.com/libra/libra>

* Libra 开发者官网：<https://developers.libra.org/>

  





