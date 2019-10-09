# storage_service



```rust
/// Starts storage service according to config.
pub fn start_storage_service(config: &NodeConfig) -> ServerHandle {
    let (storage_service, shutdown_receiver) = StorageService::new(&config.storage.get_dir());
    spawn_service_thread_with_drop_closure(
        create_storage(storage_service),
        config.storage.address.clone(),
        config.storage.port,
        "storage",
        move || {
            shutdown_receiver
                .recv()
                .expect("Failed to receive on shutdown channel when storage service was dropped")
        },
    )
}


impl StorageService {
    /// This opens a [`LibraDB`] at `path` and returns a [`StorageService`] instance serving it.
    ///
    /// A receiver side of a channel is also returned through which one can receive a notice after
    /// all resources used by the service including the underlying [`LibraDB`] instance are
    /// fully dropped.
    ///
    /// example:
    /// ```no_run,
    ///    # use storage_service::*;
    ///    # use std::path::Path;
    ///    let (service, shutdown_receiver) = StorageService::new(&Path::new("path/to/db"));
    ///
    ///    drop(service);
    ///    shutdown_receiver.recv().expect("recv() should succeed.");
    ///
    ///    // LibraDB instance is guaranteed to be properly dropped at this point.
    /// ```
    pub fn new<P: AsRef<Path>>(path: &P) -> (Self, mpsc::Receiver<()>) {
        let (db_wrapper, shutdown_receiver) = LibraDBWrapper::new(path);
        (
            Self {
                db: Arc::new(db_wrapper),
            },
            shutdown_receiver,
        )
    }
}


//////StorageService提供了这些功能
impl StorageService {
    fn update_to_latest_ledger_inner(
        &self,
        req: UpdateToLatestLedgerRequest,
    ) -> Result<UpdateToLatestLedgerResponse> {
        let rust_req = types::get_with_proof::UpdateToLatestLedgerRequest::from_proto(req)?;

        let (response_items, ledger_info_with_sigs, validator_change_events) = self
            .db
            .update_to_latest_ledger(rust_req.client_known_version, rust_req.requested_items)?;

        let rust_resp = types::get_with_proof::UpdateToLatestLedgerResponse {
            response_items,
            ledger_info_with_sigs,
            validator_change_events,
        };

        Ok(rust_resp.into_proto())
    }

    fn get_transactions_inner(
        &self,
        req: GetTransactionsRequest,
    ) -> Result<GetTransactionsResponse> {
        let rust_req = storage_proto::GetTransactionsRequest::from_proto(req)?;

        let txn_list_with_proof = self.db.get_transactions(
            rust_req.start_version,
            rust_req.batch_size,
            rust_req.ledger_version,
            rust_req.fetch_events,
        )?;

        let rust_resp = storage_proto::GetTransactionsResponse::new(txn_list_with_proof);

        Ok(rust_resp.into_proto())
    }

    fn get_account_state_with_proof_by_state_root_inner(
        &self,
        req: GetAccountStateWithProofByStateRootRequest,
    ) -> Result<GetAccountStateWithProofByStateRootResponse> {
        let rust_req = storage_proto::GetAccountStateWithProofByStateRootRequest::from_proto(req)?;

        let (account_state_blob, sparse_merkle_proof) =
            self.db.get_account_state_with_proof_by_state_root(
                rust_req.address,
                rust_req.state_root_hash,
            )?;

        let rust_resp = storage_proto::GetAccountStateWithProofByStateRootResponse {
            account_state_blob,
            sparse_merkle_proof,
        };

        Ok(rust_resp.into_proto())
    }

    fn save_transactions_inner(
        &self,
        req: SaveTransactionsRequest,
    ) -> Result<SaveTransactionsResponse> {
        let rust_req = storage_proto::SaveTransactionsRequest::from_proto(req)?;
        self.db.save_transactions(
            &rust_req.txns_to_commit,
            rust_req.first_version,
            &rust_req.ledger_info_with_signatures,
        )?;
        Ok(SaveTransactionsResponse::new())
    }

    fn get_executor_startup_info_inner(&self) -> Result<GetExecutorStartupInfoResponse> {
        let info = self.db.get_executor_startup_info()?;
        let rust_resp = storage_proto::GetExecutorStartupInfoResponse { info };
        Ok(rust_resp.into_proto())
    }
}


////////还是直接调用inner，然后把结果作为grpc的结果返回
impl Storage for StorageService {
    fn update_to_latest_ledger(
        &mut self,
        ctx: grpcio::RpcContext<'_>,
        req: UpdateToLatestLedgerRequest,
        sink: grpcio::UnarySink<UpdateToLatestLedgerResponse>,
    ) {
        debug!("[GRPC] Storage::update_to_latest_ledger");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.update_to_latest_ledger_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }

    fn get_transactions(
        &mut self,
        ctx: grpcio::RpcContext,
        req: GetTransactionsRequest,
        sink: grpcio::UnarySink<GetTransactionsResponse>,
    ) {
        debug!("[GRPC] Storage::get_transactions");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.get_transactions_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }

    fn get_account_state_with_proof_by_state_root(
        &mut self,
        ctx: grpcio::RpcContext,
        req: GetAccountStateWithProofByStateRootRequest,
        sink: grpcio::UnarySink<GetAccountStateWithProofByStateRootResponse>,
    ) {
        debug!("[GRPC] Storage::get_account_state_with_proof_by_state_root");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.get_account_state_with_proof_by_state_root_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }

    fn save_transactions(
        &mut self,
        ctx: grpcio::RpcContext,
        req: SaveTransactionsRequest,
        sink: grpcio::UnarySink<SaveTransactionsResponse>,
    ) {
        debug!("[GRPC] Storage::save_transactions");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.save_transactions_inner(req);
        provide_grpc_response(resp, ctx, sink);
    }

    fn get_executor_startup_info(
        &mut self,
        ctx: grpcio::RpcContext,
        _req: GetExecutorStartupInfoRequest,
        sink: grpcio::UnarySink<GetExecutorStartupInfoResponse>,
    ) {
        debug!("[GRPC] Storage::get_executor_startup_info");
        let _timer = SVC_COUNTERS.req(&ctx);
        let resp = self.get_executor_startup_info_inner();
        provide_grpc_response(resp, ctx, sink);
    }
}



///////////////////main中的实现，包装了一个节点，然后运行节点，开启这些服务
impl StorageNode {
    pub fn new(node_config: NodeConfig) -> Self {
        StorageNode { node_config }
    }

    pub fn run(&self) -> Result<()> {
        info!("Starting storage node");

        let _handle = storage_service::start_storage_service(&self.node_config);

        // Start Debug interface
        let debug_service =
            node_debug_interface_grpc::create_node_debug_interface(NodeDebugService::new());
        let _debug_handle = spawn_service_thread(
            debug_service,
            self.node_config.storage.address.clone(),
            self.node_config.debug_interface.storage_node_debug_port,
            "debug_service",
        );

        info!("Started Storage Service");
        loop {
            thread::park();
        }
    }
}

fn main() {
    let (config, _logger, _args) = setup_executable(
        "Libra Storage node".to_string(),
        vec![ARG_PEER_ID, ARG_CONFIG_PATH, ARG_DISABLE_LOGGING],
    );

    let storage_node = StorageNode::new(config);

    storage_node.run().expect("Unable to run storage node");
}

```

# client：

#### read_interfaces:

```rust
/// This provides storage read interfaces backed by real storage service.
#[derive(Clone)]
pub struct StorageReadServiceClient {
    client: storage_grpc::StorageClient,
}

impl StorageReadServiceClient {
    /// Constructs a `StorageReadServiceClient` with given host and port.
    pub fn new(env: Arc<Environment>, host: &str, port: u16) -> Self {
        let channel = ChannelBuilder::new(env).connect(&format!("{}:{}", host, port));
        let client = storage_grpc::StorageClient::new(channel);
        StorageReadServiceClient { client }
    }
}

impl StorageRead for StorageReadServiceClient {
    fn update_to_latest_ledger(
        &self,
        client_known_version: Version,
        requested_items: Vec<RequestItem>,
    ) -> Result<(
        Vec<ResponseItem>,
        LedgerInfoWithSignatures,
        Vec<ValidatorChangeEventWithProof>,
    )> {
        block_on(self.update_to_latest_ledger_async(client_known_version, requested_items))
    }///可以看到这里调用了加async后缀的函数

    
    fn update_to_latest_ledger_async(
        &self,
        client_known_version: Version,
        requested_items: Vec<RequestItem>,
    ) -> Pin<
        Box<
            dyn Future<
                    Output = Result<(
                        Vec<ResponseItem>,
                        LedgerInfoWithSignatures,
                        Vec<ValidatorChangeEventWithProof>,
                    )>,
                > + Send,
        >,
    > {
        let req = UpdateToLatestLedgerRequest {
            client_known_version,
            requested_items,
        };
        convert_grpc_response(self.client.update_to_latest_ledger_async(&req.into_proto()))
            .map(|resp| {
                let rust_resp = UpdateToLatestLedgerResponse::from_proto(resp?)?;
                Ok((
                    rust_resp.response_items,
                    rust_resp.ledger_info_with_sigs,
                    rust_resp.validator_change_events,
                ))
            })
            .boxed()
    }

    fn get_transactions(
        &self,
        start_version: Version,
        batch_size: u64,
        ledger_version: Version,
        fetch_events: bool,
    ) -> Result<TransactionListWithProof> {
        block_on(self.get_transactions_async(
            start_version,
            batch_size,
            ledger_version,
            fetch_events,
        ))
    }

    fn get_transactions_async(
        &self,
        start_version: Version,
        batch_size: u64,
        ledger_version: Version,
        fetch_events: bool,
    ) -> Pin<Box<dyn Future<Output = Result<TransactionListWithProof>> + Send>> {
        let req =
            GetTransactionsRequest::new(start_version, batch_size, ledger_version, fetch_events);
        convert_grpc_response(self.client.get_transactions_async(&req.into_proto()))
            .map(|resp| {
                let rust_resp = GetTransactionsResponse::from_proto(resp?)?;
                Ok(rust_resp.txn_list_with_proof)
            })
            .boxed()
    }

    fn get_account_state_with_proof_by_state_root(
        &self,
        address: AccountAddress,
        state_root_hash: HashValue,
    ) -> Result<(Option<AccountStateBlob>, SparseMerkleProof)> {
        block_on(self.get_account_state_with_proof_by_state_root_async(address, state_root_hash))
    }

    fn get_account_state_with_proof_by_state_root_async(
        &self,
        address: AccountAddress,
        state_root_hash: HashValue,
    ) -> Pin<Box<dyn Future<Output = Result<(Option<AccountStateBlob>, SparseMerkleProof)>> + Send>>
    {
        let req = GetAccountStateWithProofByStateRootRequest::new(address, state_root_hash);
        convert_grpc_response(
            self.client
                .get_account_state_with_proof_by_state_root_async(&req.into_proto()),
        )
        .map(|resp| {
            let resp = GetAccountStateWithProofByStateRootResponse::from_proto(resp?)?;
            Ok(resp.into())
        })
        .boxed()
    }

    fn get_executor_startup_info(&self) -> Result<Option<ExecutorStartupInfo>> {
        block_on(self.get_executor_startup_info_async())
    }

    fn get_executor_startup_info_async(
        &self,
    ) -> Pin<Box<dyn Future<Output = Result<Option<ExecutorStartupInfo>>> + Send>> {
        let proto_req = GetExecutorStartupInfoRequest::new();
        convert_grpc_response(self.client.get_executor_startup_info_async(&proto_req))
            .map(|resp| {
                let resp = GetExecutorStartupInfoResponse::from_proto(resp?)?;
                Ok(resp.info)
            })
            .boxed()
    }
}

```

剩下的其实都差不多，还有write啊各种，可以看storage_client客户端，以后再补充orz

