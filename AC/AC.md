# Admission Control

## 目录

```
    ├── README.md
    ├── admission_control_proto
    │   └── src
    │       └── proto                           # Protobuf definition files
    └── admission_control_service
        └── src                                 # gRPC service source files
            ├── admission_control_node.rs       # Wrapper to run AC in a separate thread
            ├── admission_control_service.rs    # gRPC service and main logic
            ├── main.rs                         # Main entry to run AC as a binary
            └── unit_tests                      # Testsv
```

## 功能：

* AC是validator与外界交互的唯一接口
* 在收到client发来的查询状态、同步到最新ledger等请求时，AC会将请求直接转发给validator的storage模块，而不是直接处理这些请求。
* 在收到submit transaction请求时，先验证这一请求的合法性（包括数字签名，账户余额等），如果不合法，返回错误(这些验证一部分是在AC进行，另一部分则通过VM进行）；如果合法，则将transaction提交到validator的mempool模块内存储到池中等待组合成区块

## 与其他模块的交互：

**Mempool ，Storage，VM_validator**

前两者是通过调用gRPC服务的形式，VM_validator 是通过直接生成一个validator实例来实现的



## 代码实现：

### admission_control_service.rs:

#### 定义：

```rust
//从这个定义可以看到ACservice实际上是通过与其他模块的交互实现的，其包括两个grpc client和一个VM_validator实例
pub struct AdmissionControlService<M, V> {
    /// gRPC client connecting Mempool.
    mempool_client: Arc<M>,
    /// gRPC client to send read requests to Storage.
    storage_read_client: Arc<dyn StorageRead>,
    /// VM validator instance to validate transactions sent from wallets.
    vm_validator: Arc<V>,
    /// Flag indicating whether we need to check mempool before validation, drop txn if check
    /// fails.
    need_to_check_mempool_before_validation: bool,
}
```

### 下面是对于这个struct的接口实现：

#### submit_transaction_inner：

```rust
 /// Validate transaction signature, then via VM, and add it to Mempool if it passes VM check.

pub(crate) fn submit_transaction_inner(
        &self,
        req: SubmitTransactionRequest,
    ) -> Result<SubmitTransactionResponse> {
        // Drop requests first if mempool is full (validator is lagging behind) so not to consume
        // unnecessary resources.
        if !self.can_send_txn_to_mempool()? {
            debug!("Mempool is full");
            OP_COUNTERS.inc_by("submit_txn.rejected.mempool_full", 1);
            let mut response = SubmitTransactionResponse::new();
            let mut status = MempoolAddTransactionStatus::new();
            status.set_code(MempoolIsFull);
            status.set_message("Mempool is full".to_string());
            response.set_mempool_status(status);
            return Ok(response);
        }

        let signed_txn_proto = req.get_signed_txn();
		//验证数字签名
        let signed_txn = match SignedTransaction::from_proto(signed_txn_proto.clone()) {
            Ok(t) => t,
            Err(e) => {
                security_log(SecurityEvent::InvalidTransactionAC)
                    .error(&e)
                    .data(&signed_txn_proto)
                    .log();
                let mut response = SubmitTransactionResponse::new();
                response.set_ac_status(
                    AdmissionControlStatus::Rejected("submit txn rejected".to_string())
                        .into_proto(),
                );
                OP_COUNTERS.inc_by("submit_txn.rejected.invalid_txn", 1);
                return Ok(response);
            }
        };

        
        let gas_cost = signed_txn.max_gas_amount();
        //////用self的VM_validator实例进行进一步验证，这里面的具体逻辑会在VM_validator中仔细说
        let validation_status = self
            .vm_validator
            .validate_transaction(signed_txn.clone())
            .wait()
            .map_err(|e| {
                security_log(SecurityEvent::InvalidTransactionAC)
                    .error(&e)
                    .data(&signed_txn)
                    .log();
                e
            })?;
        
        if let Some(validation_status) = validation_status {
            let mut response = SubmitTransactionResponse::new();
            OP_COUNTERS.inc_by("submit_txn.vm_validation.failure", 1);
            debug!(
                "txn failed in vm validation, status: {:?}, txn: {:?}",
                validation_status, signed_txn
            );
            response.set_vm_status(validation_status.into_proto());
            return Ok(response);
        }
        
        //验证成功
        let sender = signed_txn.sender();
        let account_state = block_on(get_account_state(self.storage_read_client.clone(), sender));/////通过self的storage grpc client来获取账户状态。
        let mut add_transaction_request = AddTransactionWithValidationRequest::new();
        add_transaction_request.signed_txn = req.signed_txn.clone();
        add_transaction_request.set_max_gas_cost(gas_cost);

        if let Ok((sequence_number, balance)) = account_state {
            add_transaction_request.set_account_balance(balance);
            add_transaction_request.set_latest_sequence_number(sequence_number);
        }
		
        self.add_txn_to_mempool(add_transaction_request)///加入到mempool
    }

   
```

#### update_to_latest_ledger_inner

```rust
fn update_to_latest_ledger_inner(
        &self,
        req: UpdateToLatestLedgerRequest,
    ) -> Result<UpdateToLatestLedgerResponse> {
        let rust_req = types::get_with_proof::UpdateToLatestLedgerRequest::from_proto(req)?;
         //问号运算符是rust的一个特性，类似错误检查的功能，如果这个部分运行错误，则整个函数直接返回ERR，没有错误就把返回值赋给rust_req
        ///这一部分AC部分不会有什么操作，只是单纯的当中介，转发请求和回答，具体实现是在storage和client两部分进行处理的
        let (response_items, ledger_info_with_sigs, validator_change_events) = self
            .storage_read_client
            .update_to_latest_ledger(rust_req.client_known_version, rust_req.requested_items)?;
        let rust_resp = types::get_with_proof::UpdateToLatestLedgerResponse::new(
            response_items,
            ledger_info_with_sigs,
            validator_change_events,
        );//proof
        Ok(rust_resp.into_proto())
    }
```

#### impl 一个接口：

#### submit_transaction & update_to_latest_ledger

```rust
//可以看出两个函数的实现都类似包裹了一层，都调用了上面的xxx_inner函数来实现实际的功能，之后再通过grpc转发给客户端。这两个函数负责AC对client请求的接受和结果的回复

impl<M: 'static, V> AdmissionControl for AdmissionControlService<M, V>
where
    M: MempoolClientTrait,
    V: TransactionValidation,
{
    /// Submit a transaction to the validator this AC instance connecting to.
    /// The specific transaction will be first validated by VM and then passed
    /// to Mempool for further processing.
    fn submit_transaction(
        &mut self,
        ctx: ::grpcio::RpcContext<'_>,
        req: SubmitTransactionRequest,
        sink: ::grpcio::UnarySink<SubmitTransactionResponse>,
    ) {
        debug!("[GRPC] AdmissionControl::submit_transaction");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.submit_transaction_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }

    /// This API is used to update the client to the latest ledger version and optionally also
    /// request 1..n other pieces of data.  This allows for batch queries.  All queries return
    /// proofs that a client should check to validate the data.
    /// Note that if a client only wishes to update to the latest LedgerInfo and receive the proof
    /// of this latest version, they can simply omit the requested_items (or pass an empty list).
    /// AC will not directly process this request but pass it to Storage instead.
    fn update_to_latest_ledger(
        &mut self,
        ctx: grpcio::RpcContext<'_>,
        req: types::proto::get_with_proof::UpdateToLatestLedgerRequest,
        sink: grpcio::UnarySink<types::proto::get_with_proof::UpdateToLatestLedgerResponse>,
    ) {
        debug!("[GRPC] AdmissionControl::update_to_latest_ledger");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.update_to_latest_ledger_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }
}

```

## 总结：

**看完AC部分的基本代码后，我们已经知道了client submit的transaction是怎样通过验证并加入到validator的mempool中等待组装成块的，也知道了client的query操作是如何与validator的storage建立关系的**



