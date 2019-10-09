# process_txn

## 功能：

**这一部分提供执行处理执行txn的功能，位于 ⁨libra-master⁩/language⁩/vm⁩/vm_runtime⁩/src⁩/process_txn文件夹。The transaction flow is implemented in the process_txn module.这里要求txn是verified**

## 代码：（自顶向下的

####最顶层的wrapper，包括了创建一个process_txn和validate的顶层接口：

```rust
/// The starting point for processing a transaction. All the different states involved are described
/// through the types present in submodules.
pub struct ProcessTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    txn: SignatureCheckedTransaction,
    module_cache: P,
    data_cache: &'txn dyn RemoteCache,
    allocator: &'txn Arena<LoadedModule>,
    phantom: PhantomData<&'alloc ()>,
}

impl<'alloc, 'txn, P> ProcessTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    /// Creates a new instance of `ProcessTransaction`.
    pub fn new(
        txn: SignatureCheckedTransaction,
        module_cache: P,
        data_cache: &'txn dyn RemoteCache,
        allocator: &'txn Arena<LoadedModule>,
    ) -> Self {
        Self {
            txn,
            module_cache,
            data_cache,
            allocator,
            phantom: PhantomData,
        }
    }

    /// Validates this transaction. Returns a `ValidatedTransaction` on success or `VMStatus` on
    /// failure.
    pub fn validate(
        self,
        mode: ValidationMode,
        publishing_option: &VMPublishingOption,
    ) -> Result<ValidatedTransaction<'alloc, 'txn, P>, VMStatus> {
        ValidatedTransaction::new(self, mode, publishing_option)
    }
}

```

#### execute部分：

```rust
/// Represents a transaction that has been executed.
pub struct ExecutedTransaction {
    output: TransactionOutput,
}
///这里很有意思的是，这个struct里面并没有执行交易所需要的信息，而是包含了交易执行的output，也就是树交易的执行是在这个结构体new的过程中实现的。
impl ExecutedTransaction {
    /// Creates a new instance by executing this transaction.
    pub fn new<'alloc, 'txn, P>(verified_txn: VerifiedTransaction<'alloc, 'txn, P>) -> Self
    where
        'alloc: 'txn,
        P: ModuleCache<'alloc>,
    {
        let output = execute(verified_txn);///核心就是excute函数
        Self { output }
    }

    /// Returns the `TransactionOutput` for this transaction.
    pub fn into_output(self) -> TransactionOutput {
        self.output
    }
}
////上面调用的excute函数
fn execute<'alloc, 'txn, P>(
    mut verified_txn: VerifiedTransaction<'alloc, 'txn, P>,
) -> TransactionOutput
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    let txn_state = verified_txn.take_state();

    match verified_txn
        .into_inner()
        .into_raw_transaction()
        .into_payload()
    {
        TransactionPayload::Program(program) => {
            let VerifiedTransactionState {
                mut txn_executor,
                main,
                modules,
            } = txn_state.expect("program-based transactions should always have associated state");

            let (_, args, module_bytes) = program.into_inner();
/////把所有的module都load进cache，如果有重名的会失败
            // Add modules to the cache and prepare for publishing.
            let mut publish_modules = vec![];
            for (module, raw_bytes) in modules.into_iter().zip(module_bytes) {
                let module_id = module.self_id();

                // Make sure that there is not already a module with this name published
                // under the transaction sender's account.
                // Note: although this reads from the "module cache", `get_loaded_module`
                // will read through the cache to fetch the module from the global storage
                // if it is not already cached.
                match txn_executor.module_cache().get_loaded_module(&module_id) {
                    Ok(Ok(None)) => (), // No module with this name exists. safe to publish one
                    Ok(Ok(Some(_))) | Ok(Err(_)) => {
                        // A module with this name already exists (the error case is when the module
                        // couldn't be verified, but it still exists so we should fail similarly).
                        // It is not safe to publish another one; it would clobber the old module.
                        // This would break code that links against the module and make published
                        // resources from the old module inaccessible (or worse, accessible and not
                        // typesafe).
                        //
                        // We are currently developing a versioning scheme for safe updates of
                        // modules and resources.
                        warn!("[VM] VM error duplicate module {:?}", module_id);
                        return txn_executor.failed_transaction_cleanup(Ok(Err(VMRuntimeError {
                            loc: Location::default(),
                            err: VMErrorKind::DuplicateModuleName,
                        })));
                    }
                    Err(err) => {
                        error!(
                            "[VM] VM internal error while checking for duplicate module {:?}: {:?}",
                            module_id, err
                        );
                        return ExecutedTransaction::discard_error_output(&err);
                    }
                }

                txn_executor.module_cache().cache_module(module);
                publish_modules.push((module_id, raw_bytes));
            }

            // Set up main.
            txn_executor.setup_main_args(args);

            // Run main.
            match txn_executor.execute_function_impl(main) {
                Ok(Ok(_)) => txn_executor.transaction_cleanup(publish_modules),
                Ok(Err(err)) => {
                    warn!("[VM] User error running script: {:?}", err);
                    txn_executor.failed_transaction_cleanup(Ok(Err(err)))
                }
                Err(err) => {
                    error!("[VM] VM error running script: {:?}", err);
                    ExecutedTransaction::discard_error_output(&err)
                }
            }
        }
        // WriteSet transaction. Just proceed and use the writeset as output.
        TransactionPayload::WriteSet(write_set) => TransactionOutput::new(
            write_set,
            vec![],
            0,
            VMStatus::Execution(ExecutionStatus::Executed).into(),
        ),
    }
}

impl ExecutedTransaction {
    #[inline]
    pub(crate) fn discard_error_output(err: impl Into<VMStatus>) -> TransactionOutput {
        // Since this transaction will be discarded, no writeset will be included.
        TransactionOutput::new(
            WriteSet::default(),
            vec![],
            0,
            TransactionStatus::Discard(err.into()),
        )
    }
}

```

####Validator: 对于gas，script，version等等的一系列检查

```rust
/// Represents a [`SignedTransaction`] that has been *validated*. This includes all the steps
/// required to ensure that a transaction is valid, other than verifying the submitted program.
pub struct ValidatedTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    txn: SignatureCheckedTransaction,
    txn_state: Option<ValidatedTransactionState<'alloc, 'txn, P>>,
}

/// The mode to validate transactions in.
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum ValidationMode {
    /// This is the genesis transaction. At the moment it is the only mode that allows for
    /// write-set transactions.
    Genesis,
    /// We're only validating a transaction, not executing it. This tolerates the sequence number
    /// being too new.
    Validating,
    /// We're executing a transaction. This runs the full suite of checks.
    #[allow(dead_code)]
    Executing,
}

impl<'alloc, 'txn, P> ValidatedTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    /// Creates a new instance by validating a `SignedTransaction`.
    ///
    /// This should be called through [`ProcessTransaction::validate`].
    pub(super) fn new(
        process_txn: ProcessTransaction<'alloc, 'txn, P>,
        mode: ValidationMode,
        publishing_option: &VMPublishingOption,
    ) -> Result<Self, VMStatus> {
        let ProcessTransaction {
            txn,
            module_cache,
            data_cache,
            allocator,
            ..
        } = process_txn;

        let txn_state = match txn.payload() {
            TransactionPayload::Program(program) => {
                // The transaction is too large.////如果是一个program的话
                if txn.raw_txn_bytes_len() > MAX_TRANSACTION_SIZE_IN_BYTES {
                    let error_str = format!(
                        "max size: {}, txn size: {}",
                        MAX_TRANSACTION_SIZE_IN_BYTES,
                        txn.raw_txn_bytes_len()
                    );
                    warn!(
                        "[VM] Transaction size too big {} (max {})",
                        txn.raw_txn_bytes_len(),
                        MAX_TRANSACTION_SIZE_IN_BYTES
                    );
                    return Err(VMStatus::Validation(
                        VMValidationStatus::ExceededMaxTransactionSize(error_str),
                    ));
                }

                // The submitted max gas units that the transaction can consume is greater than the
                // maximum number of gas units bound that we have set for any
                // transaction.
                if txn.max_gas_amount() > gas_schedule::MAXIMUM_NUMBER_OF_GAS_UNITS {
                    let error_str = format!(
                        "max gas units: {}, gas units submitted: {}",
                        gas_schedule::MAXIMUM_NUMBER_OF_GAS_UNITS,
                        txn.max_gas_amount()
                    );
                    warn!(
                        "[VM] Gas unit error; max {}, submitted {}",
                        gas_schedule::MAXIMUM_NUMBER_OF_GAS_UNITS,
                        txn.max_gas_amount()
                    );
                    return Err(VMStatus::Validation(
                        VMValidationStatus::MaxGasUnitsExceedsMaxGasUnitsBound(error_str),
                    ));
                }

                // The submitted transactions max gas units needs to be at least enough to cover the
                // intrinsic cost of the transaction as calculated against the size of the
                // underlying `RawTransaction`
                let min_txn_fee =
                    gas_schedule::calculate_intrinsic_gas(txn.raw_txn_bytes_len() as u64);
                if txn.max_gas_amount() < min_txn_fee {
                    let error_str = format!(
                        "min gas required for txn: {}, gas submitted: {}",
                        min_txn_fee,
                        txn.max_gas_amount()
                    );
                    warn!(
                        "[VM] Gas unit error; min {}, submitted {}",
                        min_txn_fee,
                        txn.max_gas_amount()
                    );
                    return Err(VMStatus::Validation(
                        VMValidationStatus::MaxGasUnitsBelowMinTransactionGasUnits(error_str),
                    ));
                }

                // The submitted gas price is less than the minimum gas unit price set by the VM.
                // NB: MIN_PRICE_PER_GAS_UNIT may equal zero, but need not in the future. Hence why
                // we turn off the clippy warning.
                #[allow(clippy::absurd_extreme_comparisons)]
                let below_min_bound = txn.gas_unit_price() < gas_schedule::MIN_PRICE_PER_GAS_UNIT;
                if below_min_bound {
                    let error_str = format!(
                        "gas unit min price: {}, submitted price: {}",
                        gas_schedule::MIN_PRICE_PER_GAS_UNIT,
                        txn.gas_unit_price()
                    );
                    warn!(
                        "[VM] Gas unit error; min {}, submitted {}",
                        gas_schedule::MIN_PRICE_PER_GAS_UNIT,
                        txn.gas_unit_price()
                    );
                    return Err(VMStatus::Validation(
                        VMValidationStatus::GasUnitPriceBelowMinBound(error_str),
                    ));
                }

                // The submitted gas price is greater than the maximum gas unit price set by the VM.
                if txn.gas_unit_price() > gas_schedule::MAX_PRICE_PER_GAS_UNIT {
                    let error_str = format!(
                        "gas unit max price: {}, submitted price: {}",
                        gas_schedule::MAX_PRICE_PER_GAS_UNIT,
                        txn.gas_unit_price()
                    );
                    warn!(
                        "[VM] Gas unit error; min {}, submitted {}",
                        gas_schedule::MAX_PRICE_PER_GAS_UNIT,
                        txn.gas_unit_price()
                    );
                    return Err(VMStatus::Validation(
                        VMValidationStatus::GasUnitPriceAboveMaxBound(error_str),
                    ));
                }
//////////////上面的是关于gas的硬性规定，防止有人恶意攻击等以及减少gas不足中途返回的情况
                
                ////检查script能否使用
                // Verify against whitelist if we are locked. Otherwise allow.
                if !is_allowed_script(&publishing_option, &program.code()) {
                    warn!("[VM] Custom scripts not allowed: {:?}", &program.code());
                    return Err(VMStatus::Validation(VMValidationStatus::UnknownScript));
                }

                if !publishing_option.is_open() {
                    // Not allowing module publishing for now.
                    if !program.modules().is_empty() {
                        warn!("[VM] Custom modules not allowed");
                        return Err(VMStatus::Validation(VMValidationStatus::UnknownModule));
                    }
                }

                
                let metadata = TransactionMetadata::new(&txn);
                let mut txn_state =
                    ValidatedTransactionState::new(metadata, module_cache, data_cache, allocator);

                // Run the prologue to ensure that clients have enough gas and aren't tricking us by
                // sending us garbage.
                // TODO: write-set transactions (other than genesis??) should also run the prologue.
                ////执行的第一个阶段————prologue
                
                match txn_state.txn_executor.run_prologue() {
                    Ok(Ok(_)) => {}
                    Ok(Err(ref err)) => {
                        let vm_status = convert_prologue_runtime_error(&err, &txn.sender());

                        // In validating mode, accept transactions with sequence number greater
                        // or equal to the current sequence number.
                        match (mode, vm_status) {
                            (
                                ValidationMode::Validating,
                                VMStatus::Validation(VMValidationStatus::SequenceNumberTooNew),
                            ) => {
                                trace!("[VM] Sequence number too new error ignored");
                            }
                            (_, vm_status) => {
                                warn!("[VM] Error in prologue: {:?}", err);
                                return Err(vm_status);
                            }
                        }
                    }
                    Err(ref err) => {
                        error!("[VM] VM internal error in prologue: {:?}", err);
                        return Err(err.into());
                    }
                };

                Some(txn_state)
            }
            TransactionPayload::WriteSet(write_set) => {
                // The only acceptable write-set transaction for now is for the genesis
                // transaction.
                // XXX figure out a story for hard forks.
                if mode != ValidationMode::Genesis {
                    warn!("[VM] Attempt to process genesis after initialization");
                    return Err(VMStatus::Validation(VMValidationStatus::RejectedWriteSet));
                }

                for (_access_path, write_op) in write_set {
                    // Genesis transactions only add entries, never delete them.
                    if write_op.is_deletion() {
                        error!("[VM] Bad genesis block");
                        // TODO: return more detailed error somehow?
                        return Err(VMStatus::Validation(VMValidationStatus::InvalidWriteSet));
                    }
                }

                None
            }
        };

        Ok(Self { txn, txn_state })
    }

    /// Verifies the bytecode in this transaction.
    pub fn verify(
        self,
        script_cache: &'txn ScriptCache<'alloc>,
    ) -> Result<VerifiedTransaction<'alloc, 'txn, P>, VMStatus> {
        VerifiedTransaction::new(self, script_cache)
    }

    /// Returns a reference to the `SignatureCheckedTransaction` within.
    pub fn as_inner(&self) -> &SignatureCheckedTransaction {
        &self.txn
    }

    /// Consumes `self` and returns the `SignatureCheckedTransaction` within.
    #[allow(dead_code)]
    pub fn into_inner(self) -> SignatureCheckedTransaction {
        self.txn
    }

    /// Returns the `ValidatedTransactionState` within.
    pub(super) fn take_state(&mut self) -> Option<ValidatedTransactionState<'alloc, 'txn, P>> {
        self.txn_state.take()
    }
}

/// State for program-based [`ValidatedTransaction`] instances.
pub(super) struct ValidatedTransactionState<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    // <'txn, 'txn> looks weird, but it just means that the module cache passed in (the
    // TransactionModuleCache) allocates for that long.
    pub(super) txn_executor:
        TransactionExecutor<'txn, 'txn, TransactionModuleCache<'alloc, 'txn, P>>,
}

impl<'alloc, 'txn, P> ValidatedTransactionState<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    fn new(
        metadata: TransactionMetadata,
        module_cache: P,
        data_cache: &'txn dyn RemoteCache,
        allocator: &'txn Arena<LoadedModule>,
    ) -> Self {
        // This temporary cache is used for modules published by a single transaction.
        let txn_module_cache = TransactionModuleCache::new(module_cache, allocator);
        let txn_executor = TransactionExecutor::new(txn_module_cache, data_cache, metadata);
        Self { txn_executor }
    }
}

```

#### verify:在这过程中调用了bytecode verifier

```rust
/// Represents a transaction which has been validated and for which the program has been run
/// through the bytecode verifier.
pub struct VerifiedTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    txn: SignatureCheckedTransaction,
    #[allow(dead_code)]
    txn_state: Option<VerifiedTransactionState<'alloc, 'txn, P>>,
}

impl<'alloc, 'txn, P> VerifiedTransaction<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    /// Creates a new instance by verifying the bytecode in this validated transaction.
    pub(super) fn new(
        mut validated_txn: ValidatedTransaction<'alloc, 'txn, P>,
        script_cache: &'txn ScriptCache<'alloc>,
    ) -> Result<Self, VMStatus> {
        let txn_state = validated_txn.take_state();
        let txn = validated_txn.as_inner();
        let txn_state = match txn.payload() {
            TransactionPayload::Program(program) => {
                let txn_state = txn_state
                    .expect("program-based transactions should always have associated state");

                let (main, modules) = Self::verify_program(&txn.sender(), program, script_cache)?;

                Some(VerifiedTransactionState {
                    txn_executor: txn_state.txn_executor,
                    main,
                    modules,
                })
            }
            TransactionPayload::WriteSet(_write_set) => {
                // All the checks are performed in validation, so there's no need for more checks///记得validator的时候已经有验证了
                // here.
                None
            }
        };

        Ok(Self {
            txn: validated_txn.into_inner(),
            txn_state,
        })
    }

    ////上面调用的函数的实现，会调用bytecodes verifier
    fn verify_program(
        sender_address: &AccountAddress,
        program: &Program,
        script_cache: &'txn ScriptCache<'alloc>,
    ) -> Result<(FunctionRef<'alloc>, Vec<VerifiedModule>), VMStatus> {
        // Ensure the script can correctly be resolved into main.
        let main = match script_cache.cache_script(&program.code()) {
            Ok(Ok(main)) => main,
            Ok(Err(ref err)) => return Err(err.into()),
            Err(ref err) => return Err(err.into()),
        };

        if !verify_actuals(main.signature(), program.args()) {
            return Err(VMStatus::Verification(vec![VMVerificationStatus::Script(
                VMVerificationError::TypeMismatch("Actual Type Mismatch".to_string()),
            )]));
        }

        // Make sure all the modules trying to be published in this module are valid.
        let modules: Vec<CompiledModule> = match program
            .modules()
            .iter()
            .map(|module_blob| CompiledModule::deserialize(&module_blob))
            .collect()
        {
            Ok(modules) => modules,
            Err(ref err) => {
                warn!("[VM] module deserialization failed {:?}", err);
                return Err(err.into());
            }
        };

        // Run the modules through the bytecode verifier.
        let modules = match static_verify_modules(sender_address, modules) {
            Ok(modules) => modules,
            Err(statuses) => {
                warn!("[VM] bytecode verifier returned errors");
                return Err(statuses.iter().collect());
            }
        };

        Ok((main, modules))
    }

    /// Executes this transaction.
    pub fn execute(self) -> ExecutedTransaction {
        ExecutedTransaction::new(self)
    }

    /// Returns the state stored in the transaction, if any.
    pub(super) fn take_state(&mut self) -> Option<VerifiedTransactionState<'alloc, 'txn, P>> {
        self.txn_state.take()
    }

    /// Returns a reference to the `SignatureCheckedTransaction` within.
    #[allow(dead_code)]
    pub fn as_inner(&self) -> &SignatureCheckedTransaction {
        &self.txn
    }

    /// Consumes `self` and returns the `SignatureCheckedTransaction` within.
    pub fn into_inner(self) -> SignatureCheckedTransaction {
        self.txn
    }
}

/// State for program-based [`VerifiedTransaction`] instances.
#[allow(dead_code)]
pub(super) struct VerifiedTransactionState<'alloc, 'txn, P>
where
    'alloc: 'txn,
    P: ModuleCache<'alloc>,
{
    pub(super) txn_executor:
        TransactionExecutor<'txn, 'txn, TransactionModuleCache<'alloc, 'txn, P>>,
    pub(super) main: FunctionRef<'alloc>,
    pub(super) modules: Vec<VerifiedModule>,
}

fn static_verify_modules(
    sender_address: &AccountAddress,
    modules: Vec<CompiledModule>,
) -> Result<Vec<VerifiedModule>, Vec<VerificationStatus>> {
    // It is possible to write this function without the expects, but that makes it very ugly.
    let mut statuses: Vec<Box<dyn Iterator<Item = VerificationStatus>>> = vec![];

    let modules_len = modules.len();

    let mut modules_out = vec![];
    for (module_idx, module) in modules.into_iter().enumerate() {
        // Make sure the module's self address matches the transaction sender. The self address is
        // where the module will actually be published. If we did not check this, the sender could
        // publish a module under anyone's account.
        //
        // For scripts this isn't a problem because they don't get published to accounts.
       ///publish的规则
        let self_error = if module.address() != sender_address {
            Some(VerificationError {
                kind: IndexKind::AddressPool,
                idx: CompiledModule::IMPLEMENTED_MODULE_INDEX as usize,
                err: VMStaticViolation::ModuleAddressDoesNotMatchSender,
            })
        } else {
            None
        };

        let (module, mut errors) = match VerifiedModule::new(module) {
            Ok(module) => (Some(module), vec![]),
            Err((_, errors)) => (None, errors),
        };

        if let Some(error) = self_error {
            errors.push(error);
        }

        if errors.is_empty() {
            modules_out.push(module.expect("empty errors => module should verify"));
        } else {
            statuses.push(Box::new(errors.into_iter().map(move |error| {
                VerificationStatus::Module(module_idx as u16, error)
            })));
        }
    }

    let statuses: Vec<_> = statuses.into_iter().flatten().collect();
    if statuses.is_empty() {
        assert_eq!(modules_out.len(), modules_len);
        Ok(modules_out)
    } else {
        Err(statuses)
    }
}

/// Run static checks on a program directly. Provided as an alternative API for tests.
pub fn static_verify_program(
    sender_address: &AccountAddress,
    script: CompiledScript,
    modules: Vec<CompiledModule>,
) -> Result<(VerifiedScript, Vec<VerifiedModule>), Vec<VerificationStatus>> {
    // It is possible to write this function without the expects, but that makes it very ugly.
    let mut statuses: Vec<VerificationStatus> = vec![];
    let script = match VerifiedScript::new(script) {
        Ok(script) => Some(script),
        Err((_, errors)) => {
            statuses.extend(errors.into_iter().map(VerificationStatus::Script));
            None
        }
    };

    let modules = match static_verify_modules(sender_address, modules) {
        Ok(modules) => Some(modules),
        Err(module_statuses) => {
            statuses.extend(module_statuses);
            None
        }
    };

    if statuses.is_empty() {
        Ok((
            script.expect("Ok case => script should verify"),
            modules.expect("Ok case => modules should verify"),
        ))
    } else {
        Err(statuses)
    }
}

/// Verify if the transaction arguments match the type signature of the main function.
fn verify_actuals(signature: &FunctionSignature, args: &[TransactionArgument]) -> bool {
    if signature.arg_types.len() != args.len() {
        warn!(
            "[VM] different argument length: actuals {}, formals {}",
            args.len(),
            signature.arg_types.len()
        );
        return false;
    }
    for (ty, arg) in signature.arg_types.iter().zip(args.iter()) {
        match (ty, arg) {
            (SignatureToken::U64, TransactionArgument::U64(_)) => (),
            (SignatureToken::Address, TransactionArgument::Address(_)) => (),
            (SignatureToken::ByteArray, TransactionArgument::ByteArray(_)) => (),
            (SignatureToken::String, TransactionArgument::String(_)) => (),
            _ => {
                warn!(
                    "[VM] different argument type: formal {:?}, actual {:?}",
                    ty, arg
                );
                return false;
            }
        }
    }
    true
}
/////有3个直接的verifier，分别是对program，和modular，actuals
```



