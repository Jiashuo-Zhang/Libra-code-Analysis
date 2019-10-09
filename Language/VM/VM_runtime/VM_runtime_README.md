---
id: vm-runtime
title: MoveVM Runtime
custom_edit_url: https://github.com/libra/libra/edit/master/language/vm/vm_runtime/README.md
---

# MoveVM Runtime

The MoveVM runtime is the verification and execution engine for the Move
bytecode format. The runtime is imported and loaded in 2 modes:
verification mode (by the [admission control](../../../admission_control)
and [mempool](../../../mempool) components) and execution mode (by the
[execution](../../../execution) component).

## Overview

The MoveVM runtime is a stack machine. The VM runtime receives as input a
*block* which is a list of *transaction scripts* and a *data view*. The
data view is a **read only** snapshot of the data and code in the blockchain at
a given version (i.e., block height). At the time of startup, the runtime
does not have any code or data loaded. It is effectively *“empty”*.

Every transaction executes within the context of a [Libra
account](../../stdlib/modules/libra_account.mvir)---specifically the transaction
submitter's account.  The execution of every transaction consists of three
parts: the account prologue, the transaction itself, and the account
epilogue. This is the only transaction flow known to the runtime, and it is
the only flow the runtime executes. The runtime is responsible to load the
individual transaction from the block and execute the transaction flow:

1. ***Transaction Prologue*** - in verification mode the runtime runs the
   bytecode verifier over the transaction script and executes the
   prologue defined in the [Libra account
   module](../../stdlib/modules/libra_account.mvir). The prologue is responsible
   for checking the structure of the transaction and
   rejecting obviously bad transactions. In verification mode, the runtime
   returns a status of either `success` or `failure` depending upon the
   result of running the prologue. No updates to the blockchain state are
   ever performed by the prologue.
2. ***Transaction Execution*** - in execution mode, and after verification,
   the runtime starts executing transaction-specific/client code.  A typical
   code performs updates to data in the blockchain. Execution of the
   transaction by the VM runtime produces a write set that acts as an
   atomic state change from the current state of the blockchain---received
   via the data view---to a new version that is the result of applying the
   write set.  Importantly, on-chain data is _never_ changed during the
   execution of the transaction. Further, while the write set is produced as the
   result of executing the bytecode, the changes are not applied to the global
   blockchain state by the VM---this is the responsibility of the
   [execution module](../../../execution/).
3. ***Transaction Epilogue*** - in execution mode the epilogue defined in
   the [Libra account module](../../stdlib/modules/libra_account.mvir) is
   executed to perform actions based upon the result of the execution of
   the user-submitted transaction. One example of such an action is
   debiting the gas fee for the transaction from the submitting account's
   balance.

During execution, the runtime resolves references to code by loading the
referenced code via the data view. One can think of this process as similar
to linking. Then, within the context of a block of transactions---a list of
transactions coupled with a data view---the runtime caches code and
linked and imported modules across transactions within the block.
The runtime tracks state changes (data updates) from one transaction
to the next within each block of transactions; the semantics of the
execution of a block specify that transactions are sequentially executed
and, as a consequence, state changes of previous transactions must be
visible to subsequent transactions within each block.

## Implementation Details

* The runtime top level structs are in `runtime` and `libra vm` related
  code.
* The transaction flow is implemented in the [`process_txn`](./src/process_txn.rs)
  module.
* The interpreter is implemented within the [transaction
  executor](./src/txn_executor.rs).
* Code caching logic and policies are defined under the [code
  cache](./src/code_cache/) directory.
* Runtime loaded code and the type system view for the runtime is defined
  under the [loaded data](./src/loaded_data/) directory.
* The data representation of values, and logic for write set generation can
  be found under the [value](./src/value.rs) and [data
  cache](./src/data_cache.rs) files.

## Folder Structure

```
.
├── src                 # VM Runtime files
│   ├── code_cache      # VM Runtime code cache
│   ├── loaded_data     # VM Runtime loaded data types, runtime caches over code
│   ├── unit_tests      # unit tests
├── vm_cache_map        # abstractions for the code cache
```

## This Module Interacts With

This crate is mainly used in two parts: AC and mempool use it to determine
if it should accept a transaction or not; the Executor runs the MoveVM
runtime to execute the program field in a SignedTransaction and convert
it into a TransactionOutput, which contains a writeset that the
executor need to patch to the blockchain as a side effect of this
transaction.

## 功能：

**Runtime模块对外提供了两个函数，一个是validate，一个是excute，这构成了VM的主要功能**

## 初始化与最顶层代码：

```rust
/// An instantiation of the MoveVM.
/// `code_cache` is the top level module cache that holds loaded published modules.
/// `script_cache` is the cache that stores all the scripts that have previously been invoked.
/// `publishing_option` is the publishing option that is set. This can be one of either:
/// * Locked, with a whitelist of scripts that the VM is allowed to execute. For scripts that aren't
///   in the whitelist, the VM will just reject it in `verify_transaction`.
/// * Custom scripts, which will allow arbitrary valid scripts, but no module publishing
/// * Open script and module publishing
pub struct VMRuntime<'alloc> {
    code_cache: VMModuleCache<'alloc>,
    script_cache: ScriptCache<'alloc>,
    publishing_option: VMPublishingOption,
}

impl<'alloc> VMRuntime<'alloc> {
    /// Create a new VM instance with an Arena allocator to store the modules and a `config` that
    /// contains the whitelist that this VM is allowed to execute.
    pub fn new(allocator: &'alloc Arena<LoadedModule>, config: &VMConfig) -> Self {
        VMRuntime {
            code_cache: VMModuleCache::new(allocator),
            script_cache: ScriptCache::new(allocator),
            publishing_option: config.publishing_options.clone(),
        }
    }

    /// Determine if a transaction is valid. Will return `None` if the transaction is accepted,
    /// `Some(Err)` if the VM rejects it, with `Err` as an error code. We verify the following
    /// items:
    /// 1. The signature on the `SignedTransaction` matches the public key included in the
    ///    transaction
    /// 2. The script to be executed is in the whitelist.
    /// 3. Invokes `LibraAccount.prologue`, which checks properties such as the transaction has the
    /// right sequence number and the sender has enough balance to pay for the gas. 4.
    /// Transaction arguments matches the main function's type signature. 5. Script and modules
    /// in the transaction pass the bytecode static verifier.
    ///
    /// Note: In the future. we may defer these checks to a later pass, as all the scripts we will
    /// execute are pre-verified scripts. And bytecode verification is expensive. Thus whether we
    /// want to perform this check here remains unknown.
    pub fn verify_transaction(
        &self,
        txn: SignedTransaction,
        data_view: &dyn StateView,
    ) -> Option<VMStatus> {
        debug!("[VM] Verify transaction: {:?}", txn);
        // Treat a transaction as a single block.
        let module_cache =
            BlockModuleCache::new(&self.code_cache, ModuleFetcherImpl::new(data_view));
        let data_cache = BlockDataCache::new(data_view);

        let arena = Arena::new();
        let signature_verified_txn = match txn.check_signature() {
            Ok(t) => t,
            Err(_) => return Some(VMStatus::Validation(VMValidationStatus::InvalidSignature)),
        };

        let process_txn =
            ProcessTransaction::new(signature_verified_txn, module_cache, &data_cache, &arena);
        let mode = if data_view.is_genesis() {
            ValidationMode::Genesis
        } else {
            ValidationMode::Validating
        };

        let validated_txn = match process_txn.validate(mode, &self.publishing_option) {
            Ok(validated_txn) => validated_txn,
            Err(vm_status) => {
                let res = Some(vm_status);
                report_verification_status(&res);
                return res;
            }
        };
        let res = match validated_txn.verify(&self.script_cache) {
            Ok(_) => None,
            Err(vm_status) => Some(vm_status),
        };
        ////validate和verify在process_txm中实现
        report_verification_status(&res);
        res
    }

    /// Execute a block of transactions. The output vector will have the exact same length as the
    /// input vector. The discarded transactions will be marked as `TransactionStatus::Discard` and
    /// have an empty writeset. Also the data view is immutable, and also does not have interior
    /// mutability. writes to be applied to the data view are encoded in the write set part of a
    /// transaction output.
    pub fn execute_block_transactions(
        &self,
        txn_block: Vec<SignedTransaction>,
        data_view: &dyn StateView,
    ) -> Vec<TransactionOutput> {
        execute_block(
            txn_block,
            &self.code_cache,
            &self.script_cache,
            data_view,
            &self.publishing_option,
        )
    }
}

```

## MoveVM(move_vm.rs) : a wrapper for thread safe

```rust
/// A wrapper to make VMRuntime standalone and thread safe.
#[derive(Clone)]
pub struct MoveVM {
    inner: Arc<MoveVMImpl>,
}

impl MoveVM {
    pub fn new(config: &VMConfig) -> Self {
        let inner = MoveVMImpl::new(Box::new(Arena::new()), |arena| {
            VMRuntime::new(&*arena, config)
        });
        Self {
            inner: Arc::new(inner),
        }
    }
}

impl VMVerifier for MoveVM {
    fn validate_transaction(
        &self,
        transaction: SignedTransaction,
        state_view: &dyn StateView,
    ) -> Option<VMStatus> {
        // TODO: This should be implemented as an async function.
        self.inner
            .rent(move |runtime| runtime.verify_transaction(transaction, state_view))
    }
}

impl VMExecutor for MoveVM {
    fn execute_block(
        transactions: Vec<SignedTransaction>,
        config: &VMConfig,
        state_view: &dyn StateView,
    ) -> Vec<TransactionOutput> {
        let vm = MoveVMImpl::new(Box::new(Arena::new()), |arena| {
            // XXX This means that scripts and modules are NOT tested against the whitelist! This
            // needs to be fixed.
            VMRuntime::new(&*arena, config)
        });
        vm.rent(|runtime| runtime.execute_block_transactions(transactions, state_view))
    }
}

#[test]
fn vm_thread_safe() {
    fn assert_send<T: Send>() {}
    fn assert_sync<T: Sync>() {}

    assert_send::<MoveVM>();
    assert_sync::<MoveVM>();
    assert_send::<MoveVMImpl>();
    assert_sync::<MoveVMImpl>();
}

```



