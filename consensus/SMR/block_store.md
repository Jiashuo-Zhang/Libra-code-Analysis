# block_store

## 代码：

####build_block_tree:

```rust

/// Responsible for maintaining all the blocks of payload and the dependencies of those blocks
/// (parent and previous QC links).  It is expected to be accessed concurrently by multiple threads
/// and is thread-safe.
///
/// Example tree block structure based on parent links.
///                         | -> A3
/// Genesis -> B0 -> B1 -> B2 -> B3
///             | -> C1 -> C2
///                         | -> D3
///
/// Example corresponding tree block structure for the QC links (must follow QC constraints).
///                         | -> A3
/// Genesis -> B0 -> B1 -> B2 -> B3
///             | -> C1
///             | -------> C2
///             | -------------> D3
pub struct BlockStore<T> {
    inner: Arc<RwLock<BlockTree<T>>>,
    validator_signer: ValidatorSigner,
    state_computer: Arc<dyn StateComputer<Payload = T>>,
    enforce_increasing_timestamps: bool,
    /// The persistent storage backing up the in-memory data structure, every write should go
    /// through this before in-memory tree.
    storage: Arc<dyn PersistentStorage<T>>,
}

 async fn build_block_tree(
        root: (Block<T>, QuorumCert, QuorumCert),//选择根
        blocks: Vec<Block<T>>,///block们
        quorum_certs: Vec<QuorumCert>,//QCs
        state_computer: Arc<dyn StateComputer<Payload = T>>,
        max_pruned_blocks_in_mem: usize,
    ) -> BlockTree<T> {
        let mut tree = BlockTree::new(root.0, root.1, root.2, max_pruned_blocks_in_mem);
        let quorum_certs = quorum_certs
            .into_iter()
            .map(|qc| (qc.certified_block_id(), qc))
            .collect::<HashMap<_, _>>();//吧QCs转化成一个hashmap便于查找
        for block in blocks {
            let compute_res = state_computer
                .compute(block.parent_id(), block.id(), block.get_payload())
                .await
                .expect("fail to rebuild scratchpad");
            let version = tree
                .get_state_for_block(block.parent_id())
                .expect("parent state does not exist")
                .version
                + compute_res.num_successful_txns;
            let executed_state = ExecutedState {//根据state_computer的计算结果生成excuted_state
                state_id: compute_res.new_state_id,
                version,
            };
            // if this block is certified, ensure we agree with the certified state.
            if let Some(qc) = quorum_certs.get(&block.id()) {
                assert_eq!(
                    qc.certified_state(),
                    executed_state,
                    "We have inconsistent executed state with Quorum Cert for block {}",
                    block.id()
                );
            }
            tree.insert_block(block, executed_state, compute_res)
                .expect("Block insertion failed while build the tree");
        }
        quorum_certs.into_iter().for_each(|(_, qc)| {
            tree.insert_quorum_cert(qc)
                .expect("QuorumCert insertion failed while build the tree")
        });
        tree
    }
```

####execute_and_insert_block:

```rust
/// Execute and insert a block if it passes all validation tests.
    /// Returns the Arc to the block kept in the block store after persisting it to storage
    ///
    /// This function assumes that the ancestors are present (returns MissingParent otherwise).
    ///
    /// Duplicate inserts will return the previously inserted block (
    /// note that it is considered a valid non-error case, for example, it can happen if a validator
    /// receives a certificate for a block that is currently being added).
    pub async fn execute_and_insert_block(
        &self,
        block: Block<T>,
    ) -> Result<Arc<Block<T>>, InsertError> {
        if let Some(existing_block) = self.inner.read().unwrap().get_block(block.id()) {
            return Ok(existing_block);
        }
        let (parent_id, parent_exec_version) = match self.verify_and_get_parent_info(&block) {
            Ok(t) => t,
            Err(e) => {
                security_log(SecurityEvent::InvalidBlock)
                    .error(&e)
                    .data(&block)
                    .log();
                return Err(e);
            }
        };
        let compute_res = self
            .state_computer
            .compute(parent_id, block.id(), block.get_payload())
            .await
            .map_err(|e| {
                error!("Execution failure for block {}: {:?}", block, e);
                InsertError::StateComputerError
            })?;

        let version = parent_exec_version + compute_res.num_successful_txns;

        let state = ExecutedState {
            state_id: compute_res.new_state_id,
            version,
        };
        self.storage
            .save_tree(vec![block.clone()], vec![])
            .map_err(|_| InsertError::StorageFailure)?;// 加入storage
        self.inner
            .write()
            .unwrap()
            .insert_block(block, state, compute_res)//insert
            .map_err(|e| e.into())
    }
```

#### 关于QC的一些判断：

```rust
/// Check if we're far away from this ledger info and need to sync.
    /// Returns false if we have this block in the tree or the root's round is higher than the
    /// block.
    pub fn need_sync_for_quorum_cert(
        &self,
        committed_block_id: HashValue,
        qc: &QuorumCert,
    ) -> bool {
        // LedgerInfo doesn't carry the information about the round of the committed block. However,
        // the 3-chain safety rules specify that the round of the committed block must be
        // certified_block_round() - 2. In case root().round() is greater than that the committed
        // block carried by LI is older than my current commit.
        !(self.block_exists(committed_block_id)
            || self.root().round() + 2 >= qc.certified_block_round())//后一个是说被committed 的block在root之前，前一个是说这个QC已经在树中了，也就是之前已经committed过了
    }

    /// Checks if quorum certificate can be inserted in block store without RPC
    /// Returns the enum to indicate the detailed status.
    pub fn need_fetch_for_quorum_cert(&self, qc: &QuorumCert) -> NeedFetchResult {
        if qc.certified_block_round() < self.root().round() {
            return NeedFetchResult::QCRoundBeforeRoot;
        }
        if self
            .get_quorum_cert_for_block(qc.certified_block_id())
            .is_some()
        {
            return NeedFetchResult::QCAlreadyExist;
        }
        if self.block_exists(qc.certified_block_id()) {
            return NeedFetchResult::QCBlockExist;
        }
        NeedFetchResult::NeedFetch
    }

```

#### 插入QC和插入vote：

```rust
 /// Validates quorum certificates and inserts it into block tree assuming dependencies exist.
    pub async fn insert_single_quorum_cert(&self, qc: QuorumCert) -> Result<(), InsertError> {
        // Ensure executed state is consistent with Quorum Cert, otherwise persist the quorum's
        // state and hopefully we restart and agree with it.
        let executed_state = self
            .get_state_for_block(qc.certified_block_id())
            .ok_or_else(|| InsertError::MissingParentBlock(qc.certified_block_id()))?;
        assert_eq!(
            executed_state,
            qc.certified_state(),
            "We have inconsistent executed state with the executed state from the quorum \
             certificate for block {}, will kill this validator and rely on state synchronization \
             to try to achieve consistent state with the quorum certificate.",
            qc.certified_block_id(),
        );
        self.storage
            .save_tree(vec![], vec![qc.clone()])
            .map_err(|_| InsertError::StorageFailure)?;
        self.inner
            .write()
            .unwrap()
            .insert_quorum_cert(qc)
            .map_err(|e| e.into())
    }

    /// Adds a vote for the block.
    /// The returned value either contains the vote result (with new / old QC etc.) or a
    /// verification error.
    /// A block store does not verify that the block, which is voted for, is present locally.
    /// It returns QC, if it is formed, but does not insert it into block store, because it might
    /// not have required dependencies yet
    /// Different execution ids are treated as different blocks (e.g., if some proposal is
    /// executed in a non-deterministic fashion due to a bug, then the votes for execution result
    /// A and the votes for execution result B are aggregated separately).
    pub async fn insert_vote(
        &self,
        vote_msg: VoteMsg,
        min_votes_for_qc: usize,
    ) -> VoteReceptionResult {
        self.inner
            .write()
            .unwrap()
            .insert_vote(&vote_msg, min_votes_for_qc)
    }
```

#### prune the tree:修建tree，给树换根

```rust
/// Prune the tree up to next_root_id (keep next_root_id's block).  Any branches not part of
    /// the next_root_id's tree should be removed as well.
    ///
    /// For example, root = B_0
    /// B_0 -> B_1 -> B_2
    ///         |  -> B_3 -> B4
    ///
    /// prune_tree(B_3) should be left with
    /// B_3 -> B_4, root = B_3
    ///
    /// Returns the block ids of the blocks removed.
    pub async fn prune_tree(&self, next_root_id: HashValue) -> VecDeque<HashValue> {
        let id_to_remove = self
            .inner
            .read()
            .unwrap()
            .find_blocks_to_prune(next_root_id);
        if let Err(e) = self
            .storage
            .prune_tree(id_to_remove.clone().into_iter().collect())
        {
            // it's fine to fail here, as long as the commit succeeds, the next restart will clean
            // up dangling blocks, and we need to prune the tree to keep the root consistent with
            // executor.
            error!("fail to delete block: {:?}", e);
        }
        self.inner
            .write()
            .unwrap()
            .process_pruned_blocks(next_root_id, id_to_remove.clone());
        id_to_remove
    }
```

#### verify_and_get_parent_info，ledger_info_placeholder等如名称所示。返回block parent的信息以及自己的信息

