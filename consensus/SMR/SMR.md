# SMR

## Start：

  ```rust
//SMRstart的代码，总体来看他完成了一系列功能：开启了network，同步到root，创建了block_store，proposal_generator， safety_rules，pacemaker , event_processor。SMR是用一种 event based fashion的机制工作的，在这里面，eventProcess就负责处理event们， It is exposing the async processing functions for each event type.

impl<T: Payload, P: ProposerInfo> StateMachineReplication for ChainedBftSMR<T, P> {
    type Payload = T;

    fn start(
        &mut self,
        txn_manager: Arc<dyn TxnManager<Payload = Self::Payload>>,
        state_computer: Arc<dyn StateComputer<Payload = Self::Payload>>,
    ) -> Result<()> {
        let executor = self
            .runtime
            .as_mut()
            .expect("Consensus start: No valid runtime found!")
            .executor();
        let time_service = Arc::new(ClockTimeService::new(executor.clone()));

        // We first start the network and retrieve the network receivers (this function needs a
        // mutable reference).
        // Must do it here before giving the clones of network to other components.
        let network_receivers = self.network.start(&executor);
        let initial_data = self
            .initial_data
            .take()
            .expect("already started, initial data is None");
        let consensus_state = initial_data.state();
        let highest_timeout_certificates = initial_data.highest_timeout_certificates().clone();
        if initial_data.need_sync() {
            loop {
                // make sure we sync to the root state in case we're not
                let status = block_on(state_computer.sync_to(initial_data.root_ledger_info()));
                match status {
                    Ok(SyncStatus::Finished) => break,
                    Ok(SyncStatus::DownloadFailed) => {
                        warn!("DownloadFailed, we may not establish connection with peers yet, sleep and retry");
                        // we can remove this when we start to handle NewPeer/LostPeer events.
                        thread::sleep(Duration::from_secs(2));
                    }
                    Ok(e) => panic!(
                    "state synchronizer failure: {:?}, this validator will be killed as it can not \
                 recover from this error.  After the validator is restarted, synchronization will \
                 be retried.",
                    e
                ),
                    Err(e) => panic!(
                    "state synchronizer failure: {:?}, this validator will be killed as it can not \
                 recover from this error.  After the validator is restarted, synchronization will \
                 be retried.",
                    e
                ),
                }
            }
        }

        let block_store = Arc::new(block_on(BlockStore::new(
            Arc::clone(&self.storage),
            initial_data,
            self.signer.clone(),
            Arc::clone(&state_computer),
            true,
            self.config.max_pruned_blocks_in_mem,
        )));
        self.block_store = Some(Arc::clone(&block_store));

        // txn manager is required both by proposal generator (to pull the proposers)
        // and by event processor (to update their status).
        let proposal_generator = ProposalGenerator::new(
            block_store.clone(),
            Arc::clone(&txn_manager),
            time_service.clone(),
            self.config.max_block_size,
            true,
        );

        let safety_rules = Arc::new(RwLock::new(SafetyRules::new(
            block_store.clone(),
            consensus_state,
        )));

        let (external_timeout_sender, external_timeout_receiver) =
            channel::new(1_024, &counters::PENDING_PACEMAKER_TIMEOUTS);
        let (new_round_events_sender, new_round_events_receiver) =
            channel::new(1_024, &counters::PENDING_NEW_ROUND_EVENTS);
        let pacemaker = self.create_pacemaker(
            executor.clone(),
            self.storage.persistent_liveness_storage(),
            safety_rules.read().unwrap().last_committed_round(),
            block_store.highest_certified_block().round(),
            highest_timeout_certificates,
            time_service.clone(),
            new_round_events_sender,
            external_timeout_sender,
        );

        let (winning_proposals_sender, winning_proposals_receiver) =
            channel::new(1_024, &counters::PENDING_WINNING_PROPOSALS);
        let proposer_election = self.create_proposer_election(winning_proposals_sender);
        let event_processor = Arc::new(futures_locks::RwLock::new(EventProcessor::new(
            self.author,
            Arc::clone(&block_store),
            Arc::clone(&pacemaker),
            Arc::clone(&proposer_election),
            proposal_generator,
            safety_rules,
            state_computer,
            txn_manager,
            self.network.clone(),
            Arc::clone(&self.storage),
            time_service.clone(),
            true,
        )));

        self.start_event_processing(
            event_processor,
            executor.clone(),
            new_round_events_receiver,
            winning_proposals_receiver,
            network_receivers,
            external_timeout_receiver,
        );

        debug!("Chained BFT SMR started.");
        Ok(())
    }

  ```

## 几个模块：将分别介绍

- **BlockStore** maintains the tree of proposal blocks, block execution, votes, quorum certificates, and persistent storage. It is responsible for maintaining the consistency of the combination of these data structures and can be concurrently accessed by other subcomponents.
- **EventProcessor** is responsible for processing the individual events (e.g., process_new_round, process_proposal, process_vote). It exposes the async processing functions for each event type and drives the protocol 
- **Pacemaker** is responsible for the liveness of the consensus protocol. It changes rounds due to timeout certificates or quorum certificates and proposes blocks when it is the proposer for the current round.
- **SafetyRules** is responsible for the safety of the consensus protocol. It processes quorum certificates and LedgerInfo to learn about new commits and guarantees that the two voting rules are followed — even in the case of restart (since all safety data is persisted to local storage).



