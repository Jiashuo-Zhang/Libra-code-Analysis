# shared_mempool

## 功能：实现shared_mempool，可以与别的validator等待共享

## 代码实现：

#### peer管理：

```rust
/// state of last sync with peer
/// `timeline_id` is position in log of ready transactions
/// `is_alive` - is connection healthy
#[derive(Clone)]
struct PeerSyncState {
    timeline_id: u64,
    is_alive: bool,
}

type PeerInfo = HashMap<PeerId, PeerSyncState>;
//map记录所有peer的状态

type PeerInfo = HashMap<PeerId, PeerSyncState>;

/// Outbound peer syncing event emitted by [`IntervalStream`].、
//通俗来说应该是所有需要发送的事件
#[derive(Debug)]
pub(crate) struct SyncEvent;

type IntervalStream = Pin<Box<dyn Stream<Item = Result<SyncEvent>> + Send + 'static>>;

#[derive(Copy, Clone, Debug, PartialEq)]
pub enum SharedMempoolNotification {
    Sync,
    PeerStateChange,
    NewTransactions,
}
/// Struct that owns all dependencies required by shared mempool routines
//通俗来说应该是所有需要加的锁
struct SharedMempool<V>
where
    V: TransactionValidation + 'static,
{
    mempool: Arc<Mutex<CoreMempool>>,
    network_sender: MempoolNetworkSender,
    config: MempoolConfig,
    storage_read_client: Arc<dyn StorageRead>,
    validator: Arc<V>,
    peer_info: Arc<Mutex<PeerInfo>>,
    subscribers: Vec<UnboundedSender<SharedMempoolNotification>>,
}

```

#### notify_subscribers

```rust

fn notify_subscribers(
    event: SharedMempoolNotification,
    subscribers: &[UnboundedSender<SharedMempoolNotification>],
) {
    for subscriber in subscribers {
        let _ = subscriber.unbounded_send(event);
    }
}
挨个通知订阅者
```

#### peer_handle:处理新发现peer和失去peer

```rust
/// new peer discovery handler
/// adds new entry to `peer_info`
fn new_peer(peer_info: &Mutex<PeerInfo>, peer_id: PeerId) {
    peer_info
        .lock()
        .expect("[shared mempool] failed to acquire peer_info lock")
        .entry(peer_id)
        .or_insert(PeerSyncState {
            timeline_id: 0,
            is_alive: true,
        })
        .is_alive = true;
}

/// lost peer handler. Marks connection as dead
fn lost_peer(peer_info: &Mutex<PeerInfo>, peer_id: PeerId) {
    if let Some(state) = peer_info
        .lock()
        .expect("[shared mempool] failed to acquire peer_info lock")
        .get_mut(&peer_id)
    {
        state.is_alive = false;
    }
}
```

#### sync_with_peers：与peer进行同步，把某个timelineid之后的数据发送给peer们，并更新peer的状态

```rust
/// sync routine
/// used to periodically broadcast ready to go transactions to peers
async fn sync_with_peers<'a>(
    peer_info: &'a Mutex<PeerInfo>,
    mempool: &'a Mutex<CoreMempool>,
    network_sender: &'a mut MempoolNetworkSender,
    batch_size: usize,
) {
    // Clone the underlying peer_info map and use this to sync and collect
    // state updates. We do this instead of holding the lock for the whole
    // function since that would hold the lock across await points which is bad.
    let peer_info_copy = peer_info
        .lock()
        .expect("[shared mempool] failed to acquire peer_info lock")
        .deref()
        .clone();

    let mut state_updates = vec![];
//对每一个peer都进行一次发送
    for (peer_id, peer_state) in peer_info_copy.into_iter() {
        if peer_state.is_alive {
            let timeline_id = peer_state.timeline_id;

            let (transactions, new_timeline_id) = mempool
                .lock()
                .expect("[shared mempool] failed to acquire mempool lock")
                .read_timeline(timeline_id, batch_size);
            ///从mempool里面取出timeline ID之后的所有交易

            if !transactions.is_empty() {
                OP_COUNTERS.inc_by("smp.sync_with_peers", transactions.len());
                let mut msg = MempoolSyncMsg::new();
                msg.set_peer_id(peer_id.into());
                msg.set_transactions(
                    transactions
                        .into_iter()
                        .map(IntoProto::into_proto)
                        .collect(),
                );
                //把交易们打包

                debug!(
                    "MempoolNetworkSender.send_to peer {} msg {:?}",
                    peer_id, msg
                );
                // Since this is a direct-send, this will only error if the network
                // module has unexpectedly crashed or shutdown.
                network_sender
                    .send_to(peer_id, msg)
                    .await
                    .expect("[shared mempool] failed to direct-send mempool sync message");
            }
            //把打包后的交易们通过network_sender发送给peer

            state_updates.push((peer_id, new_timeline_id));
            //因为我们是定期发送的，不是对面发请求我们回应，所以可能会有一系列peer或者network shutdown或者block 的情况，我们要记录下来这些状态，等待下面获取锁之后更新peer状态
        }
    }

    // Lock the shared peer_info and apply state updates.
    //加锁，更新peer状态
    let mut peer_info = peer_info
        .lock()
        .expect("[shared mempool] failed to acquire peer_info lock");
    for (peer_id, new_timeline_id) in state_updates {
        peer_info
            .entry(peer_id)
            .and_modify(|t| t.timeline_id = new_timeline_id);
    }
}
```



#### process_incoming_transactions:处理peer们发来的交易

```rust
// used to validate incoming transactions and add them to local Mempool
async fn process_incoming_transactions<V>(
    smp: SharedMempool<V>,
    peer_id: PeerId,
    transactions: Vec<SignedTransaction>,
) where
    V: TransactionValidation,
{
    let validations = join_all(
        transactions
            .iter()
            .map(|t| smp.validator.validate_transaction(t.clone()).compat()),
    )
    .await;//选出所有验证通过的交易，批量操作

    let account_states = join_all(
        transactions
            .iter()
            .map(|t| get_account_state(smp.storage_read_client.clone(), t.sender())),
    )
    .await;
//获得所有相关的account的state，批量操作
    let mut mempool = smp
        .mempool
        .lock()
        .expect("[shared mempool] failed to acquire mempool lock");

    for (idx, transaction) in transactions.into_iter().enumerate() {
        if let Ok(None) = validations[idx] {
            if let Ok((sequence_number, balance)) = account_states[idx] {
                let gas_cost = transaction.max_gas_amount();
                let insertion_result = mempool.add_txn(
                    transaction,
                    gas_cost,
                    sequence_number,
                    balance,
                    TimelineState::NonQualified,
                );
                if insertion_result.code == MempoolAddTransactionStatusCode::Valid {
                    OP_COUNTERS.inc(&format!("smp.transactions.success.{:?}", peer_id));
                }
            }//尝试加入mempool并返回状态
        }
    }
    notify_subscribers(SharedMempoolNotification::NewTransactions, &smp.subscribers);//通知订阅着有新的交易加入
}

```

#### outbound_sync_task:我理解为类似脉冲发生器的功能，定期的与外界进行sync_with_peers和notify_subscribers

```rust
/// This task handles [`SyncEvent`], which is periodically emitted for us to
/// broadcast ready to go transactions to peers.
async fn outbound_sync_task<V>(smp: SharedMempool<V>, mut interval: IntervalStream)
where
    V: TransactionValidation,
{
    let peer_info = smp.peer_info;
    let mempool = smp.mempool;
    let mut network_sender = smp.network_sender;
    let batch_size = smp.config.shared_mempool_batch_size;
    let subscribers = smp.subscribers;

    while let Some(sync_event) = interval.next().await {
        trace!("SyncEvent: {:?}", sync_event);
        match sync_event {
            Ok(_) => {
                sync_with_peers(&peer_info, &mempool, &mut network_sender, batch_size).await;
                notify_subscribers(SharedMempoolNotification::Sync, &subscribers);
            }
            Err(e) => {
                error!("Error in outbound_sync_task timer interval: {:?}", e);
                break;
            }
        }
    }

    crit!("SharedMempool outbound_sync_task terminated");
}
```

#### inbound_network_task：

```rust
/// This task handles inbound network events.
async fn inbound_network_task<V>(smp: SharedMempool<V>, network_events: MempoolNetworkEvents)
where
    V: TransactionValidation,
{
    let peer_info = smp.peer_info.clone();
    let subscribers = smp.subscribers.clone();
    let max_inbound_syncs = smp.config.shared_mempool_max_concurrent_inbound_syncs;

    // Handle the NewPeer/LostPeer events immediatedly, since they are not async
    // and we don't want to buffer them or let them get reordered. The inbound
    // direct-send messages are placed in a bounded FuturesUnordered queue and
    // allowed to execute concurrently. The .buffer_unordered() also correctly
    // handles back-pressure, so if mempool is slow the back-pressure will
    // propagate down to network.
    
    ///即时处理new peer和lost peer 
    let f_inbound_network_task = network_events
        .filter_map(move |network_event| {
            trace!("SharedMempoolEvent::NetworkEvent::{:?}", network_event);
            match network_event {
                Ok(network_event) => match network_event {
                    Event::NewPeer(peer_id) => {
                        OP_COUNTERS.inc("smp.event.new_peer");
                        new_peer(&peer_info, peer_id);
                        notify_subscribers(
                            SharedMempoolNotification::PeerStateChange,
                            &subscribers,
                        );
                        future::ready(None)
                    }
                    Event::LostPeer(peer_id) => {
                        OP_COUNTERS.inc("smp.event.lost_peer");
                        lost_peer(&peer_info, peer_id);
                        notify_subscribers(
                            SharedMempoolNotification::PeerStateChange,
                            &subscribers,
                        );
                        future::ready(None)
                    }
                    // Pass through messages to next combinator
                    Event::Message((peer_id, msg)) => future::ready(Some((peer_id, msg))),
                    _ => {
                        security_log(SecurityEvent::InvalidNetworkEventMP)
                            .error("UnexpectedNetworkEvent")
                            .data(&network_event)
                            .log();
                        unreachable!("Unexpected network event")
                    }
                },
                Err(e) => {
                    security_log(SecurityEvent::InvalidNetworkEventMP)
                        .error(&e)
                        .log();
                    future::ready(None)
                }
            }
        })
        // Run max_inbound_syncs number of `process_incoming_transactions` concurrently
    并发的处理peer发来的transactions
        .for_each_concurrent(
            max_inbound_syncs, /* limit */
            move |(peer_id, mut msg)| {
                OP_COUNTERS.inc("smp.event.message");
                let transactions: Vec<_> = msg
                    .take_transactions()
                    .into_iter()
                    .filter_map(|txn| match SignedTransaction::from_proto(txn) {
                        Ok(t) => Some(t),
                        Err(e) => {
                            security_log(SecurityEvent::InvalidTransactionMP)
                                .error(&e)
                                .data(&msg)
                                .log();
                            None
                        }
                    })
                    .collect();
                OP_COUNTERS.inc_by(
                    &format!("smp.transactions.received.{:?}", peer_id),
                    transactions.len(),
                );

                process_incoming_transactions(smp.clone(), peer_id, transactions)
            },
        );

    // drive the inbound futures to completion
    f_inbound_network_task.await;

    crit!("SharedMempool inbound_network_task terminated");
}
```

#### gc_task：定期进行GC

```rust
/// GC all expired transactions by SystemTTL
async fn gc_task(mempool: Arc<Mutex<CoreMempool>>, gc_interval_ms: u64) {
    let mut interval = Interval::new_interval(Duration::from_millis(gc_interval_ms)).compat();
    while let Some(res) = interval.next().await {
        match res {
            Ok(_) => {
                mempool
                    .lock()
                    .expect("[shared mempool] failed to acquire mempool lock")
                    .gc_by_system_ttl();
            }
            Err(e) => {
                error!("Error in gc_task timer interval: {:?}", e);
                break;
            }
        }
    }

    crit!("SharedMempool gc_task terminated");
}
```

#### start_shared_mempool:启动shared_mempool,核心就是启动了三个线程，来处理各种事件

```rust
/// bootstrap of SharedMempool
/// creates separate Tokio Runtime that runs following routines:
///   - outbound_sync_task (task that periodically broadcasts transactions to peers)
///   - inbound_network_task (task that handles inbound mempool messages and network events)
///   - gc_task (task that performs GC of all expired transactions by SystemTTL)
pub(crate) fn start_shared_mempool<V>(
    config: &NodeConfig,
    mempool: Arc<Mutex<CoreMempool>>,
    network_sender: MempoolNetworkSender,
    network_events: MempoolNetworkEvents,
    storage_read_client: Arc<dyn StorageRead>,
    validator: Arc<V>,
    subscribers: Vec<UnboundedSender<SharedMempoolNotification>>,
    timer: Option<IntervalStream>,
) -> Runtime
where
    V: TransactionValidation + 'static,
{
    let runtime = Builder::new()
        .name_prefix("shared-mem-")
        .build()
        .expect("[shared mempool] failed to create runtime");
    let executor = runtime.executor();

    let peer_info = Arc::new(Mutex::new(PeerInfo::new()));

    let smp = SharedMempool {
        mempool: mempool.clone(),
        config: config.mempool.clone(),
        network_sender,
        storage_read_client,
        validator,
        peer_info,
        subscribers,
    };

    let interval =
        timer.unwrap_or_else(|| default_timer(config.mempool.shared_mempool_tick_interval_ms));

    executor.spawn(
        outbound_sync_task(smp.clone(), interval)
            .boxed()
            .unit_error()
            .compat(),
    );

    executor.spawn(
        inbound_network_task(smp, network_events)
            .boxed()
            .unit_error()
            .compat(),
    );

    executor.spawn(
        gc_task(mempool, config.mempool.system_transaction_gc_interval_ms)
            .boxed()
            .unit_error()
            .compat(),
    );

    runtime
}
/// bootstrap of SharedMempool
/// creates separate Tokio Runtime that runs following routines:
///   - outbound_sync_task (task that periodically broadcasts transactions to peers)
///   - inbound_network_task (task that handles inbound mempool messages and network events)
///   - gc_task (task that performs GC of all expired transactions by SystemTTL)
pub(crate) fn start_shared_mempool<V>(
    config: &NodeConfig,
    mempool: Arc<Mutex<CoreMempool>>,
    network_sender: MempoolNetworkSender,
    network_events: MempoolNetworkEvents,
    storage_read_client: Arc<dyn StorageRead>,
    validator: Arc<V>,
    subscribers: Vec<UnboundedSender<SharedMempoolNotification>>,
    timer: Option<IntervalStream>,
) -> Runtime
where
    V: TransactionValidation + 'static,
{
    let runtime = Builder::new()
        .name_prefix("shared-mem-")
        .build()
        .expect("[shared mempool] failed to create runtime");
    let executor = runtime.executor();

    let peer_info = Arc::new(Mutex::new(PeerInfo::new()));

    let smp = SharedMempool {
        mempool: mempool.clone(),
        config: config.mempool.clone(),
        network_sender,
        storage_read_client,
        validator,
        peer_info,
        subscribers,
    };

    let interval =
        timer.unwrap_or_else(|| default_timer(config.mempool.shared_mempool_tick_interval_ms));

    executor.spawn(
        outbound_sync_task(smp.clone(), interval)
            .boxed()
            .unit_error()
            .compat(),
    );

    executor.spawn(
        inbound_network_task(smp, network_events)
            .boxed()
            .unit_error()
            .compat(),
    );

    executor.spawn(
        gc_task(mempool, config.mempool.system_transaction_gc_interval_ms)
            .boxed()
            .unit_error()
            .compat(),
    );

    runtime
}

```





