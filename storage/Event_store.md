# Event_store

## 功能：这一部分提供event的存储，对外提供api 来获取/插入event

```rust

pub(crate) struct EventStore {
    db: Arc<DB>,//可以看出实际上eventStore就是一个db
}

impl EventStore {
    pub fn new(db: Arc<DB>) -> Self {
        Self { db }
    }
////在下面的一系列实现中，可以看出每一个event跟触发他的transaction的version是紧密联系的，还有一些函数使用了他在同一个version中各个事件中的index
    
    
    //一个transaction可以触发一系列的事件，然后每一个event都会注明是哪个version的transaction触发的他，所以我们可以根据transaction的version来获取这一交易触发的所有事件。
    /// Get all of the events given a transaction version.
    /// We don't need a proof for this because it's only used to get all events
    /// for a version which can be proved from the root hash of the event tree.
    pub fn get_events_by_version(&self, version: Version) -> Result<Vec<ContractEvent>> {
        let mut events = vec![];

        let mut iter = self.db.iter::<EventSchema>(ReadOptions::default())?;
        // Grab the first event and then iterate until we get all events for this version.
        iter.seek(&version)?;
        while let Some(((ver, index), event)) = iter.next().transpose()? {
            if ver != version {
                break;
            }
            events.push(event);
        }

        Ok(events)
    }

    /// Get the event raw data given transaction version and the index of the event queried.
///这个一并返回了proof，是获取某个version的transaction触发的，第index个事件
    pub fn get_event_with_proof_by_version_and_index(
        &self,
        version: Version,
        index: u64,
    ) -> Result<(ContractEvent, AccumulatorProof)> {
        // Get event content.
        let event = self
            .db
            .get::<EventSchema>(&(version, index))?
            .ok_or_else(|| LibraDbError::NotFound(format!("Event {} of Txn {}", index, version)))?;

        // Get the number of events in total for the transaction at `version`.
        let mut iter = self.db.iter::<EventSchema>(ReadOptions::default())?;
        iter.seek_for_prev(&(version + 1))?;
        let num_events = match iter.next().transpose()? {
            Some(((ver, index), _)) if ver == version => index + 1,
            _ => unreachable!(), // since we've already got at least one event above
        };

        // Get proof.
        let proof =
            Accumulator::get_proof(&EventHashReader::new(self, version), num_events, index)?;

        Ok((event, proof))
    }

    fn get_txn_ver_by_seq_num(&self, access_path: &AccessPath, seq_num: u64) -> Result<u64> {
        let (ver, _) = self
            .db
            .get::<EventByAccessPathSchema>(&(access_path.clone(), seq_num))?
            .ok_or_else(|| format_err!("Index entry should exist for seq_num {}", seq_num))?;
        Ok(ver)
    }

    /// Get the latest sequence number on `access_path` considering all transactions with versions
    /// no greater than `ledger_version`.
    ////在这个path上最新的sequence number
    pub fn get_latest_sequence_number(
        &self,
        ledger_version: Version,
        access_path: &AccessPath,
    ) -> Result<Option<u64>> {
        let mut iter = self
            .db
            .iter::<EventByAccessPathSchema>(ReadOptions::default())?;
        iter.seek_for_prev(&(access_path.clone(), u64::max_value()));
        if let Some(res) = iter.next() {
            let ((path, mut seq), (ver, _idx)) = res?;
            if path == *access_path {
                if ver <= ledger_version {
                    return Ok(Some(seq));
                }

                // Queries tend to base on very recent ledger infos, so first try to linear search
                // from the most recent end, for limited tries.
                // TODO: Optimize: Physical store use reverse order.
                ////先线性查找
                let mut n_try_recent = 10;
                #[cfg(test)]
                let mut n_try_recent = 1;
                while seq > 0 && n_try_recent > 0 {
                    seq -= 1;
                    n_try_recent -= 1;
                    let ver = self.get_txn_ver_by_seq_num(access_path, seq)?;
                    if ver <= ledger_version {
                        return Ok(Some(seq));
                    }
                }

                // Fall back to binary search if the above short linear search didn't work out.
                let (mut begin, mut end) = (0, seq);
                while begin < end {
                    let mid = end - (end - begin) / 2;
                    let ver = self.get_txn_ver_by_seq_num(access_path, mid)?;
                    if ver <= ledger_version {
                        begin = mid;
                    } else {
                        end = mid - 1;
                    }
                }
                return Ok(Some(begin));
            }
        }
        Ok(None)
    }

    /// Given access path and start sequence number, return events identified by transaction index
    /// and index among all events yielded by the same transaction. Result won't contain records
    /// with a txn_version > `ledger_version` and is in ascending order.
    pub fn lookup_events_by_access_path(
        &self,
        access_path: &AccessPath,
        start_seq_num: u64,
        limit: u64,
        ledger_version: u64,
    ) -> Result<
        Vec<(
            u64,     // sequence number
            Version, // transaction version it belongs to
            u64,     // index among events for the same transaction
        )>,
    > {
        let mut iter = self
            .db
            .iter::<EventByAccessPathSchema>(ReadOptions::default())?;
        iter.seek(&(access_path.clone(), start_seq_num))?;

        let mut result = Vec::new();
        let mut cur_seq = start_seq_num;
        for res in iter.take(limit as usize) {
            let ((path, seq), (ver, idx)) = res?;
            if path != *access_path || ver > ledger_version {
                break;
            }
            ensure!(
                seq == cur_seq,
                "DB corrupt: Sequence number not continuous, expected: {}, actual: {}.",
                cur_seq,
                seq
            );
            result.push((seq, ver, idx));
            cur_seq += 1;
        }

        Ok(result)
    }

    /// Save contract events yielded by the transaction at `version` and return root hash of the
    /// event accumulator formed by these events.
    pub fn put_events(
        &self,
        version: u64,
        events: &[ContractEvent],
        batch: &mut SchemaBatch,
    ) -> Result<HashValue> {
        // EventSchema and EventByAccessPathSchema updates
        events
            .iter()
            .enumerate()
            .map(|(idx, event)| {
                batch.put::<EventSchema>(&(version, idx as u64), event)?;
                batch.put::<EventByAccessPathSchema>(
                    &(event.access_path().clone(), event.sequence_number()),
                    &(version, idx as u64),
                )?;
                Ok(())
            })
            .collect::<Result<()>>()?;

        // EventAccumulatorSchema updates
        let event_hashes: Vec<HashValue> = events.iter().map(ContractEvent::hash).collect();
        let (root_hash, writes) = EmptyAccumulator::append(&EmptyReader, 0, &event_hashes)?;
        writes
            .into_iter()
            .map(|(pos, hash)| batch.put::<EventAccumulatorSchema>(&(version, pos), &hash))
            .collect::<Result<()>>()?;

        Ok(root_hash)
    }
}
```

