# runtime

## 功能：

bundle of shared mempool and gRPC service

初始化了两部分，一个是grpc服务，一个是shared mempool

## 代码实现：

对各个部分进行了初始化，以及起了服务。这之后就mempool就可以正常运行了。

```rust
impl MempoolRuntime {
    /// setup Mempool runtime
    pub fn bootstrap(
        config: &NodeConfig,
        network_sender: MempoolNetworkSender,
        network_events: MempoolNetworkEvents,
    ) -> Self {
        let mempool = Arc::new(Mutex::new(CoreMempool::new(&config)));

        // setup grpc server
        let env = Arc::new(
            EnvBuilder::new()
                .name_prefix("grpc-mempool-")
                .cq_count(unsafe { max(grpcio_sys::gpr_cpu_num_cores() as usize / 2, 2) })
                .build(),
        );
        let handle = MempoolService {
            core_mempool: Arc::clone(&mempool),
        };
        let service = mempool_grpc::create_mempool(handle);
        let grpc_server = ::grpcio::ServerBuilder::new(env)
            .register_service(service)
            .bind(
                config.mempool.address.clone(),
                config.mempool.mempool_service_port,
            )/
            .build()
            .expect("[mempool] unable to create grpc server");

        // setup shared mempool
      ////下面是初始化shared mempool，先初始化了两个shared mempool需要的部分，即storage_client,还有vm_validator,之后起了shared_mempool
        let storage_client: Arc<dyn StorageRead> = Arc::new(StorageReadServiceClient::new(
            Arc::new(EnvBuilder::new().name_prefix("grpc-mem-sto-").build()),
            "localhost",
            config.storage.port,
        ));
        let vm_validator = Arc::new(VMValidator::new(&config, Arc::clone(&storage_client)));
        let shared_mempool = start_shared_mempool(
            config,
            mempool,
            network_sender,
            network_events,
            storage_client,
            vm_validator,
            vec![],
            None,
        );
        Self {
            grpc_server: ServerHandle::setup(grpc_server),
            shared_mempool,
        }
    }
}

```



