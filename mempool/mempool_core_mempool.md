# core_mempool:

## 目录：

* index.rs:
* mempool.rs:
* transaction_store.rs
* transtion.rs:

## 代码实现：

### mempool.rs:

**组合起来各个模块，包括：各种为快速查找txns实现的的index（索引），sequenc number map等等部分，来实现增删txn，同步，取block等等功能。**

**注：这一部分是mempool中最底层的部分，代码逻辑比较琐碎，如果阅读起来有困难可以只大致了解提供了什么API，直接看后面的更上层的部分。**

#### 定义:

```rust
pub struct Mempool {
    // stores metadata of all transactions in mempool (of all states)
    transactions: TransactionStore,
   //存储所有交易的原始的样子
    sequence_number_cache: LruCache<AccountAddress, u64>,
    //记录每个账户的sequence number，用来排序和选择合法交易组成下一区块
    
    // temporary DS. TODO: eventually retire it
    // for each transaction, entry with timestamp is added when transaction enters mempool
    // used to measure e2e latency of transaction in system, as well as time it takes to pick it up
    // by consensus
    metrics_cache: TtlCache<(AccountAddress, u64), i64>,
    //记录每一笔交易的ttl时间，后面ttl based garbage collection会用
    pub system_transaction_timeout: Duration,
}

/// TransactionStore is in-memory storage for all transactions in mempool
pub struct TransactionStore {
    // main DS
    transactions: HashMap<AccountAddress, AccountTransactions>,

    // indexes
    priority_index: PriorityIndex,
    // TTLIndex based on client-specified expiration time
    expiration_time_index: TTLIndex,
    // TTLIndex based on system expiration time
    // we keep it separate from `expiration_time_index` so Mempool can't be clogged
    //  by old transactions even if it hasn't received commit callbacks for a while
    system_ttl_index: TTLIndex,
    timeline_index: TimelineIndex,
    // keeps track of "non-ready" txns (transactions that can't be included in next block)
    parking_lot_index: ParkingLotIndex,
  	//这里面的txn在收到信号之后会进入order queue来准备进入下一个区块。（如：上一个sequence number的交易已经被加入区块了，会发送信号）
    // configuration
    capacity: usize,/////////在这里设置最多接收多少个txns
    capacity_per_user: usize,
}
```

#### impl mempool：

```rust
impl Mempool {
    pub(crate) fn new(config: &NodeConfig) -> Self {
        Mempool {
            transactions: TransactionStore::new(&config.mempool),
            sequence_number_cache: LruCache::new(config.mempool.sequence_cache_capacity),
            metrics_cache: TtlCache::new(config.mempool.capacity),
            system_transaction_timeout: Duration::from_secs(
                config.mempool.system_transaction_timeout_secs,
            ),
        }
    }

    /// This function will be called once the transaction has been stored
    pub(crate) fn remove_transaction(
        &mut self,
        sender: &AccountAddress,
        sequence_number: u64,
        is_rejected: bool,////这个txn是否被接受
    ) {
        debug!(
            "[Mempool] Removing transaction from mempool: {}:{}",
            sender, sequence_number
        );
        self.log_latency(sender.clone(), sequence_number, "e2e.latency");////把延迟写到日志里面
        self.metrics_cache.remove(&(*sender, sequence_number));

        // update current cached sequence number for account
        let cached_value = self
            .sequence_number_cache
            .remove(sender)
            .unwrap_or_default();

        let new_sequence_number = if is_rejected {
            min(sequence_number, cached_value)
        } else {
            max(cached_value, sequence_number + 1)
        };
        self.sequence_number_cache
            .insert(sender.clone(), new_sequence_number);

        self.transactions
            .commit_transaction(&sender, sequence_number);
    }

    fn log_latency(&mut self, account: AccountAddress, sequence_number: u64, metric: &str) {
        if let Some(&creation_time) = self.metrics_cache.get(&(account, sequence_number)) {
            OP_COUNTERS.observe(
                metric,
                (Utc::now().timestamp_millis() - creation_time) as f64,
            );
        }
    }

    fn get_required_balance(&mut self, txn: &SignedTransaction, gas_amount: u64) -> u64 {
        txn.gas_unit_price() * gas_amount + self.transactions.get_required_balance(&txn.sender())
    }

    /// Used to add a transaction to the Mempool
    /// Performs basic validation: checks account's balance and sequence number
    pub(crate) fn add_txn(
        &mut self,
        txn: SignedTransaction,
        gas_amount: u64,
        db_sequence_number: u64,
        balance: u64,
        timeline_state: TimelineState,
    ) -> MempoolAddTransactionStatus {
        debug!(
            "[Mempool] Adding transaction to mempool: {}:{}",
            &txn.sender(),
            db_sequence_number
        );

        let required_balance = self.get_required_balance(&txn, gas_amount);
        if balance < required_balance {
            return MempoolAddTransactionStatus::new(
                MempoolAddTransactionStatusCode::InsufficientBalance,
                format!(
                    "balance: {}, required_balance: {}, gas_amount: {}",
                    balance, required_balance, gas_amount
                ),
            );
        }

        let cached_value = self.sequence_number_cache.get_mut(&txn.sender());
        let sequence_number = match cached_value {
            Some(value) => max(*value, db_sequence_number),
            None => db_sequence_number,
        };
        self.sequence_number_cache
            .insert(txn.sender(), sequence_number);
        // don't accept old transactions (e.g. seq is less than account's current seq_number)
        if txn.sequence_number() < sequence_number {
            return MempoolAddTransactionStatus::new(
                MempoolAddTransactionStatusCode::InvalidSeqNumber,
                format!(
                    "transaction sequence number is {}, current sequence number is  {}",
                    txn.sequence_number(),
                    sequence_number,
                ),
            );
        }

        let expiration_time = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .expect("init timestamp failure")
            + self.system_transaction_timeout;
        if timeline_state != TimelineState::NonQualified {
            self.metrics_cache.insert(
                (txn.sender(), txn.sequence_number()),
                Utc::now().timestamp_millis(),
                Duration::from_secs(100),
            );
        }

        let txn_info = MempoolTransaction::new(txn, expiration_time, gas_amount, timeline_state);

        let status = self.transactions.insert(txn_info, sequence_number);
        OP_COUNTERS.inc(&format!("insert.{:?}", status));
        status
    }

    /// Fetches next block of transactions for consensus
    /// `batch_size` - size of requested block
    /// `seen_txns` - transactions that were sent to Consensus but were not committed yet
    ///  Mempool should filter out such transactions
    pub(crate) fn get_block(
        &mut self,
        batch_size: u64,
        mut seen: HashSet<TxnPointer>,
    ) -> Vec<SignedTransaction> {
        let mut result = vec![];
        // Helper DS. Helps to mitigate scenarios where account submits several transactions
        // with increasing gas price (e.g. user submits transactions with sequence number 1, 2
        // and gas_price 1, 10 respectively)
        // Later txn has higher gas price and will be observed first in priority index iterator,
        // but can't be executed before first txn. Once observed, such txn will be saved in
        // `skipped` DS and rechecked once it's ancestor becomes available
        let mut skipped = HashSet::new();

        // iterate over the queue of transactions based on gas price
        'main: for txn in self.transactions.iter_queue() {
            if seen.contains(&TxnPointer::from(txn)) {
                continue;
            }
            let mut seq = txn.sequence_number;
            let account_sequence_number = self.sequence_number_cache.get_mut(&txn.address);
            let seen_previous = seq > 0 && seen.contains(&(txn.address, seq - 1));
            // include transaction if it's "next" for given account or
            // we've already sent its ancestor to Consensus
            if seen_previous || account_sequence_number == Some(&mut seq) {
                let ptr = TxnPointer::from(txn);
                seen.insert(ptr);
                result.push(ptr);
                if (result.len() as u64) == batch_size {
                    break;
                }

                // check if we can now include some transactions
                // that were skipped before for given account
                let mut skipped_txn = (txn.address, seq + 1);
                while skipped.contains(&skipped_txn) {
                    seen.insert(skipped_txn);
                    result.push(skipped_txn);
                    if (result.len() as u64) == batch_size {
                        break 'main;
                    }
                    skipped_txn = (txn.address, skipped_txn.1 + 1);
                }
            } else {
                skipped.insert(TxnPointer::from(txn));
            }
        }
        // convert transaction pointers to real values
        let block: Vec<_> = result
            .into_iter()
            .filter_map(|(address, seq)| self.transactions.get(&address, seq))
            .collect();
        for transaction in &block {
            self.log_latency(
                transaction.sender(),
                transaction.sequence_number(),
                "txn_pre_consensus_ms",
            );
        }
        block
    }

    /// TTL based garbage collection. Remove all transactions that got expired
    pub(crate) fn gc_by_system_ttl(&mut self) {
        self.transactions.gc_by_system_ttl();
    }

    /// Garbage collection based on client-specified expiration time
    pub(crate) fn gc_by_expiration_time(&mut self, block_time: Duration) {
        self.transactions.gc_by_expiration_time(block_time);
    }

    /// Read `count` transactions from timeline since `timeline_id`
    /// Returns block of transactions and new last_timeline_id
    pub(crate) fn read_timeline(
        &mut self,
        timeline_id: u64,
        count: usize,
    ) -> (Vec<SignedTransaction>, u64) {
        self.transactions.read_timeline(timeline_id, count)
    }

    /// Check the health of core mempool.
    pub(crate) fn health_check(&self) -> bool {
        self.transactions.health_check()
    }
}
```

#### impl transaction_store：

```rust
/// fetch transaction by account address + sequence_number
    pub(crate) fn get(
        &self,
        address: &AccountAddress,
        sequence_number: u64,
    ) -> Option<SignedTransaction> {
        if let Some(txns) = self.transactions.get(&address) {
            if let Some(txn) = txns.get(&sequence_number) {
                return Some(txn.txn.clone());
            }
        }
        None
    }

    /// insert transaction into TransactionStore
    /// performs validation checks and updates indexes
    pub(crate) fn insert(
        &mut self,
        txn: MempoolTransaction,
        current_sequence_number: u64,
    ) -> MempoolAddTransactionStatus {
        let (is_update, status) = self.check_for_update(&txn);//这一交易已经在mempool中了，这次交易是为了进行更新。（我们允许对交易的gas price进行更新来加速交易进入区块，这些更新在check_for update函数中完成）
        if is_update {
            return status;
        }
        if self.check_if_full() {//mempool满了，直接返回
            return MempoolAddTransactionStatus::new(
                MempoolAddTransactionStatusCode::MempoolIsFull,
                format!(
                    "mempool size: {}, capacity: {}",
                    self.system_ttl_index.size(),
                    self.capacity,
                ),
            );
        }

        let address = txn.get_sender();
        let sequence_number = txn.get_sequence_number();

        self.transactions
            .entry(address)
            .or_insert_with(AccountTransactions::new);

        if let Some(txns) = self.transactions.get_mut(&address) {//交易太长，直接返回
            // capacity check
            if txns.len() >= self.capacity_per_user {
                return MempoolAddTransactionStatus::new(
                    MempoolAddTransactionStatusCode::TooManyTransactions,
                    format!(
                        "txns length: {} capacity per user: {}",
                        txns.len(),
                        self.capacity_per_user,
                    ),
                );
            }

            // insert into storage and other indexes
            self.system_ttl_index.insert(&txn);
            self.expiration_time_index.insert(&txn);
            txns.insert(sequence_number, txn);
            OP_COUNTERS.set("txn.system_ttl_index", self.system_ttl_index.size());
        }
        self.process_ready_transactions(&address, current_sequence_number);
        MempoolAddTransactionStatus::new(MempoolAddTransactionStatusCode::Valid, "".to_string())
    }
```

#### 将池中交易删除（交易已经store进storage中）

```rust
 /// This function will be called once the transaction has been stored
pub(crate) fn remove_transaction(
        &mut self,
        sender: &AccountAddress,
        sequence_number: u64,
        is_rejected: bool,
    ) {
        debug!(
            "[Mempool] Removing transaction from mempool: {}:{}",
            sender, sequence_number
        );
        //记录延迟，写进日志里
        self.log_latency(sender.clone(), sequence_number, "e2e.latency");
        self.metrics_cache.remove(&(*sender, sequence_number));

        // update current cached sequence number for account
        let cached_value = self
            .sequence_number_cache
            .remove(sender)
            .unwrap_or_default();
		
        let new_sequence_number = if is_rejected {
            min(sequence_number, cached_value)
        } else {
            max(cached_value, sequence_number + 1)
        };
        self.sequence_number_cache
            .insert(sender.clone(), new_sequence_number);
		
        self.transactions
            .commit_transaction(&sender, sequence_number);
    }

```

#### garbage colletion：回收超时的交易

```rust
 //两种base的garbage collection
/// TTL based garbage collection. Remove all transactions that got expired
    pub(crate) fn gc_by_system_ttl(&mut self) {
        self.transactions.gc_by_system_ttl();
    }

    /// Garbage collection based on client-specified expiration time
    pub(crate) fn gc_by_expiration_time(&mut self, block_time: Duration) {
        self.transactions.gc_by_expiration_time(block_time);
    }
```

```rust
///Garbage collection在transaction中的实现

/// GC old transactions
    pub(crate) fn gc_by_system_ttl(&mut self) {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .expect("init timestamp failure");

        self.gc(now, true);
    }

    /// GC old transactions based on client-specified expiration time
    pub(crate) fn gc_by_expiration_time(&mut self, block_time: Duration) {
        self.gc(block_time, false);
    }

    fn gc(&mut self, now: Duration, by_system_ttl: bool) {
        let (index_name, index) = if by_system_ttl {
            ("gc.system_ttl_index", &mut self.system_ttl_index)
        } else {
            ("gc.expiration_time_index", &mut self.expiration_time_index)
        };
        OP_COUNTERS.inc(index_name);

        for key in index.gc(now) {
            if let Some(txns) = self.transactions.get_mut(&key.address) {
                // mark all following transactions as non-ready
                for (_, t) in txns.range((Bound::Excluded(key.sequence_number), Bound::Unbounded)) {
                    self.parking_lot_index.insert(&t);
                    self.priority_index.remove(&t);
                    self.timeline_index.remove(&t);
                }
                if let Some(txn) = txns.remove(&key.sequence_number) {
                    let is_active = self.priority_index.contains(&txn);
                    let status = if is_active { "active" } else { "parked" };
                    OP_COUNTERS.inc(&format!("{}.{}", index_name, status));
                    self.index_remove(&txn);
                }
            }
        }
        OP_COUNTERS.set("txn.system_ttl_index", self.system_ttl_index.size());
    }
```

#### 从池中取出区块给consensus

```rust
/// Fetches next block of transactions for consensus
    /// `batch_size` - size of requested block
    /// `seen_txns` - transactions that were sent to Consensus but were not committed yet
    ///  Mempool should filter out such transactions
    pub(crate) fn get_block(
        &mut self,
        batch_size: u64,
        mut seen: HashSet<TxnPointer>,
    ) -> Vec<SignedTransaction> {
        let mut result = vec![];
        // Helper DS. Helps to mitigate scenarios where account submits several transactions
        // with increasing gas price (e.g. user submits transactions with sequence number 1, 2
        // and gas_price 1, 10 respectively)
        // Later txn has higher gas price and will be observed first in priority index iterator,
        // but can't be executed before first txn. Once observed, such txn will be saved in
        // `skipped` DS and rechecked once it's ancestor becomes available
        let mut skipped = HashSet::new();

        // iterate over the queue of transactions based on gas price
        'main: for txn in self.transactions.iter_queue() {
            if seen.contains(&TxnPointer::from(txn)) {
                continue;
            }
            let mut seq = txn.sequence_number;
            let account_sequence_number = self.sequence_number_cache.get_mut(&txn.address);
            let seen_previous = seq > 0 && seen.contains(&(txn.address, seq - 1));
            // include transaction if it's "next" for given account or
            // we've already sent its ancestor to Consensus
            if seen_previous || account_sequence_number == Some(&mut seq) {
                let ptr = TxnPointer::from(txn);
                seen.insert(ptr);
                result.push(ptr);
                if (result.len() as u64) == batch_size {
                    break;
                }

                // check if we can now include some transactions
                // that were skipped before for given account
                let mut skipped_txn = (txn.address, seq + 1);
                while skipped.contains(&skipped_txn) {
                    seen.insert(skipped_txn);
                    result.push(skipped_txn);
                    if (result.len() as u64) == batch_size {
                        break 'main;
                    }
                    skipped_txn = (txn.address, skipped_txn.1 + 1);
                }
            } else {
                skipped.insert(TxnPointer::from(txn));
            }
        }
        // convert transaction pointers to real values
        let block: Vec<_> = result
            .into_iter()
            .filter_map(|(address, seq)| self.transactions.get(&address, seq))
            .collect();
        for transaction in &block {
            self.log_latency(
                transaction.sender(),
                transaction.sequence_number(),
                "txn_pre_consensus_ms",
            );
        }
        block
    }
```

#### transaction_store.rs剩余部分:

```rust
	 /// fixes following invariants:
    /// all transactions of given account that are sequential to current sequence number
    /// supposed to be included in both PriorityIndex (ordering for Consensus) and
    /// TimelineIndex (txns for SharedMempool)
    /// Other txns are considered to be "non-ready" and should be added to ParkingLotIndex
   

//ready queue和non-readyqueue的分类
fn process_ready_transactions(
        &mut self,
        address: &AccountAddress,
        current_sequence_number: u64,
    ) {
        if let Some(txns) = self.transactions.get_mut(&address) {
            let mut sequence_number = current_sequence_number;
            while let Some(txn) = txns.get_mut(&sequence_number) {
                self.priority_index.insert(txn);

                if txn.timeline_state == TimelineState::NotReady {
                    self.timeline_index.insert(txn);
                }
                sequence_number += 1;
            }
            for (_, txn) in txns.range_mut((Bound::Excluded(sequence_number), Bound::Unbounded)) {
                match txn.timeline_state {
                    TimelineState::Ready(_) => {}
                    _ => {
                        self.parking_lot_index.insert(&txn);
                    }
                }
            }
        }
    }

    /// handles transaction commit
    /// it includes deletion of all transactions with sequence number <= `sequence_number`
    /// and potential promotion of sequential txns to PriorityIndex/TimelineIndex
    pub(crate) fn commit_transaction(&mut self, account: &AccountAddress, sequence_number: u64) {
        if let Some(txns) = self.transactions.get_mut(&account) {
            // remove all previous seq number transactions for this account
            // This can happen if transactions are sent to multiple nodes and one of
            // nodes has sent the transaction to consensus but this node still has the
            // transaction sitting in mempool
            let mut active = txns.split_off(&(sequence_number + 1));
            let txns_for_removal = txns.clone();
            txns.clear();
            txns.append(&mut active);
			//active之前的全部被清空
            for transaction in txns_for_removal.values() {
                self.index_remove(transaction);
                //将清除的交易对应的index全部移除
            }
        }
        self.process_ready_transactions(account, sequence_number + 1);
    }
```

#### index.rs:这个文件实现了mempool中用到的各个index

priority index：用于选取交易组成区块

```rust

/// PriorityIndex represents main Priority Queue in Mempool
/// It's used to form transaction block for Consensus
/// Transactions are ordered by gas price. Second level ordering is done by expiration time
///
/// We don't store full content of transaction in index
/// Instead we use `OrderedQueueKey` - logical reference to transaction in main store
pub struct PriorityIndex {
    data: BTreeSet<OrderedQueueKey>,
}
pub type PriorityQueueIter<'a> = Rev<Iter<'a, OrderedQueueKey>>;
//。。。。。忽略常见的插入删除查找等操作。很简单的实现
//key的实现：
pub struct OrderedQueueKey {
    pub gas_price: u64,
    pub expiration_time: Duration,
    pub address: AccountAddress,
    pub sequence_number: u64,
}
///make_key函数：
fn make_key(&self, txn: &MempoolTransaction) -> OrderedQueueKey {
        OrderedQueueKey {
            gas_price: txn.get_gas_price(),
            expiration_time: txn.expiration_time,
            address: txn.get_sender(),
            sequence_number: txn.get_sequence_number(),
        }
    }

//实现key的排序的两个接口
impl PartialOrd for OrderedQueueKey {
    fn partial_cmp(&self, other: &OrderedQueueKey) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for OrderedQueueKey {//依据优先级依次比较
    fn cmp(&self, other: &OrderedQueueKey) -> Ordering {
        match self.gas_price.cmp(&other.gas_price) {
            Ordering::Equal => {}
            ordering => return ordering,
        }
        match self.expiration_time.cmp(&other.expiration_time).reverse() {
            Ordering::Equal => {}
            ordering => return ordering,
        }
        match self.address.cmp(&other.address) {
            Ordering::Equal => {}
            ordering => return ordering,
        }
        self.sequence_number.cmp(&other.sequence_number).reverse()
    }
}

```

TTLIndex：用于垃圾回收

```rust
/// TTLIndex is used to perform garbage collection of old transactions in Mempool
/// Periodically separate GC-like job queries this index to find out transactions that have to be
/// removed Index is represented as `BTreeSet<TTLOrderingKey>`
///   where `TTLOrderingKey` is logical reference to TxnInfo
/// Index is ordered by `TTLOrderingKey::expiration_time`
pub struct TTLIndex {
    data: BTreeSet<TTLOrderingKey>,
    get_expiration_time: Box<dyn Fn(&MempoolTransaction) -> Duration + Send + Sync>,
}

//一些接口，其中gc比较新颖
impl TTLIndex {
    pub(crate) fn new<F>(get_expiration_time: Box<F>) -> Self
    where
        F: Fn(&MempoolTransaction) -> Duration + 'static + Send + Sync,
    {
        Self {
            data: BTreeSet::new(),
            get_expiration_time,
        }
    }

    /// add transaction to index
    pub(crate) fn insert(&mut self, txn: &MempoolTransaction) {
        self.data.insert(self.make_key(&txn));
    }

    /// remove transaction from index
    pub(crate) fn remove(&mut self, txn: &MempoolTransaction) {
        self.data.remove(&self.make_key(&txn));
    }

    /// GC all old transactions
    ////删除所有old 的transactions
    pub(crate) fn gc(&mut self, now: Duration) -> Vec<TTLOrderingKey> {
        let ttl_key = TTLOrderingKey {
            expiration_time: now,
            address: AccountAddress::default(),
            sequence_number: 0,
        };

        let mut active = self.data.split_off(&ttl_key);//核心操作
        let ttl_transactions = self.data.iter().cloned().collect();
        self.data.clear();
        self.data.append(&mut active);
        ttl_transactions
    }

    fn make_key(&self, txn: &MempoolTransaction) -> TTLOrderingKey {
        TTLOrderingKey {
            expiration_time: (self.get_expiration_time)(txn),
            address: txn.get_sender(),
            sequence_number: txn.get_sequence_number(),
        }
    }

    pub(crate) fn size(&self) -> usize {
        self.data.len()
    }impl TTLIndex {
    pub(crate) fn new<F>(get_expiration_time: Box<F>) -> Self
    where
        F: Fn(&MempoolTransaction) -> Duration + 'static + Send + Sync,
    {
        Self {
            data: BTreeSet::new(),
            get_expiration_time,
        }
    }
//简单的插入和删除
    /// add transaction to index
    pub(crate) fn insert(&mut self, txn: &MempoolTransaction) {
        self.data.insert(self.make_key(&txn));
    }

    /// remove transaction from index
    pub(crate) fn remove(&mut self, txn: &MempoolTransaction) {
        self.data.remove(&self.make_key(&txn));
    }

    /// GC all old transactions
    pub(crate) fn gc(&mut self, now: Duration) -> Vec<TTLOrderingKey> {
        let ttl_key = TTLOrderingKey {
            expiration_time: now,
            address: AccountAddress::default(),
            sequence_number: 0,
        };

        let mut active = self.data.split_off(&ttl_key);
        let ttl_transactions = self.data.iter().cloned().collect();
        self.data.clear();
        self.data.append(&mut active);
        ttl_transactions
    }

    fn make_key(&self, txn: &MempoolTransaction) -> TTLOrderingKey {
        TTLOrderingKey {
            expiration_time: (self.get_expiration_time)(txn),
            address: txn.get_sender(),
            sequence_number: txn.get_sequence_number(),
        }
    }

    pub(crate) fn size(&self) -> usize {
        self.data.len()
    }
 }
    
    
    
//key的比较函数实现
   pub struct TTLOrderingKey {
    pub expiration_time: Duration,
    pub address: AccountAddress,
    pub sequence_number: u64,
}

impl Ord for TTLOrderingKey {
    fn cmp(&self, other: &TTLOrderingKey) -> Ordering {
        match self.expiration_time.cmp(&other.expiration_time) {//依据expiration time来排序
            Ordering::Equal => {
                (&self.address, self.sequence_number).cmp(&(&other.address, other.sequence_number))
            }
            ordering => ordering,
        }
    }
}
```

TimelineIndex：

 ```rust
/// TimelineIndex is ordered log of all transactions that are "ready" for broadcast
/// we only add transaction to index if it has a chance to be included in next consensus block
/// it means it's status != NotReady or it's sequential to other "ready" transaction
///
/// It's represented as Map <timeline_id, (Address, sequence_number)>
///    where timeline_id is auto increment unique id of "ready" transaction in local Mempool
///    (Address, sequence_number) is a logical reference to transaction content in main storage
pub struct TimelineIndex {
    timeline_id: u64,
    timeline: BTreeMap<u64, (AccountAddress, u64)>,
}


 /// read all transactions from timeline since <timeline_id>
    pub(crate) fn read_timeline(
        &mut self,
        timeline_id: u64,
        count: usize,
    ) -> Vec<(AccountAddress, u64)> {
        let mut batch = vec![];
        for (_, &(address, sequence_number)) in self
            .timeline
            .range((Bound::Excluded(timeline_id), Bound::Unbounded))
        {
            batch.push((address, sequence_number));
            if batch.len() == count { //到了count就停了
                break;
            }
        }
        batch
    }
 ```

ParkingLotIndex:在丢弃non—ready的交易时使用

```rust
/// ParkingLotIndex keeps track of "not_ready" transactions
/// e.g. transactions that can't be included in next block
/// (because their sequence number is too high)
/// we keep separate index to be able to efficiently evict them when Mempool is full

pub struct ParkingLotIndex {
    data: BTreeSet<TxnPointer>,
}
```

TxnPointer:维护一个每一笔交易的指针

```rust
/// Logical pointer to `MempoolTransaction`
/// Includes Account's address and transaction sequence number
pub type TxnPointer = (AccountAddress, u64);

impl From<&MempoolTransaction> for TxnPointer {
    fn from(transaction: &MempoolTransaction) -> Self {
        (transaction.get_sender(), transaction.get_sequence_number())
    }
}

impl From<&OrderedQueueKey> for TxnPointer {
    fn from(key: &OrderedQueueKey) -> Self {
        (key.address, key.sequence_number)
    }
}

```





