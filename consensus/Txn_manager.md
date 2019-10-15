# Txn_Manager

## 功能：

是consensus模块与mempool模块的接口，主要用于

- 当自己是leader的时候pull出一个块来进行propose
- 有block被commit之后，将这个block中的txns都从mempool中删除

## 代码实现：

**代码逻辑非常简单，实现了上面所说的两个功能。**

```rust
/// Proxy interface to mempool
pub struct MempoolProxy {
    mempool: Arc<MempoolClient>,
}

impl MempoolProxy {
    pub fn new(mempool: Arc<MempoolClient>) -> Self {
        Self {
            mempool: Arc::clone(&mempool),
        }
    }

    /// Generate mempool commit transactions request given the set of txns and their status
  ////这里值得注意的是txn的状态有rejected和not rejected，mempool会根据标注的这个状态来确定txn的去留
    fn gen_commit_transactions_request(
        txns: &[SignedTransaction],
        compute_result: &StateComputeResult,
        timestamp_usecs: u64,
    ) -> CommitTransactionsRequest {
        let mut all_updates = Vec::new();
        assert_eq!(txns.len(), compute_result.compute_status.len());
        for (txn, status) in txns.iter().zip(compute_result.compute_status.iter()) {
            let mut transaction = CommittedTransaction::default();
            transaction.sender = txn.sender().as_ref().to_vec();
            transaction.sequence_number = txn.sequence_number();
            match status {
                TransactionStatus::Keep(_) => {
                    counters::SUCCESS_TXNS_COUNT.inc();
                    transaction.is_rejected = false;//标注状态
                }
                TransactionStatus::Discard(_) => {
                    counters::FAILED_TXNS_COUNT.inc();
                    transaction.is_rejected = true;
                }
            };
            all_updates.push(transaction);
        }
        let mut req = CommitTransactionsRequest::default();
        req.transactions = all_updates;
        req.block_timestamp_usecs = timestamp_usecs;
        req
    }

    /// Submit the request and return the future, which is fulfilled when the response is received.
    fn submit_commit_transactions_request(
        &self,
        req: CommitTransactionsRequest,
    ) -> Pin<Box<dyn Future<Output = Result<()>> + Send>> {
        match self.mempool.commit_transactions_async(&req) {
            Ok(receiver) => async move {
                match receiver.compat().await {
                    Ok(_) => Ok(()),
                    Err(e) => Err(e.into()),
                }
            }
                .boxed(),
            Err(e) => future::err(e.into()).boxed(),
        }
    }
}

impl TxnManager for MempoolProxy {
    type Payload = Vec<SignedTransaction>;

    /// The returned future is fulfilled with the vector of SignedTransactions
    fn pull_txns(
        &self,
        max_size: u64,
        exclude_payloads: Vec<&Self::Payload>,
    ) -> Pin<Box<dyn Future<Output = Result<Self::Payload>> + Send>> {
        let mut exclude_txns = vec![];
        for payload in exclude_payloads {
            for signed_txn in payload {
                let mut txn_meta = TransactionExclusion::default();
                txn_meta.sender = signed_txn.sender().into();
                txn_meta.sequence_number = signed_txn.sequence_number();
                exclude_txns.push(txn_meta);
            }
        }
        let mut get_block_request = GetBlockRequest::default();
        get_block_request.max_block_size = max_size;
        get_block_request.transactions = exclude_txns;
        match self.mempool.get_block_async(&get_block_request) {
            Ok(receiver) => async move {
                match receiver.compat().await {
                    Ok(response) => Ok(response
                        .block
                        .unwrap_or_else(Default::default)
                        .transactions
                        .into_iter()
                        .filter_map(|proto_txn| {
                            match SignedTransaction::try_from(proto_txn.clone()) {
                                Ok(t) => Some(t),
                                Err(e) => {
                                    security_log(SecurityEvent::InvalidTransactionConsensus)
                                        .error(&e)
                                        .data(&proto_txn)
                                        .log();
                                    None
                                }
                            }
                        })
                        .collect()),
                    Err(e) => Err(e.into()),
                }
            }
                .boxed(),
            Err(e) => future::err(e.into()).boxed(),
        }
    }

    fn commit_txns<'a>(
        &'a self,
        txns: &Self::Payload,
        compute_result: &StateComputeResult,
        // Monotonic timestamp_usecs of committed blocks is used to GC expired transactions.
        timestamp_usecs: u64,
    ) -> Pin<Box<dyn Future<Output = Result<()>> + Send + 'a>> {
        counters::COMMITTED_BLOCKS_COUNT.inc();
        counters::COMMITTED_TXNS_COUNT.inc_by(txns.len() as i64);
        counters::NUM_TXNS_PER_BLOCK.observe(txns.len() as f64);
        let req =
            Self::gen_commit_transactions_request(txns.as_slice(), compute_result, timestamp_usecs);
        self.submit_commit_transactions_request(req)
    }
}

```

## 总结：

**这一部分的逻辑比较简单，主要使用了与mempool通信的client，主要的功能就是向mempool发送请求，具体功能的完成是mempool收到请求之后做的**