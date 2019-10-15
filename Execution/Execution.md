# Execution

###The execution component provides two primary APIs —— execute_block and commit_block

## 目录：

```
    execution
            └── execution_client   # A Rust wrapper on top of GRPC clients.
            └── execution_proto    # All interfaces provided by the execution component.
            └── execution_service  # Execution component as a GRPC service.
            └── executor           # The main implementation of execution component.
```

