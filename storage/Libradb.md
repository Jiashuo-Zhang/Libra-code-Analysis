# Libradb

###功能：libradb是比较核心的部分，他调度了ledger_store,transaction_store,state_store,event_store几个模块，对于这几个模块我们暂且不去深究里面的具体实现了（只分析了event_store），只关注他们对外提供的API们是如何工作的。

## 代码：

#### 定义：

```rust
/// This holds a handle to the underlying DB responsible for physical storage and provides APIs for
/// access to the core Libra data structures.
pub struct LibraDB {
    db: Arc<DB>,
    ledger_store: LedgerStore,
    transaction_store: TransactionStore,
    state_store: StateStore,
    event_store: EventStore,
}

impl LibraDB {
    /// This creates an empty LibraDB instance on disk or opens one if it already exists.
    pub fn new<P: AsRef<Path> + Clone>(db_root_path: P) -> Self {
        let cf_opts_map: ColumnFamilyOptionsMap = [
            (
                /* LedgerInfo CF = */ DEFAULT_CF_NAME,
                ColumnFamilyOptions::default(),
            ),
            (ACCOUNT_STATE_CF_NAME, ColumnFamilyOptions::default()),
            (EVENT_ACCUMULATOR_CF_NAME, ColumnFamilyOptions::default()),
            (EVENT_BY_ACCESS_PATH_CF_NAME, ColumnFamilyOptions::default()),
            (EVENT_CF_NAME, ColumnFamilyOptions::default()),
            (RETIRED_STATE_RECORD_CF_NAME, ColumnFamilyOptions::default()),
            (SIGNED_TRANSACTION_CF_NAME, ColumnFamilyOptions::default()),
            (STATE_MERKLE_NODE_CF_NAME, ColumnFamilyOptions::default()),
            (
                TRANSACTION_ACCUMULATOR_CF_NAME,
                ColumnFamilyOptions::default(),
            ),
            (TRANSACTION_INFO_CF_NAME, ColumnFamilyOptions::default()),
            (VALIDATOR_CF_NAME, ColumnFamilyOptions::default()),
        ]
        .iter()
        .cloned()
        .collect();

        let path = db_root_path.as_ref().join("libradb");
        let instant = Instant::now();
        let db = Arc::new(
            DB::open(path.clone(), cf_opts_map)
                .unwrap_or_else(|e| panic!("LibraDB open failed: {:?}", e)),
        );

        info!(
            "Opened LibraDB at {:?} in {} ms",
            path,
            instant.elapsed().as_millis()
        );

        LibraDB {
            db: Arc::clone(&db),
            event_store: EventStore::new(Arc::clone(&db)),
            ledger_store: LedgerStore::new(Arc::clone(&db)),
            state_store: StateStore::new(Arc::clone(&db)),
            transaction_store: TransactionStore::new(Arc::clone(&db)),
        }
    }

    // 
```

####get_account_state_with_proof:

```rust
 // ================================== Public API ==================================
    /// Returns the account state corresponding to the given version and account address with proof
    /// based on `ledger_version`
// 
    fn get_account_state_with_proof(
        &self,
        address: AccountAddress,
        version: Version,
        ledger_version: Version,
    ) -> Result<AccountStateWithProof> {
        ensure!(
            version <= ledger_version,
            "The queried version {} should be equal to or older than ledger version {}.",
            version,
            ledger_version
        );
        let latest_version = self.get_latest_version()?;
        ensure!(
            ledger_version <= latest_version,
            "The ledger version {} is greater than the latest version currently in ledger: {}",
            ledger_version,
            latest_version
        );

        let (txn_info, txn_info_accumulator_proof) = self
            .ledger_store
            .get_transaction_info_with_proof(version, ledger_version)?;
        let (account_state_blob, sparse_merkle_proof) = self
            .state_store
            .get_account_state_with_proof_by_state_root(address, txn_info.state_root_hash())?;
        Ok(AccountStateWithProof::new(
            version,
            account_state_blob,
            AccountStateProof::new(txn_info_accumulator_proof, txn_info, sparse_merkle_proof),
        ))
        ///值得注意的是这里面的proof有3个组成部分，分别是这个version的txn和txns_proof
        //,以及state树的proof
    }
```

####get_events_by_event_access_path:

```rust
 /// Returns events specified by `access_path` with sequence number in range designated by
    /// `start_seq_num`, `ascending` and `limit`. If ascending is true this query will return up to
    /// `limit` events that were emitted after `start_event_seq_num`. Otherwise it will return up to
    /// `limit` events in the reverse order. Both cases are inclusive.
    fn get_events_by_event_access_path(
        &self,
        access_path: &AccessPath,
        start_seq_num: u64,
        ascending: bool,
        limit: u64,
        ledger_version: Version,
    ) -> Result<(Vec<EventWithProof>, Option<AccountStateWithProof>)> {
        error_if_too_many_requested(limit, MAX_LIMIT)?;

        let get_latest = !ascending && start_seq_num == u64::max_value();
        let cursor = if get_latest {
            // Caller wants the latest, figure out the latest seq_num.
            // In the case of no events on that path, use 0 and expect empty result below.
            self.event_store
                .get_latest_sequence_number(ledger_version, access_path)?
                .unwrap_or(0)
        } else {
            start_seq_num
        };

        // Convert requested range and order to a range in ascending order.
        let (first_seq, real_limit) = get_first_seq_num_and_limit(ascending, cursor, limit)?;

        // Query the index.
        let mut event_keys = self.event_store.lookup_events_by_access_path(
            access_path,
            first_seq,
            real_limit,
            ledger_version,
        )?;

        // When descending, it's possible that user is asking for something beyond the latest
        // sequence number, in which case we will consider it a bad request and return an empty
        // list.
        // For example, if the latest sequence number is 100, and the caller is asking for 110 to
        // 90, we will get 90 to 100 from the index lookup above. Seeing that the last item
        // is 100 instead of 110 tells us 110 is out of bound.
        if !ascending {
            if let Some((seq_num, _, _)) = event_keys.last() {
                if *seq_num < cursor {
                    event_keys = Vec::new();//如果超过了范围就最直接返回空
                }
            }
        }

        let mut events_with_proof = event_keys//获取proof
            .into_iter()
            .map(|(seq, ver, idx)| {
                let (event, event_proof) = self
                    .event_store
                    .get_event_with_proof_by_version_and_index(ver, idx)?;
                ensure!(
                    seq == event.sequence_number(),
                    "Index broken, expected seq:{}, actual:{}",
                    seq,
                    event.sequence_number()
                );
                let (txn_info, txn_info_proof) = self
                    .ledger_store
                    .get_transaction_info_with_proof(ver, ledger_version)?;
                let proof = EventProof::new(txn_info_proof, txn_info, event_proof);
                Ok(EventWithProof::new(ver, idx, event, proof))
            })
            .collect::<Result<Vec<_>>>()?;
        if !ascending {
            events_with_proof.reverse();
        }

        // There are two cases where we need to return proof_of_latest_event to let the caller know
        // the latest sequence number:
        //   1. The user asks for the latest event by using u64::max() as the cursor, apparently
        // he doesn't know the latest sequence number.
        //   2. We are going to return less than `real_limit` items. (Two cases can lead to that:
        // a. the cursor is beyond the latest sequence number; b. in ascending order we don't have
        // enough items to return because the latest sequence number is hit). In this case we
        // need to return the proof to convince the caller we didn't hide any item from him. Note
        // that we use `real_limit` instead of `limit` here because it takes into account the case
        // of hitting 0 in descending order, which is valid and doesn't require the proof.
        ////如果我们确实没有足够的event返回回去的话，我们就会生成一个latest_event的prrof，来证明确实是我们没有足够的event，而不是想要隐瞒什么
        let proof_of_latest_event = if get_latest || events_with_proof.len() < real_limit as usize {
            Some(self.get_account_state_with_proof(
                access_path.address,
                ledger_version,
                ledger_version,
            )?)
        } else {
            None
        };

        Ok((events_with_proof, proof_of_latest_event))
    }

```

####get_txn_by_account_and_seq  &  get_latest_version:

```rust
 /// Returns a signed transaction that is the `seq_num`-th one associated with the given account.
    /// If the signed transaction with given `seq_num` doesn't exist, returns `None`.
    // TODO(gzh): Use binary search for now. We may create seq_num index in the future.
//将来要做的事为txn们建立一个索引
    fn get_txn_by_account_and_seq(
        &self,
        address: AccountAddress,
        seq_num: u64,
        ledger_version: Version,
        fetch_events: bool,
    ) -> Result<Option<SignedTransactionWithProof>> {
        // If txn with seq_num n is at some version, the corresponding account state at the
        // same version will be the first account state that has seq_num n + 1.
        //二分查找，定位是在哪一个version里面
        let seq_num = seq_num + 1;
        let (mut start_version, mut end_version) = (0, ledger_version);
        while start_version < end_version {
            let mid_version = start_version + (end_version - start_version) / 2;
            let account_seq_num = self.get_account_seq_num_by_version(address, mid_version)?;
            if account_seq_num >= seq_num {
                end_version = mid_version;
            } else {
                start_version = mid_version + 1;
            }
        }
        assert_eq!(start_version, end_version);
        

        let seq_num_found = self.get_account_seq_num_by_version(address, start_version)?;
        if seq_num_found < seq_num {
            return Ok(None);
        } else if seq_num_found > seq_num {
            // log error
            bail!("internal error: seq_num is not continuous.")
        }
        // start_version cannot be 0 (genesis version).
        assert_eq!(
            self.get_account_seq_num_by_version(address, start_version - 1)?,
            seq_num_found - 1
        );
        self.get_transaction_with_proof(start_version, ledger_version, fetch_events)
            .map(Some)
    }

    /// Gets the latest version number available in the ledger.
    fn get_latest_version(&self) -> Result<Version> {
        Ok(self
            .ledger_store
            .get_latest_ledger_info()?
            .ledger_info()
            .version())
    }
```

#### save类操作：

 ```rust
/// Persist transactions. Called by the executor module when either syncing nodes or committing
    /// blocks during normal operation.
    ///
    /// When `ledger_info_with_sigs` is provided, verify that the transaction accumulator root hash
    /// it carries is generated after the `txns_to_commit` are applied.
    pub fn save_transactions(
        &self,
        txns_to_commit: &[TransactionToCommit],
        first_version: Version,
        ledger_info_with_sigs: &Option<LedgerInfoWithSignatures>,
    ) -> Result<()> {
        let num_txns = txns_to_commit.len() as u64;
        // ledger_info_with_sigs could be None if we are doing state synchronization. In this case
        // txns_to_commit should not be empty. Otherwise it is okay to commit empty blocks.
        ensure!(
            ledger_info_with_sigs.is_some() || num_txns > 0,
            "txns_to_commit is empty while ledger_info_with_sigs is None.",
        );

        let cur_state_root_hash = if first_version == 0 {
            *SPARSE_MERKLE_PLACEHOLDER_HASH
        } else {
            self.ledger_store
                .get_transaction_info(first_version - 1)?
                .state_root_hash()
        };

        let last_version = first_version + num_txns - 1;
        if let Some(x) = ledger_info_with_sigs {
            let claimed_last_version = x.ledger_info().version();
            ensure!(
                claimed_last_version == last_version,
                "Transaction batch not applicable: first_version {}, num_txns {}, last_version {}",
                first_version,
                num_txns,
                claimed_last_version,
            );
        }
/////前面是大致先检查了一下，txn数目够不够，
        // Gather db mutations to `batch`.
        let mut batch = SchemaBatch::new();

        
        //然后调用save_transactions_impl函数，来本地运行这些交易并获得本地运行的结果
        let new_root_hash = self.save_transactions_impl(
            txns_to_commit,
            first_version,
            cur_state_root_hash,
            &mut batch,
        )?;

        // If expected ledger info is provided, verify result root hash and save the ledger info.
        if let Some(x) = ledger_info_with_sigs {
            let expected_root_hash = x.ledger_info().transaction_accumulator_hash();
            ensure!(
                new_root_hash == expected_root_hash,
                "Root hash calculated doesn't match expected. {:?} vs {:?}",////本地的结果和发过来的结果必须一样
                new_root_hash,
                expected_root_hash,
            );

            self.ledger_store.put_ledger_info(x, &mut batch)?;
        }

        // Persist.
        self.commit(batch)?;
        // Only increment counter if commit(batch) succeeds.
        OP_COUNTER.inc_by("committed_txns", txns_to_commit.len());
        OP_COUNTER.set("latest_transaction_version", last_version as usize);
        Ok(())
    }

/////在上面的函数里面实际上也调用了这个函数这个函数是说吧这一堆txns都执行一遍之后更新相关的各种状态
    fn save_transactions_impl(
        &self,
        txns_to_commit: &[TransactionToCommit],
        first_version: u64,
        cur_state_root_hash: HashValue,
        mut batch: &mut SchemaBatch,
    ) -> Result<HashValue> {
        
        ///
        let last_version = first_version + txns_to_commit.len() as u64 - 1;

        // Account state updates. Gather account state root hashes
        let account_state_sets = txns_to_commit
            .iter()
            .map(|txn_to_commit| txn_to_commit.account_states().clone())
            .collect::<Vec<_>>();
        let state_root_hashes = self.state_store.put_account_state_sets(
            account_state_sets,
            first_version,
            cur_state_root_hash,
            &mut batch,
        )?;

        // Event updates. Gather event accumulator root hashes.
        let event_root_hashes = zip_eq(first_version..=last_version, txns_to_commit)
            .map(|(ver, txn_to_commit)| {
                self.event_store
                    .put_events(ver, txn_to_commit.events(), &mut batch)
            })
            .collect::<Result<Vec<_>>>()?;

        // Transaction updates. Gather transaction hashes.
        zip_eq(first_version..=last_version, txns_to_commit)
            .map(|(ver, txn_to_commit)| {
                self.transaction_store
                    .put_transaction(ver, txn_to_commit.signed_txn(), &mut batch)
            })
            .collect::<Result<()>>()?;
        let txn_hashes = txns_to_commit
            .iter()
            .map(|txn_to_commit| txn_to_commit.signed_txn().hash())
            .collect::<Vec<_>>();
        let gas_amounts = txns_to_commit
            .iter()
            .map(TransactionToCommit::gas_used)
            .collect::<Vec<_>>();

        // Transaction accumulator updates. Get result root hash.
        let txn_infos = izip!(
            txn_hashes,
            state_root_hashes,
            event_root_hashes,
            gas_amounts
        )
        .map(|(t, s, e, g)| TransactionInfo::new(t, s, e, g))
        .collect::<Vec<_>>();
        assert_eq!(txn_infos.len(), txns_to_commit.len());

        let new_root_hash =
            self.ledger_store
                .put_transaction_infos(first_version, &txn_infos, &mut batch)?;

        Ok(new_root_hash)
    }

 ```

####update_to_latest_ledger:

```rust
/// This backs the `UpdateToLatestLedger` public read API which returns the latest
    /// [`LedgerInfoWithSignatures`] together with items requested and proofs relative to the same
    /// ledger info.

////在execution阶段executor会调用这个，就是先要返回最新的ledger，同时他还会发一长串request_items,这个函数后面的一大堆类似switch的语句就是在说判断一下这个request是像要什么，然后给他处理好。最后所有的结果一曲返回回去
    pub fn update_to_latest_ledger(
        &self,
        _client_known_version: u64,
        request_items: Vec<RequestItem>,
    ) -> Result<(
        Vec<ResponseItem>,
        LedgerInfoWithSignatures,
        Vec<ValidatorChangeEventWithProof>,
    )> {
        error_if_too_many_requested(request_items.len() as u64, MAX_REQUEST_ITEMS)?;

        // Get the latest ledger info and signatures
        let ledger_info_with_sigs = self.ledger_store.get_latest_ledger_info()?;
        let ledger_version = ledger_info_with_sigs.ledger_info().version();

        // Fulfill all request items
        let response_items = request_items
            .into_iter()
            .map(|request_item| match request_item {
                RequestItem::GetAccountState { address } => Ok(ResponseItem::GetAccountState {
                    account_state_with_proof: self.get_account_state_with_proof(
                        address,
                        ledger_version,
                        ledger_version,
                    )?,
                }),
                RequestItem::GetAccountTransactionBySequenceNumber {
                    account,
                    sequence_number,
                    fetch_events,
                } => {
                    let signed_transaction_with_proof = self.get_txn_by_account_and_seq(
                        account,
                        sequence_number,
                        ledger_version,
                        fetch_events,
                    )?;

                    let proof_of_current_sequence_number = match signed_transaction_with_proof {
                        Some(_) => None,
                        None => Some(self.get_account_state_with_proof(
                            account,
                            ledger_version,
                            ledger_version,
                        )?),
                    };

                    Ok(ResponseItem::GetAccountTransactionBySequenceNumber {
                        signed_transaction_with_proof,
                        proof_of_current_sequence_number,
                    })
                }

                RequestItem::GetEventsByEventAccessPath {
                    access_path,
                    start_event_seq_num,
                    ascending,
                    limit,
                } => {
                    let (events_with_proof, proof_of_latest_event) = self
                        .get_events_by_event_access_path(
                            &access_path,
                            start_event_seq_num,
                            ascending,
                            limit,
                            ledger_version,
                        )?;
                    Ok(ResponseItem::GetEventsByEventAccessPath {
                        events_with_proof,
                        proof_of_latest_event,
                    })
                }
                RequestItem::GetTransactions {
                    start_version,
                    limit,
                    fetch_events,
                } => {
                    let txn_list_with_proof =
                        self.get_transactions(start_version, limit, ledger_version, fetch_events)?;

                    Ok(ResponseItem::GetTransactions {
                        txn_list_with_proof,
                    })
                }
            })
            .collect::<Result<Vec<_>>>()?;

        Ok((
            response_items,
            ledger_info_with_sigs,
            vec![], /* TODO: validator_change_events */
        ))
    }

```

### save类操作和上面的update to the latest ledger都是executor调用的操作

#### 两类API：excution和state stnchronizer使用的几个api，他们的使用场景在相关模块的代码中能够咋熬到

```rust
// =========================== Execution Internal APIs ========================================

    /// Gets an account state by account address, out of the ledger state indicated by the state
    /// Merkle tree root hash.
    ///
    /// This is used by the executor module internally.
    pub fn get_account_state_with_proof_by_state_root(
        &self,
        address: AccountAddress,
        state_root: HashValue,
    ) -> Result<(Option<AccountStateBlob>, SparseMerkleProof)> {
        self.state_store
            .get_account_state_with_proof_by_state_root(address, state_root)
    }

    /// Gets information needed from storage during the startup of the executor module.
    ///
    /// This is used by the executor module internally.
    pub fn get_executor_startup_info(&self) -> Result<Option<ExecutorStartupInfo>> {
        // Get the latest ledger info. Return None if not bootstrapped.
        let ledger_info_with_sigs = match self.ledger_store.get_latest_ledger_info_option()? {
            Some(x) => x,
            None => return Ok(None),
        };
        let ledger_info = ledger_info_with_sigs.ledger_info().clone();

        let (latest_version, txn_info) = self.ledger_store.get_latest_transaction_info()?;

        let account_state_root_hash = txn_info.state_root_hash();

        let ledger_frozen_subtree_hashes = self
            .ledger_store
            .get_ledger_frozen_subtree_hashes(latest_version)?;

        Ok(Some(ExecutorStartupInfo {
            ledger_info,
            latest_version,
            account_state_root_hash,
            ledger_frozen_subtree_hashes,
        }))
    }

    // ======================= State Synchronizer Internal APIs ===================================
    /// Gets a batch of transactions for the purpose of synchronizing state to another node.
    ///
    /// This is used by the State Synchronizer module internally.
    pub fn get_transactions(
        &self,
        start_version: Version,
        limit: u64,
        ledger_version: Version,
        fetch_events: bool,
    ) -> Result<TransactionListWithProof> {
        error_if_too_many_requested(limit, MAX_LIMIT)?;

        if start_version > ledger_version || limit == 0 {
            return Ok(TransactionListWithProof::new_empty());
        }

        let limit = std::cmp::min(limit, ledger_version - start_version + 1);
        let txn_and_txn_info_list = (start_version..start_version + limit)
            .into_iter()
            .map(|version| {
                Ok((
                    self.transaction_store.get_transaction(version)?,
                    self.ledger_store.get_transaction_info(version)?,
                ))
            })
            .collect::<Result<Vec<_>>>()?;
        let proof_of_first_transaction = Some(
            self.ledger_store
                .get_transaction_proof(start_version, ledger_version)?,
        );
        let proof_of_last_transaction = if limit == 1 {
            None
        } else {
            Some(
                self.ledger_store
                    .get_transaction_proof(start_version + limit - 1, ledger_version)?,
            )
        };
        let events = if fetch_events {
            Some(
                (start_version..start_version + limit)
                    .into_iter()
                    .map(|version| Ok(self.event_store.get_events_by_version(version)?))
                    .collect::<Result<Vec<_>>>()?,
            )
        } else {
            None
        };

        Ok(TransactionListWithProof::new(
            txn_and_txn_info_list,
            events,
            Some(start_version),
            proof_of_first_transaction,
            proof_of_last_transaction,
        ))
    }

```



#### private APIs:

```rust
 // ================================== Private APIs ==================================
    /// Write the whole schema batch including all data necessary to mutate the ledge
    /// state of some transaction by leveraging rocksdb atomicity support.
///没看懂，因为schhema模块还没看，暂时也不准备看了。。。。。。。。。。。。。。。。。。。。
    fn commit(&self, batch: SchemaBatch) -> Result<()> {
        self.db.write_schemas(batch)?;

        match self.db.get_approximate_sizes_cf() {
            Ok(cf_sizes) => {
                for (cf_name, size) in cf_sizes {
                    OP_COUNTER.set(&format!("cf_size_bytes_{}", cf_name), size as usize);
                }
            }
            Err(err) => warn!(
                "Failed to get approximate size of column families: {}.",
                err
            ),
        }

        Ok(())
    }

    fn get_account_seq_num_by_version(
        &self,
        address: AccountAddress,
        version: Version,
    ) -> Result<u64> {
        let (account_state_blob, _proof) = self
            .state_store
            .get_account_state_with_proof_by_state_root(
                address,
                self.ledger_store
                    .get_transaction_info(version)?
                    .state_root_hash(),
            )?;

        // If an account does not exist, we treat it as if it has sequence number 0.
        Ok(get_account_resource_or_default(&account_state_blob)?.sequence_number())
    }

    fn get_transaction_with_proof(
        &self,
        version: Version,
        ledger_version: Version,
        fetch_events: bool,
    ) -> Result<SignedTransactionWithProof> {
        let proof = {
            let (txn_info, txn_info_accumulator_proof) = self
                .ledger_store
                .get_transaction_info_with_proof(version, ledger_version)?;
            SignedTransactionProof::new(txn_info_accumulator_proof, txn_info)
        };
        let signed_transaction = self.transaction_store.get_transaction(version)?;

        // If events were requested, also fetch those.
        let events = if fetch_events {
            Some(self.event_store.get_events_by_version(version)?)
        } else {
            None
        };

        Ok(SignedTransactionWithProof {
            version,
            signed_transaction,
            events,
            proof,
        })
    }
}

// Convert requested range and order to a range in ascending order.
fn get_first_seq_num_and_limit(ascending: bool, cursor: u64, limit: u64) -> Result<(u64, u64)> {
    ensure!(limit > 0, "limit should > 0, got {}", limit);

    Ok(if ascending {
        (cursor, limit)
    } else if limit <= cursor {
        (cursor - limit + 1, limit)
    } else {
        (0, cursor + 1)
    })
}

```





