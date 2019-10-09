### 这一部分有点偷懒，之后会补上



## execution_service

```rust
pub struct ExecutionService {
    /// `ExecutionService` simply contains an `Executor` and uses it to process requests. We wrap
    /// it in `Arc` because `ExecutionService` has to implement `Clone`.
    executor: Arc<Executor<MoveVM>>,
}

impl ExecutionService {
    /// Constructs an `ExecutionService`.
    pub fn new(
        storage_read_client: Arc<dyn StorageRead>,
        storage_write_client: Arc<dyn StorageWrite>,
        config: &NodeConfig,
    ) -> Self {
        let executor = Arc::new(Executor::new(
            storage_read_client,
            storage_write_client,
            config,
        ));
        ExecutionService { executor }
    }
}
```

#### 提供的几种命令的rpc：

 ```rust
  fn execute_block(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::ExecuteBlockRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::ExecuteBlockResponse>,
    ) {
        match ExecuteBlockRequest::from_proto(request) {
            Ok(req) => {
                let fut = process_response(
                    self.executor.execute_block(
                        req.transactions,
                        req.parent_block_id,
                        req.block_id,
                    ),
                    sink,
                )
                .boxed()
                .unit_error()
                .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }

    fn commit_block(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::CommitBlockRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::CommitBlockResponse>,
    ) {
        match CommitBlockRequest::from_proto(request) {
            Ok(req) => {
                let fut =
                    process_response(self.executor.commit_block(req.ledger_info_with_sigs), sink)
                        .boxed()
                        .unit_error()
                        .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }

    fn execute_chunk(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::ExecuteChunkRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::ExecuteChunkResponse>,
    ) {
        match ExecuteChunkRequest::from_proto(request) {
            Ok(req) => {
                let fut = process_response(
                    self.executor
                        .execute_chunk(req.txn_list_with_proof, req.ledger_info_with_sigs),
                    sink,
                )
                .boxed()
                .unit_error()
                .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }
}
  fn execute_block(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::ExecuteBlockRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::ExecuteBlockResponse>,
    ) {
        match ExecuteBlockRequest::from_proto(request) {
            Ok(req) => {
                let fut = process_response(
                    self.executor.execute_block(
                        req.transactions,
                        req.parent_block_id,
                        req.block_id,
                    ),
                    sink,
                )
                .boxed()
                .unit_error()
                .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }

    fn commit_block(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::CommitBlockRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::CommitBlockResponse>,
    ) {
        match CommitBlockRequest::from_proto(request) {
            Ok(req) => {
                let fut =
                    process_response(self.executor.commit_block(req.ledger_info_with_sigs), sink)
                        .boxed()
                        .unit_error()
                        .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }

    fn execute_chunk(
        &mut self,
        ctx: grpcio::RpcContext,
        request: execution_proto::proto::execution::ExecuteChunkRequest,
        sink: grpcio::UnarySink<execution_proto::proto::execution::ExecuteChunkResponse>,
    ) {
        match ExecuteChunkRequest::from_proto(request) {
            Ok(req) => {
                let fut = process_response(
                    self.executor
                        .execute_chunk(req.txn_list_with_proof, req.ledger_info_with_sigs),
                    sink,
                )
                .boxed()
                .unit_error()
                .compat();
                ctx.spawn(fut);
            }
            Err(err) => {
                let fut = process_conversion_error(err, sink);
                ctx.spawn(fut);
            }
        }
    }
}

 ```

 #### 处理回应的RPC：

```rust
async fn process_response<T>(
    resp: oneshot::Receiver<Result<T>>,
    sink: grpcio::UnarySink<<T as IntoProto>::ProtoType>,
) where
    T: IntoProto,
{
    match resp.await {
        Ok(Ok(response)) => {
            sink.success(response.into_proto());
        }
        Ok(Err(err)) => {
            set_failure_message(
                RpcStatusCode::Unknown,
                format!("Failed to process request: {}", err),
                sink,
            );
        }
        Err(oneshot::Canceled) => {
            set_failure_message(
                RpcStatusCode::Internal,
                "Executor Internal error: sender is dropped.".to_string(),
                sink,
            );
        }
    }
}

```



## client

```rust
pub struct ExecutionClient {
    client: execution_grpc::ExecutionClient,
}

impl ExecutionClient {
    pub fn new(env: Arc<Environment>, host: &str, port: u16) -> Self {
        let channel = ChannelBuilder::new(env).connect(&format!("{}:{}", host, port));
        let client = execution_grpc::ExecutionClient::new(channel);
        ExecutionClient { client }
    }

    pub fn execute_block(&self, request: ExecuteBlockRequest) -> Result<ExecuteBlockResponse> {
        let proto_request = request.into_proto();
        match self.client.execute_block(&proto_request) {
            Ok(proto_response) => Ok(ExecuteBlockResponse::from_proto(proto_response)?),
            Err(err) => bail!("GRPC error: {}", err),
        }
    }

    pub fn commit_block(&self, ledger_info_with_sigs: LedgerInfoWithSignatures) -> Result<()> {
        let proto_ledger_info = ledger_info_with_sigs.into_proto();
        let mut request = CommitBlockRequest::new();
        request.set_ledger_info_with_sigs(proto_ledger_info);
        match self.client.commit_block(&request) {
            Ok(_proto_response) => Ok(()),
            Err(err) => bail!("GRPC error: {}", err),
        }
    }
}

```

