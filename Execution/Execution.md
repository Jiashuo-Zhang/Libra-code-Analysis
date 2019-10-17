# Execution

**Execution是Libra中的一个重要模块，负责接收有一定顺序的交易，调用MoveVM计算每一个交易的结果，并把结果应用到当前的state里面一部分主要提供了两个API，分别是：**

* execute block
* commit block

这两部分的顶层实现我们在**executionclients&service.md**中进行了展示。



## 目录：

```
    execution
            └── execution_client   # A Rust wrapper on top of GRPC clients.
            └── execution_proto    # All interfaces provided by the execution component.
            └── execution_service  # Execution component as a GRPC service.
            └── executor           # The main implementation of execution component.
```

