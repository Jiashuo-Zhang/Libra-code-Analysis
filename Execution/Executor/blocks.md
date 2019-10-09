# blocks

## 功能：

**为了简介一点，这一部分包括transactions_blocks.rs 和block_processor.rs 两个部分** 。这两个部分注视比较详细

## 代码：

#### transaction_block 的定义和简单接口们：  同时他还实现了之前提到的block trait的接口，这里面值得注意的函数是有关commit_response和 execute_response 的send和set等，应用场景是：consensus发来消息commit了某个block，那我们操作完commit之后要发送response到sonsensus，这几个函数就实现了这种的功能

```rust
/// `TransactionBlock` holds everything about the block of transactions.
#[derive(Debug)]
pub struct TransactionBlock {
    /// Whether consensus has decided to commit this block.
    committed: bool,

    /// Id of this block.
    id: HashValue,

    /// Id of the parent block.
    parent_id: HashValue,

    /// The set of children.
    children: HashSet<HashValue>,

    /// The transactions themselves.
    transactions: Vec<SignedTransaction>,

    /// The result of processing VM's output.
    output: Option<ProcessedVMOutput>,

    /// The signatures on this block. Not all committed blocks will have signatures, if multiple
    /// blocks are committed at once.
    ledger_info_with_sigs: Option<LedgerInfoWithSignatures>,

    /// The response for `execute_block` request.
    execute_response: Option<ExecuteBlockResponse>,

    /// The senders associated with this block. These senders are like the promises associated with
    /// the futures returned by `execute_block` and `commit_block` APIs, which are fulfilled when
    /// the responses are ready.
    execute_response_senders: Vec<oneshot::Sender<Result<ExecuteBlockResponse>>>,
    commit_response_sender: Option<oneshot::Sender<Result<CommitBlockResponse>>>,
}

impl TransactionBlock {
    /// Constructs a new block. A `TransactionBlock` is constructed as soon as consensus gives us a
    /// new block. It has not been executed yet so output is `None`.
    pub fn new(
        transactions: Vec<SignedTransaction>,
        parent_id: HashValue,
        id: HashValue,
        execute_response_sender: oneshot::Sender<Result<ExecuteBlockResponse>>,
    ) -> Self {
        TransactionBlock {
            committed: false,
            id,
            parent_id,
            children: HashSet::new(),
            transactions,
            output: None,
            ledger_info_with_sigs: None,
            execute_response: None,
            execute_response_senders: vec![execute_response_sender],
            commit_response_sender: None,
        }
    }

    /// Returns the list of transactions.
    pub fn transactions(&self) -> &[SignedTransaction] {
        &self.transactions
    }

    /// Returns the output of the block.
    pub fn output(&self) -> &Option<ProcessedVMOutput> {
        &self.output
    }

    /// Returns the signatures on this block.
    pub fn ledger_info_with_sigs(&self) -> &Option<LedgerInfoWithSignatures> {
        &self.ledger_info_with_sigs
    }

    /// Saves the response in the block. If there are any queued senders, send the response.
    pub fn set_execute_block_response(&mut self, response: ExecuteBlockResponse) {
        assert!(self.execute_response.is_none(), "Response is already set.");
        self.execute_response = Some(response.clone());
        // Send the response since it's now available.
        self.send_execute_block_response(Ok(response));
    }

    /// Puts a sender in the queue. The response will be sent via the sender once available
    /// (possibly as soon as the function is called if the response if already available).
    pub fn queue_execute_block_response_sender(
        &mut self,
        sender: oneshot::Sender<Result<ExecuteBlockResponse>>,
    ) {
        // If the response is already available, just send it. Otherwise store the sender for later
        // use.
        match self.execute_response {
            Some(ref response) => {
                if let Err(_err) = sender.send(Ok(response.clone())) {
                    warn!("Failed to send execute block response.");
                }
            }
            None => self.execute_response_senders.push(sender),
        }
    }

    /// Sends finished `ExecuteBlockResponse` to consensus. This removes all the existing senders.
    pub fn send_execute_block_response(&mut self, response: Result<ExecuteBlockResponse>) {
        while let Some(sender) = self.execute_response_senders.pop() {
            // We need to send the result multiple times, but the error is not cloneable, thus the
            // result is not cloneable. This is a bit workaround.
            let resp = match &response {
                Ok(resp) => Ok(resp.clone()),
                Err(err) => Err(format_err!("{}", err)),
            };
            if let Err(_err) = sender.send(resp) {
                warn!("Failed to send execute block response.");
            }
        }
    }

    /// When the block is created, no one has called `commit_block` on this block yet, so we do not
    /// have the sender and `self.commit_response_sender` is initialized to `None`. When consensus
    /// calls `commit_block` on a block, we will put the sender in the block. So when this block is
    /// persisted in storage later, we will call `send_commit_response` and consensus will receive
    /// the response.
    pub fn set_commit_response_sender(
        &mut self,
        commit_response_sender: oneshot::Sender<Result<CommitBlockResponse>>,
    ) {
        assert!(
            self.commit_response_sender.is_none(),
            "CommitBlockResponse sender should not exist."
        );
        self.commit_response_sender = Some(commit_response_sender);
    }

    /// Sends finished `CommitBlockResponse` to consensus.
    pub fn send_commit_block_response(&mut self, response: Result<CommitBlockResponse>) {
        let sender = self
            .commit_response_sender
            .take()
            .expect("CommitBlockResponse sender should exist.");
        if let Err(_err) = sender.send(response) {
            warn!("Failed to send commit block response:.");
        }
    }

    /// Returns a pointer to the Sparse Merkle Tree representing the state at the end of the block.
    /// Should only be called when the block has finished execution and `set_output` has been
    /// called.
    pub fn clone_state_tree(&self) -> Rc<SparseMerkleTree> {
        self.output
            .as_ref()
            .expect("The block has no output yet.")
            .clone_state_tree()
    }

    /// Returns a pointer to the Merkle Accumulator representing the end of the block. Should only
    /// be called when the block has finished execution and `set_output` has been called.
    pub fn clone_transaction_accumulator(&self) -> Rc<Accumulator<TransactionAccumulatorHasher>> {
        self.output
            .as_ref()
            .expect("The block has no output yet.")
            .clone_transaction_accumulator()
    }
}
/// `TransactionBlock` holds everything about the block of transactions.
#[derive(Debug)]
pub struct TransactionBlock {
    /// Whether consensus has decided to commit this block.
    committed: bool,

    /// Id of this block.
    id: HashValue,

    /// Id of the parent block.
    parent_id: HashValue,

    /// The set of children.
    children: HashSet<HashValue>,

    /// The transactions themselves.
    transactions: Vec<SignedTransaction>,

    /// The result of processing VM's output.
    output: Option<ProcessedVMOutput>,

    /// The signatures on this block. Not all committed blocks will have signatures, if multiple
    /// blocks are committed at once.
    ledger_info_with_sigs: Option<LedgerInfoWithSignatures>,

    /// The response for `execute_block` request.
    execute_response: Option<ExecuteBlockResponse>,

    /// The senders associated with this block. These senders are like the promises associated with
    /// the futures returned by `execute_block` and `commit_block` APIs, which are fulfilled when
    /// the responses are ready.
    execute_response_senders: Vec<oneshot::Sender<Result<ExecuteBlockResponse>>>,
    commit_response_sender: Option<oneshot::Sender<Result<CommitBlockResponse>>>,
}

impl TransactionBlock {
    /// Constructs a new block. A `TransactionBlock` is constructed as soon as consensus gives us a
    /// new block. It has not been executed yet so output is `None`.
    pub fn new(
        transactions: Vec<SignedTransaction>,
        parent_id: HashValue,
        id: HashValue,
        execute_response_sender: oneshot::Sender<Result<ExecuteBlockResponse>>,
    ) -> Self {
        TransactionBlock {
            committed: false,
            id,
            parent_id,
            children: HashSet::new(),
            transactions,
            output: None,
            ledger_info_with_sigs: None,
            execute_response: None,
            execute_response_senders: vec![execute_response_sender],
            commit_response_sender: None,
        }
    }

    /// Returns the list of transactions.
    pub fn transactions(&self) -> &[SignedTransaction] {
        &self.transactions
    }

    /// Returns the output of the block.
    pub fn output(&self) -> &Option<ProcessedVMOutput> {
        &self.output
    }

    /// Returns the signatures on this block.
    pub fn ledger_info_with_sigs(&self) -> &Option<LedgerInfoWithSignatures> {
        &self.ledger_info_with_sigs
    }

    /// Saves the response in the block. If there are any queued senders, send the response.
    pub fn set_execute_block_response(&mut self, response: ExecuteBlockResponse) {
        assert!(self.execute_response.is_none(), "Response is already set.");
        self.execute_response = Some(response.clone());
        // Send the response since it's now available.
        self.send_execute_block_response(Ok(response));
    }

    /// Puts a sender in the queue. The response will be sent via the sender once available
    /// (possibly as soon as the function is called if the response if already available).
    pub fn queue_execute_block_response_sender(
        &mut self,
        sender: oneshot::Sender<Result<ExecuteBlockResponse>>,
    ) {
        // If the response is already available, just send it. Otherwise store the sender for later
        // use.
        match self.execute_response {
            Some(ref response) => {
                if let Err(_err) = sender.send(Ok(response.clone())) {
                    warn!("Failed to send execute block response.");
                }
            }
            None => self.execute_response_senders.push(sender),
        }
    }

    /// Sends finished `ExecuteBlockResponse` to consensus. This removes all the existing senders.
    pub fn send_execute_block_response(&mut self, response: Result<ExecuteBlockResponse>) {
        while let Some(sender) = self.execute_response_senders.pop() {
            // We need to send the result multiple times, but the error is not cloneable, thus the
            // result is not cloneable. This is a bit workaround.
            let resp = match &response {
                Ok(resp) => Ok(resp.clone()),
                Err(err) => Err(format_err!("{}", err)),
            };
            if let Err(_err) = sender.send(resp) {
                warn!("Failed to send execute block response.");
            }
        }
    }

    /// When the block is created, no one has called `commit_block` on this block yet, so we do not
    /// have the sender and `self.commit_response_sender` is initialized to `None`. When consensus
    /// calls `commit_block` on a block, we will put the sender in the block. So when this block is
    /// persisted in storage later, we will call `send_commit_response` and consensus will receive
    /// the response.
    pub fn set_commit_response_sender(
        &mut self,
        commit_response_sender: oneshot::Sender<Result<CommitBlockResponse>>,
    ) {
        assert!(
            self.commit_response_sender.is_none(),
            "CommitBlockResponse sender should not exist."
        );
        self.commit_response_sender = Some(commit_response_sender);
    }

    /// Sends finished `CommitBlockResponse` to consensus.
    pub fn send_commit_block_response(&mut self, response: Result<CommitBlockResponse>) {
        let sender = self
            .commit_response_sender
            .take()
            .expect("CommitBlockResponse sender should exist.");
        if let Err(_err) = sender.send(response) {
            warn!("Failed to send commit block response:.");
        }
    }

    /// Returns a pointer to the Sparse Merkle Tree representing the state at the end of the block.
    /// Should only be called when the block has finished execution and `set_output` has been
    /// called.
    pub fn clone_state_tree(&self) -> Rc<SparseMerkleTree> {
        self.output
            .as_ref()
            .expect("The block has no output yet.")
            .clone_state_tree()
    }

    /// Returns a pointer to the Merkle Accumulator representing the end of the block. Should only
    /// be called when the block has finished execution and `set_output` has been called.
    pub fn clone_transaction_accumulator(&self) -> Rc<Accumulator<TransactionAccumulatorHasher>> {
        self.output
            .as_ref()
            .expect("The block has no output yet.")
            .clone_transaction_accumulator()
    }
}

```

#### BlockProcessor 定义：

  ```rust
enum Mode {
    Normal,
    Syncing,
}

pub(crate) struct BlockProcessor<V> {
    /// Where the processor receives commands.
    command_receiver: mpsc::Receiver<Command>,

    /// The timestamp of the last committed ledger info.
    committed_timestamp_usecs: u64,

    /// The in-memory Sparse Merkle Tree representing last committed state. This tree always has a
    /// single Subtree node (or Empty node) whose hash equals the root hash of the newest Sparse
    /// Merkle Tree in storage.
    committed_state_tree: Rc<SparseMerkleTree>,

    /// The in-memory Merkle Accumulator representing all the committed transactions.
    committed_transaction_accumulator: Rc<Accumulator<TransactionAccumulatorHasher>>,

    /// The main block tree data structure that holds all the uncommitted blocks in memory.
    block_tree: BlockTree<TransactionBlock>,

    /// The blocks that are ready to be sent to storage. After pruning `block_tree` we always put
    /// the blocks here before sending them to storage, so in the case when storage is temporarily
    /// unavailable, we will still prune `block_tree` as normal but blocks will stay here for a bit
    /// longer.
    blocks_to_store: VecDeque<TransactionBlock>,

    /// Client to storage service.
    storage_read_client: Arc<dyn StorageRead>,
    storage_write_client: Arc<dyn StorageWrite>,

    /// The current mode. If we are doing state synchronization, we will refuse to serve normal
    /// execute_block and commit_block requests.
    mode: Mode,

    /// Configuration for the VM. The block processor currently creates a new VM for each block.
    vm_config: VMConfig,

    phantom: PhantomData<V>,
}
  ```

#### BlockProcessor  函数们：

```rust
 /// Keeps processing blocks until the command sender is disconnected.
    pub fn run(&mut self) {
        loop {
            // Fetch and process all commands sent by consensus until there is no more left in the
            // channel.
            while let Ok(cmd) = self.command_receiver.try_recv() {
                self.process_command(cmd);
            }

            // Prune the block tree and check if there are eligible blocks ready to be sent to
            // storage (the blocks that have finished execution and been marked as committed). This
            // will move these blocks from the block tree to `self.blocks_to_store`.
            //
            // Note: If save_blocks_to_storage below fails, these blocks will stay in
            // `self.blocks_to_store`. This is okay because consensus will not retry committing
            // these blocks after it receives the errors. Instead it will try to commit a
            // descendant block later, which will be found in the block tree and cause the entire
            // chain to be saved if storage has recovered. (If consensus retries committing these
            // moved blocks, we won't find these blocks in the block tree because we only look up
            // the blocks in the block tree, so we will return an error.)
            self.blocks_to_store
                .extend(self.block_tree.prune().into_iter());
            if !self.blocks_to_store.is_empty() {
                let time = std::time::Instant::now();
                let mut save_op = || {
                    self.save_blocks_to_storage().map_err(|err| {
                        error!("Failed to save blocks to storage: {}", err);
                        backoff::Error::Transient(err)
                    })
                };
                let mut backoff = Self::storage_retry_backoff();
                match save_op.retry(&mut backoff) {
                    Ok(()) => OP_COUNTERS
                        .observe("blocks_commit_time_us", time.elapsed().as_micros() as f64),
                    Err(_err) => crit!(
                        "Failed to save blocks to storage after trying for {} seconds.",
                        backoff.get_elapsed_time().as_secs(),
                    ),
                }
            }

            // If we do not have anything else to do, check if there is a block pending execution.
            // Continue if this function made progress (executed one block).
            if self.maybe_execute_block() {
                continue;
            }

            // In case the previous attempt to send blocks to storage failed, we want to retry
            // instead of waiting for new command.
            if !self.blocks_to_store.is_empty() {
                continue;
            }

            // We really have nothing to do. Just block the thread until consensus sends us new
            // command.
            match self.command_receiver.recv() {
                Ok(cmd) => self.process_command(cmd),
                Err(mpsc::RecvError) => break,
            }
        }
    }




///上面的函数每次收到发来的command都会调用这个函数
/// Processes a single command from consensus. Note that this only modifies the block tree, the
    /// actual block execution and commit may happen later.
    fn process_command(&mut self, cmd: Command) {
        match cmd {
            Command::ExecuteBlock {
                transactions,
                parent_id,
                id,
                resp_sender,
            } => {
                if let Mode::Syncing = self.mode {
                    Self::send_error_when_syncing(resp_sender, id);
                    return;
                }

                // If the block already exists, we simply store the sender via which the response
                // will be sent when available. Otherwise construct a block and add to the block
                // tree.
                match self.block_tree.get_block_mut(id) {
                    Some(block) => {
                        warn!("Block {:x} already exists.", id);
                        block.queue_execute_block_response_sender(resp_sender);
                    }
                    None => {
                        let block = TransactionBlock::new(transactions, parent_id, id, resp_sender);
                        // If `add_block` errors, we return the error immediately. Otherwise the
                        // response will be returned once the block is executed.
                        if let Err(err) = self.block_tree.add_block(block) {
                            let resp = Err(format_err!("{}", err));
                            let mut block = err.into_block();
                            block.send_execute_block_response(resp);
                        }
                    }
                }
            }
            Command::CommitBlock {
                ledger_info_with_sigs,
                resp_sender,
            } => {
                let id = ledger_info_with_sigs.ledger_info().consensus_block_id();
                if let Mode::Syncing = self.mode {
                    Self::send_error_when_syncing(resp_sender, id);
                    return;
                }

                match self.block_tree.mark_as_committed(id, ledger_info_with_sigs) {
                    Ok(()) => {
                        let block = self
                            .block_tree
                            .get_block_mut(id)
                            .expect("Block must exist if mark_as_committed succeeded.");
                        // We have successfully marked the block as committed, but the real
                        // response will not be sent to consensus until the block is successfully
                        // persisted in storage. So we just save the sender in the block.
                        block.set_commit_response_sender(resp_sender);
                    }
                    Err(err) => resp_sender
                        .send(Err(format_err!("{}", err)))
                        .expect("Failed to send error message."),
                }
            }
            Command::ExecuteChunk {
                txn_list_with_proof,
                ledger_info_with_sigs,
                resp_sender,
            } => {
                let res = self
                    .execute_and_commit_chunk(
                        txn_list_with_proof.clone(),
                        ledger_info_with_sigs.clone(),
                    )
                    .map_err(|e| {
                        security_log(SecurityEvent::InvalidChunkExecutor)
                            .error(&e)
                            .data(txn_list_with_proof)
                            .data(ledger_info_with_sigs)
                            .log();
                        e
                    });
                resp_sender
                    .send(res.map(|_| ExecuteChunkResponse {}))
                    .expect("Failed to send execute chunk response.");
            }
        }
    }





```



#### 针对txns的一系列操作：

````rust
 /// Verifies the transactions based on the provided proofs and ledger info. If the transactions
    /// are valid, executes them and commits immediately if execution results match the proofs.
    fn execute_and_commit_chunk(
        &mut self,
        txn_list_with_proof: TransactionListWithProof,
        ledger_info_with_sigs: LedgerInfoWithSignatures,
    ) -> Result<()> {
        if ledger_info_with_sigs.ledger_info().timestamp_usecs() <= self.committed_timestamp_usecs {
            warn!(
                "Ledger info is too old: local timestamp: {}, timestamp in request: {}.",
                self.committed_timestamp_usecs,
                ledger_info_with_sigs.ledger_info().timestamp_usecs(),
            );
            return Ok(());
        }

        if let Mode::Normal = self.mode {
            self.mode = Mode::Syncing;
            info!("Start syncing...");
        }
        info!(
            "Local version: {}. First transaction version in request: {:?}. \
             Number of transactions in request: {}.",
            self.committed_transaction_accumulator.num_elements() - 1,
            txn_list_with_proof.first_transaction_version,
            txn_list_with_proof.transaction_and_infos.len(),
        );

        let (num_txns_to_skip, first_version) =
            self.verify_chunk(&txn_list_with_proof, &ledger_info_with_sigs)?;
        info!("Skipping the first {} transactions.", num_txns_to_skip);
        let (transactions, infos): (Vec<_>, Vec<_>) = txn_list_with_proof
            .transaction_and_infos
            .into_iter()
            .skip(num_txns_to_skip as usize)
            .unzip();

        // Construct a StateView and pass the transactions to VM.
        let db_root_hash = self.committed_state_tree.root_hash();
        let state_view = VerifiedStateView::new(
            Arc::clone(&self.storage_read_client),
            db_root_hash,
            &self.committed_state_tree,
        );
        let vm_outputs = {
            let time = std::time::Instant::now();
            let out = V::execute_block(transactions.clone(), &self.vm_config, &state_view);
            OP_COUNTERS.observe(
                "vm_execute_chunk_time_us",
                time.elapsed().as_micros() as f64,
            );
            out
        };

        // Since other validators have committed these transactions, their status should all be
        // TransactionStatus::Keep.
        for output in &vm_outputs {
            if let TransactionStatus::Discard(_) = output.status() {
                bail!("Syncing transactions that should be discarded.");
            }
        }

        let (account_to_btree, account_to_proof) = state_view.into();
        let output = Self::process_vm_outputs(
            account_to_btree,
            account_to_proof,
            &transactions,
            vm_outputs,
            Rc::clone(&self.committed_state_tree),
            Rc::clone(&self.committed_transaction_accumulator),
        )?;

        // Since we have verified the proofs, we just need to verify that each TransactionInfo
        // object matches what we have computed locally.
        let mut txns_to_commit = vec![];
        for ((txn, txn_data), (i, txn_info)) in itertools::zip_eq(
            itertools::zip_eq(transactions, output.transaction_data()),
            infos.into_iter().enumerate(),
        ) {
            ensure!(
                txn_info.state_root_hash() == txn_data.state_root_hash(),
                "State root hashes do not match for {}-th transaction in chunk.",
                i,
            );
            ensure!(
                txn_info.event_root_hash() == txn_data.event_root_hash(),
                "Event root hashes do not match for {}-th transaction in chunk.",
                i,
            );
            ensure!(
                txn_info.gas_used() == txn_data.gas_used(),
                "Gas used do not match for {}-th transaction in chunk.",
                i,
            );
            txns_to_commit.push(TransactionToCommit::new(
                txn,
                txn_data.account_blobs().clone(),
                txn_data.events().to_vec(),
                txn_data.gas_used(),
            ));
        }

        // If this is the last chunk corresponding to this ledger info, send the ledger info to
        // storage.
        let ledger_info_to_commit = if self.committed_transaction_accumulator.num_elements()
            + txns_to_commit.len() as u64
            == ledger_info_with_sigs.ledger_info().version() + 1
        {
            // We have constructed the transaction accumulator root and checked that it matches the
            // given ledger info in the verification process above, so this check can possibly fail
            // only when input transaction list is empty.
            ensure!(
                ledger_info_with_sigs
                    .ledger_info()
                    .transaction_accumulator_hash()
                    == output.clone_transaction_accumulator().root_hash(),
                "Root hash in ledger info does not match local computation."
            );
            Some(ledger_info_with_sigs)
        } else {
            None
        };
        self.storage_write_client.save_transactions(
            txns_to_commit,
            first_version,
            ledger_info_to_commit.clone(),
        )?;

        self.committed_state_tree = output.clone_state_tree();
        self.committed_transaction_accumulator = output.clone_transaction_accumulator();
        if let Some(ledger_info_with_sigs) = ledger_info_to_commit {
            self.committed_timestamp_usecs = ledger_info_with_sigs.ledger_info().timestamp_usecs();
            self.block_tree
                .reset(ledger_info_with_sigs.ledger_info().consensus_block_id());
            self.mode = Mode::Normal;
            info!(
                "Synced to version {}.",
                ledger_info_with_sigs.ledger_info().version()
            );
        }

        Ok(())
    }



/// Verifies the proofs using provided ledger info. Also verifies that the version of the first
    /// transaction matches the lastest committed transaction. If the first few transaction happens
    /// to be older, returns how many need to be skipped and the first version to be committed.
//这个函数进行了一系列检查，用来verify这一部分txns
    fn verify_chunk(
        &self,
        txn_list_with_proof: &TransactionListWithProof,
        ledger_info_with_sigs: &LedgerInfoWithSignatures,
    ) -> Result<(u64, Version)> {
        txn_list_with_proof.verify(
            ledger_info_with_sigs.ledger_info(),
            txn_list_with_proof.first_transaction_version,
        )?;

        let num_committed_txns = self.committed_transaction_accumulator.num_elements();
        if txn_list_with_proof.transaction_and_infos.is_empty() {
            return Ok((0, num_committed_txns /* first_version */));
        }

        let first_txn_version = txn_list_with_proof
            .first_transaction_version
            .expect("first_transaction_version should exist.");

        ensure!(
            first_txn_version <= num_committed_txns,
            "Transaction list too new. Expected version: {}. First transaction version: {}.",
            num_committed_txns,
            first_txn_version
        );
        Ok((num_committed_txns - first_txn_version, num_committed_txns))
    }

````

#### block to storage：

```rust
 /// Saves eligible blocks to persistent storage. If the blocks are successfully persisted, they
    /// will be removed from `self.blocks_to_store` and the in-memory Sparse Merkle Trees in these
    /// blocks will be pruned. Otherwise nothing happens.
    ///
    /// If we have multiple blocks and not all of them have signatures, we may send them to storage
    /// in a few batches. For example, if we have
    /// ```text
    /// A <- B <- C <- D <- E
    /// ```
    /// and only `C` and `E` have signatures, we will send `A`, `B` and `C` in the first batch,
    /// then `D` and `E` later in the another batch.
    fn save_blocks_to_storage(&mut self) -> Result<()> {
        // The blocks we send to storage in this batch. In the above example, this means block A, B
        // and C.
        let mut block_batch = vec![];
        for block in &mut self.blocks_to_store {
            let should_stop = block.ledger_info_with_sigs().is_some();
            block_batch.push(block);
            if should_stop {
                break;
            }
        }
        assert!(!block_batch.is_empty());

        // All transactions that need to go to storage. In the above example, this means all the
        // transactions in A, B and C whose status == TransactionStatus::Keep.
        let mut txns_to_commit = vec![];
        let mut num_accounts_created = 0;
        for block in &block_batch {
            for (txn, txn_data) in itertools::zip_eq(
                block.transactions(),
                block
                    .output()
                    .as_ref()
                    .expect("All blocks in self.blocks_to_store should have finished execution.")
                    .transaction_data(),
            ) {
                if let TransactionStatus::Keep(_) = txn_data.status() {
                    txns_to_commit.push(TransactionToCommit::new(
                        txn.clone(),
                        txn_data.account_blobs().clone(),
                        txn_data.events().to_vec(),
                        txn_data.gas_used(),
                    ));
                    num_accounts_created += txn_data.num_account_created();
                }
            }
        }

        let last_block = block_batch
            .last_mut()
            .expect("There must be at least one block with signatures.");

        // Check that the version in ledger info (computed by consensus) matches the version
        // computed by us. TODO: we should also verify signatures and check that timestamp is
        // strictly increasing.
        let ledger_info_with_sigs = last_block
            .ledger_info_with_sigs()
            .as_ref()
            .expect("This block must have signatures.");
        let version = ledger_info_with_sigs.ledger_info().version();
        let num_txns_in_accumulator = last_block.clone_transaction_accumulator().num_elements();
        assert_eq!(
            version + 1,
            num_txns_in_accumulator,
            "Number of transactions in ledger info ({}) does not match number of transactions \
             in accumulator ({}).",
            version + 1,
            num_txns_in_accumulator,
        );

        let num_txns_to_commit = txns_to_commit.len() as u64;
        {
            let time = std::time::Instant::now();
            self.storage_write_client.save_transactions(
                txns_to_commit,
                version + 1 - num_txns_to_commit, /* first_version */
                Some(ledger_info_with_sigs.clone()),
            )?;
            OP_COUNTERS.observe(
                "storage_save_transactions_time_us",
                time.elapsed().as_micros() as f64,
            );
        }
        // Only bump the counter when the commit succeeds.
        OP_COUNTERS.inc_by("num_accounts", num_accounts_created);

        // Now that the blocks are persisted successfully, we can reply to consensus and update
        // in-memory state.
        self.committed_timestamp_usecs = ledger_info_with_sigs.ledger_info().timestamp_usecs();
        self.committed_state_tree = last_block.clone_state_tree();
        self.committed_transaction_accumulator = last_block.clone_transaction_accumulator();
        last_block.send_commit_block_response(Ok(CommitBlockResponse::Succeeded));

        let num_saved = block_batch.len();
        for _i in 0..num_saved {
            let block = self
                .blocks_to_store
                .pop_front()
                .expect("self.blocks_to_store must have more blocks.");
            let block_data = block
                .output()
                .as_ref()
                .expect("All blocks in self.blocks_to_store should have output.");
            for txn_data in block_data.transaction_data() {
                txn_data.prune_state_tree();
            }
        }

        Ok(())
    }

```

block & execution:

```rust
/// Checks if there is a block in the tree ready for execution, if so run it by calling the VM.
    /// Returns `true` if a block was successfully executed, `false` if there was no block to
    /// execute.
    fn maybe_execute_block(&mut self) -> bool {
        let id = match self.block_tree.get_block_to_execute() {
            Some(block_id) => block_id,
            None => return false,
        };

        {
            let time = std::time::Instant::now();
            self.execute_block(id);
            OP_COUNTERS.observe("block_execute_time_us", time.elapsed().as_micros() as f64);
        }

        true
    }

    fn execute_block(&mut self, id: HashValue) {
        let (previous_state_tree, previous_transaction_accumulator) =
            self.get_trees_from_parent(id);

        let block_to_execute = self
            .block_tree
            .get_block_mut(id)
            .expect("Block to execute should exist.");

        // Construct a StateView and pass the transactions to VM.
        let db_root_hash = self.committed_state_tree.root_hash();
        let state_view = VerifiedStateView::new(
            Arc::clone(&self.storage_read_client),
            db_root_hash,
            &previous_state_tree,
        );
        let vm_outputs = {
            let time = std::time::Instant::now();
            let out = V::execute_block(
                block_to_execute.transactions().to_vec(),
                &self.vm_config,
                &state_view,
            );
            OP_COUNTERS.observe(
                "vm_execute_block_time_us",
                time.elapsed().as_micros() as f64,
            );
            out
        };

        let status: Vec<_> = vm_outputs
            .iter()
            .map(TransactionOutput::status)
            .cloned()
            .collect();
        if !status.is_empty() {
            debug!("Execution status: {:?}", status);
        }

        let (account_to_btree, account_to_proof) = state_view.into();
        match Self::process_vm_outputs(
            account_to_btree,
            account_to_proof,
            block_to_execute.transactions(),
            vm_outputs,
            previous_state_tree,
            previous_transaction_accumulator,
        ) {
            Ok(output) => {
                let accumulator = output.clone_transaction_accumulator();
                let root_hash = accumulator.root_hash();
                let version = accumulator.num_elements() - 1;
                block_to_execute.set_output(output);

                // Now that we have the root hash and execution status we can send the response to
                // consensus.
                // TODO: The VM will support a special transaction to set the validators for the
                // next epoch that is part of a block execution.
                let execute_block_response =
                    ExecuteBlockResponse::new(root_hash, status, version, None);
                block_to_execute.set_execute_block_response(execute_block_response);
            }
            Err(err) => {
                block_to_execute.send_execute_block_response(Err(format_err!(
                    "Failed to execute block: {}",
                    err
                )));
                // If we failed to execute this block, remove the block and its descendants from
                // the block tree.
                self.block_tree.remove_subtree(id);
            }
        }
    }

```

#### block & process the result:

```rust
/// Post-processing of what the VM outputs. Returns the entire block's output.
    fn process_vm_outputs(
        mut account_to_btree: HashMap<AccountAddress, BTreeMap<Vec<u8>, Vec<u8>>>,
        account_to_proof: HashMap<HashValue, SparseMerkleProof>,
        transactions: &[SignedTransaction],
        vm_outputs: Vec<TransactionOutput>,
        previous_state_tree: Rc<SparseMerkleTree>,
        previous_transaction_accumulator: Rc<Accumulator<TransactionAccumulatorHasher>>,
    ) -> Result<ProcessedVMOutput> {
        // The data of each individual transaction. For convenience purpose, even for the
        // transactions that will be discarded, we will compute its in-memory Sparse Merkle Tree
        // (it will be identical to the previous one).
        let mut txn_data = vec![];
        let mut current_state_tree = previous_state_tree;
        // The hash of each individual TransactionInfo object. This will not include the
        // transactions that will be discarded, since they do not go into the transaction
        // accumulator.
        let mut txn_info_hashes = vec![];

        let proof_reader = ProofReader::new(account_to_proof);
        for (vm_output, signed_txn) in
            itertools::zip_eq(vm_outputs.into_iter(), transactions.iter())
        {
            let (blobs, state_tree, num_accounts_created) = Self::process_write_set(
                signed_txn,
                &mut account_to_btree,
                &proof_reader,
                vm_output.write_set().clone(),
                &current_state_tree,
            )?;

            let event_tree = Accumulator::<EventAccumulatorHasher>::default()
                .append(vm_output.events().iter().map(CryptoHash::hash).collect());

            match vm_output.status() {
                TransactionStatus::Keep(_) => {
                    ensure!(
                        !vm_output.write_set().is_empty(),
                        "Transaction with empty write set should be discarded.",
                    );
                    // Compute hash for the TransactionInfo object. We need the hash of the
                    // transaction itself, the state root hash as well as the event root hash.
                    let txn_info = TransactionInfo::new(
                        signed_txn.hash(),
                        state_tree.root_hash(),
                        event_tree.root_hash(),
                        vm_output.gas_used(),
                    );
                    txn_info_hashes.push(txn_info.hash());
                }
                TransactionStatus::Discard(_) => {
                    ensure!(
                        vm_output.write_set().is_empty(),
                        "Discarded transaction has non-empty write set.",
                    );
                    ensure!(
                        vm_output.events().is_empty(),
                        "Discarded transaction has non-empty events.",
                    );
                }
            }

            txn_data.push(TransactionData::new(
                blobs,
                vm_output.events().to_vec(),
                vm_output.status().clone(),
                Rc::clone(&state_tree),
                Rc::new(event_tree),
                vm_output.gas_used(),
                num_accounts_created,
            ));
            current_state_tree = state_tree;
        }

        let current_transaction_accumulator =
            previous_transaction_accumulator.append(txn_info_hashes);
        Ok(ProcessedVMOutput::new(
            txn_data,
            Rc::new(current_transaction_accumulator),
            current_state_tree,
        ))
    }

    /// For all accounts modified by this transaction, find the previous blob and update it based
    /// on the write set. Returns the blob value of all these accounts as well as the newly
    /// constructed state tree.
    fn process_write_set(
        transaction: &SignedTransaction,
        account_to_btree: &mut HashMap<AccountAddress, BTreeMap<Vec<u8>, Vec<u8>>>,
        proof_reader: &ProofReader,
        write_set: WriteSet,
        previous_state_tree: &SparseMerkleTree,
    ) -> Result<(
        HashMap<AccountAddress, AccountStateBlob>,
        Rc<SparseMerkleTree>,
        usize, /* num_account_created */
    )> {
        let mut updated_blobs = HashMap::new();
        let mut num_accounts_created = 0;

        // Find all addresses this transaction touches while processing each write op.
        let mut addrs = HashSet::new();
        for (access_path, write_op) in write_set.into_iter() {
            let address = access_path.address;
            let path = access_path.path;
            match account_to_btree.entry(address) {
                hash_map::Entry::Occupied(mut entry) => {
                    let account_btree = entry.get_mut();
                    // TODO(gzh): we check account creation here for now. Will remove it once we
                    // have a better way.
                    if account_btree.is_empty() {
                        num_accounts_created += 1;
                    }
                    Self::update_account_btree(account_btree, path, write_op);
                }
                hash_map::Entry::Vacant(entry) => {
                    // Before writing to an account, VM should always read that account. So we
                    // should not reach this code path. The exception is genesis transaction (and
                    // maybe other FTVM transactions).
                    match transaction.payload() {
                        TransactionPayload::Program(_) => {
                            bail!("Write set should be a subset of read set.")
                        }
                        TransactionPayload::WriteSet(_) => (),
                    }

                    let mut account_btree = BTreeMap::new();
                    Self::update_account_btree(&mut account_btree, path, write_op);
                    entry.insert(account_btree);
                }
            }
            addrs.insert(address);
        }

        for addr in addrs {
            let account_btree = account_to_btree.get(&addr).expect("Address should exist.");
            let account_blob = AccountStateBlob::try_from(account_btree)?;
            updated_blobs.insert(addr, account_blob);
        }
        let state_tree = Rc::new(
            previous_state_tree
                .update(
                    updated_blobs
                        .iter()
                        .map(|(addr, value)| (addr.hash(), value.clone()))
                        .collect(),
                    proof_reader,
                )
                .expect("Failed to update state tree."),
        );

        Ok((updated_blobs, state_tree, num_accounts_created))
    }

    fn update_account_btree(
        account_btree: &mut BTreeMap<Vec<u8>, Vec<u8>>,
        path: Vec<u8>,
        write_op: WriteOp,
    ) {
        match write_op {
            WriteOp::Value(new_value) => account_btree.insert(path, new_value),
            WriteOp::Deletion => account_btree.remove(&path),
        };
    }
}

```

