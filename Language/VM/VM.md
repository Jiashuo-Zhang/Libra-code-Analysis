# VM

## 功能：

**Libra Core components interact with the language component through the VM. Specifically, the admission control component uses a limited, read-only subset of the VM functionality to discard invalid transactions before they are admitted to the mempool and consensus(这一部分我们单独列在了VM_validator文件夹下). The execution component uses the VM to execute a block of transactions.(我们把这一部分称作MoveVM Core)**

##MoveVM Core:

The MoveVM executes transactions expressed in the Move bytecode. There are
two main crates: the core VM and the VM runtime. The VM core contains the low-level
data type for the VM - mostly the file format and abstraction over it. A gas
metering logical abstraction is also defined there.

## 目录

```
├── cost_synthesis  # Infrastructure for gas cost synthesis
├── src             # VM core files
├── tests           # Proptests
├── vm_genesis      # Helpers to generate a genesis block, the initial state of the blockchain
└── vm_runtime      # Interpreter and runtime data types (see README in that folder)
```







