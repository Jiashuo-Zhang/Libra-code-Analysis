# EventProcessor:

## 功能：

**EventProcessoris responsible for processing the individual events (e.g., process_new_round, process_proposal, process_vote). It exposes the async processing functions for each event type and drives the protocol** 

**并且EventProcessor是SMR里面比较核心的部分，他实现了很全面的各种事件的处理，正如之前提到的，SMR本身就是事件驱动的，EventProcessor把各个部分结合起来，为每一个事件调度各个部分，最终实现SMR核心功能 。下面我们会细致分析这里面实现的所有事件的函数以理解其功能**

## 代码：

```rust
/// Consensus SMR is working in an event based fashion: EventProcessor is responsible for
/// processing the individual events (e.g., process_new_round, process_proposal, process_vote,
/// etc.). It is exposing the async processing functions for each event type.
/// The caller is responsible for running the event loops and driving the execution via some
/// executors.
//可以看到他包含了各种的处理函数，进行调度来响应事件
pub struct EventProcessor<T, P> {
    author: P,
    block_store: Arc<BlockStore<T>>,
    pacemaker: Arc<dyn Pacemaker>,
    proposer_election: Arc<dyn ProposerElection<T, P> + Send + Sync>,
    proposal_generator: ProposalGenerator<T>,
    safety_rules: Arc<RwLock<SafetyRules<T>>>,
    state_computer: Arc<dyn StateComputer<Payload = T>>,
    txn_manager: Arc<dyn TxnManager<Payload = T>>,
    network: ConsensusNetworkImpl,
    storage: Arc<dyn PersistentStorage<T>>,
    sync_manager: SyncManager<T>,
    time_service: Arc<dyn TimeService>,
    enforce_increasing_timestamps: bool,
}
 pub fn new(
        author: P,
        block_store: Arc<BlockStore<T>>,
        pacemaker: Arc<dyn Pacemaker>,
        proposer_election: Arc<dyn ProposerElection<T, P> + Send + Sync>,
        proposal_generator: ProposalGenerator<T>,
        safety_rules: Arc<RwLock<SafetyRules<T>>>,
        state_computer: Arc<dyn StateComputer<Payload = T>>,
        txn_manager: Arc<dyn TxnManager<Payload = T>>,
        network: ConsensusNetworkImpl,
        storage: Arc<dyn PersistentStorage<T>>,
        time_service: Arc<dyn TimeService>,
        enforce_increasing_timestamps: bool,
    ) -> Self {
        let sync_manager = SyncManager::new(
            Arc::clone(&block_store),
            Arc::clone(&storage),
            network.clone(),
            Arc::clone(&state_computer),
        );
```

#### process_new_round_event:“新一轮开始”事件处理

```rust
 /// Leader:
    ///
///在上一round得到了QC之后或者超时了之后，就会触发这一事件
    /// This event is triggered by a new quorum certificate at the previous round or a
    /// timeout certificate at the previous round.  In either case, if this replica is the new
    /// proposer for this round, it is ready to propose and guarantee that it can create a proposal
    /// that all honest replicas can vote for.  While this method should only be invoked at most
    /// once per round, we ensure that only at most one proposal can get generated per round to
    /// avoid accidental equivocation of proposals.
    ///
    /// Replica:
    ///
    /// Do nothing
    pub async fn process_new_round_event(&self, new_round_event: NewRoundEvent) {
        debug!("Processing {}", new_round_event);
        counters::CURRENT_ROUND.set(new_round_event.round as i64);
        counters::ROUND_TIMEOUT_MS.set(new_round_event.timeout.as_millis() as i64);
        match new_round_event.reason {
            NewRoundReason::QCReady => {
                counters::QC_ROUNDS_COUNT.inc();//上个QC Record 顺利生成
            }
            NewRoundReason::Timeout { .. } => {//上个QC超时了（可能是网络原因，或者没有收集到2f+1的签名等）
                counters::TIMEOUT_ROUNDS_COUNT.inc();
            }
        };
        let proposer_info = match self
            .proposer_election
            .is_valid_proposer(self.author, new_round_event.round)
        {
            Some(pi) => pi,
            None => {
                return; //self不是合法的proposer，直接return
            }
        };
       
//////self是合法的proposer，需要来产生一个proposal
        // Proposal generator will ensure that at most one proposal is generated per round
       //产生一个proposal来让大家vote for
        let proposal = match self
            .proposal_generator
            .generate_proposal(
                new_round_event.round,
                self.pacemaker.current_round_deadline(),
            )
            .await
        {
            Err(e) => {
                error!("Error while generating proposal: {:?}", e);
                return;
            }
            Ok(proposal) => proposal,
        };
        
        
        let mut network = self.network.clone();
        debug!("Propose {}", proposal);
        let timeout_certificate = match new_round_event.reason {
            NewRoundReason::Timeout { cert } => Some(cert),
            _ => None,
        };
        let highest_ledger_info = (*self.block_store.highest_ledger_info()).clone();
        network
            .broadcast_proposal(ProposalInfo {//广播出去组装好的proposal
                proposal,
                proposer_info,
                timeout_certificate,
                highest_ledger_info,
            })
            .await;
        counters::PROPOSALS_COUNT.inc();
    }

```

####process_proposal:“收到proposal”事件处理

```rust
/// The function is responsible for processing the incoming proposals and the Quorum
    /// Certificate. 1. commit to the committed state the new QC carries
    /// 2. fetch all the blocks from the committed state to the QC
    /// 3. forwarding the proposals to the ProposerElection queue,
    /// which is going to eventually trigger one winning proposal per round
    /// (to be processed via a separate function).
    /// The reason for separating `process_proposal` from `process_winning_proposal` is to
    /// (a) asynchronously prefetch dependencies and
    /// (b) allow the proposer election to choose one proposal out of many.
    pub async fn process_proposal(
        &self,
        proposal: ProposalInfo<T, P>,
    ) -> ProcessProposalResult<T, P> {
        debug!("Receive proposal {}", proposal);
        // Pacemaker is going to be updated with all the proposal certificates later,
        // but it's known that the pacemaker's round is not going to decrease so we can already
        // filter out the proposals from old rounds.
        let current_round = self.pacemaker.current_round();
        if proposal.proposal.round() < self.pacemaker.current_round() {
            warn!(
                "Proposal {} is ignored because its round {} != current round {}",
                proposal,
                proposal.proposal.round(),
                current_round
            );
            return ProcessProposalResult::Done;
        }
        if self
            .proposer_election
            .is_valid_proposer(proposal.proposer_info, proposal.proposal.round())
            .is_none()
        {
            warn!(
                "Proposer {} for block {} is not a valid proposer for this round",
                proposal.proposal.author(),
                proposal.proposal
            );
            return ProcessProposalResult::Done;
        }
//经过了几个简单的validate，验证round和proposer合法
        let deadline = self.pacemaker.current_round_deadline();
        if let Some(committed_block_id) = proposal.highest_ledger_info.committed_block_id() {
            if self
                .block_store
                .need_sync_for_quorum_cert(committed_block_id, &proposal.highest_ledger_info)
            {
                return ProcessProposalResult::NeedSync(deadline, proposal);//需要同步，
            }
        } else {
            warn!("Highest ledger info {} has no committed block", proposal);
            return ProcessProposalResult::Done;
        }

        match self
            .block_store
            .need_fetch_for_quorum_cert(proposal.proposal.quorum_cert())
        {
            NeedFetchResult::NeedFetch => {
                return ProcessProposalResult::NeedFetch(deadline, proposal)
            }
            NeedFetchResult::QCRoundBeforeRoot => {
                warn!("Proposal {} has a highest quorum certificate with round older than root round {}", proposal, self.block_store.root().round());
                return ProcessProposalResult::Done;
            }
            NeedFetchResult::QCBlockExist => {
                if let Err(e) = self
                    .block_store
                    .insert_single_quorum_cert(proposal.proposal.quorum_cert().clone())
                    .await
                {
                    warn!(
                        "Quorum certificate for proposal {} could not be inserted to the block store: {:?}",
                        proposal, e
                    );
                    return ProcessProposalResult::Done;
                }
            }
            NeedFetchResult::QCAlreadyExist => (),
        }

        self.finish_proposal_processing(proposal).await;
        ProcessProposalResult::Done
    }

```

### finish_proposal_processing和其他两个用来处理proposal的函数:

```rust
/// Finish proposal processing: note that multiple tasks can execute this function in parallel
    /// so be careful with the updates. The safest thing to do is to pass the proposal further
    /// to the proposal election.
    /// This function is invoked when all the dependencies for the given proposal are ready.

//注意有很多地方都会invoke这个函数，只要是满足了dependencies就可以invoke。比如下面的sync_and_process_proposal和fetch_and_process_proposal豆调用了这个函数
//这个函数做的就是  forwarding the proposals to the ProposerElection queue,（copy自上面的函数的第三步
    async fn finish_proposal_processing(&self, proposal: ProposalInfo<T, P>) {
        let qc = proposal.proposal.quorum_cert();
        self.pacemaker
            .process_certificates(
                qc.certified_block_round(),
                proposal.timeout_certificate.as_ref(),
            )
            .await;

        let current_round = self.pacemaker.current_round();
        if self.pacemaker.current_round() != proposal.proposal.round() {
            warn!(
                "Proposal {} is ignored because its round {} != current round {}",
                proposal,
                proposal.proposal.round(),
                current_round
            );
            return;
        }

        self.proposer_election.process_proposal(proposal).await;
    }

    /// Fetches and completes processing proposal in dedicated task
    pub async fn fetch_and_process_proposal(
        &self,
        deadline: Instant,
        proposal: ProposalInfo<T, P>,
    ) {
        if let Err(e) = self
            .sync_manager
            .fetch_quorum_cert(
                proposal.proposal.quorum_cert().clone(),
                proposal.proposer_info.get_author(),
                deadline,
            )////同步赶上到QC
            .await
        {
            warn!(
                "Quorum certificate for proposal {} could not be added to the block store: {:?}",
                proposal, e
            );
            return;
        }
        self.finish_proposal_processing(proposal).await;//完成了所有的dependencie，可以调用这个
    }


 /// Takes mutable reference to avoid race with other processing and perform state
    /// synchronization, then completes processing proposal in dedicated task
    pub async fn sync_and_process_proposal(
        &mut self,
        deadline: Instant,
        proposal: ProposalInfo<T, P>,
    ) {
        // check if we still need sync
        if let Err(e) = self
            .sync_manager
            .sync_to(
                deadline,
                SyncInfo {
                    highest_ledger_info: proposal.highest_ledger_info.clone(),
                    highest_quorum_cert: proposal.proposal.quorum_cert().clone(),
                    peer: proposal.proposer_info.get_author(),
                },
            )
            .await
        {
            warn!(
                "Quorum certificate for proposal {} could not be added to the block store: {:?}",
                proposal, e
            );
            return;
        }
        self.finish_proposal_processing(proposal).await;
    }

```

#### process_timeout_msg:

```rust
 /// Upon receiving TimeoutMsg, ensure that any branches with higher quorum certificates are
    /// populated to this replica prior to processing the pacemaker timeout.  This ensures that when
    /// a pacemaker timeout certificate is formed with 2f+1 timeouts, the next proposer will be
    /// able to chain a proposal block to a highest quorum certificate such that all honest replicas
    /// can vote for it.
    pub async fn process_timeout_msg(&mut self, timeout_msg: TimeoutMsg) {
        debug!(
            "Received a new round msg for round {} from {}",
            timeout_msg.pacemaker_timeout().round(),
            timeout_msg.author().short_str()
        );
        let current_highest_quorum_cert_round = self
            .block_store
            .highest_quorum_cert()
            .certified_block_round();
        let new_round_highest_quorum_cert_round = timeout_msg
            .highest_quorum_certificate()
            .certified_block_round();

        if current_highest_quorum_cert_round < new_round_highest_quorum_cert_round {
            // The timeout message carries a QC higher than what this node has seen before:
            // run state synchronization.
            //收到同步消息，发现这个消息说明有一个比当前自己状态更新的QC，这时候需要主动同步到这个状态
            let deadline = self.pacemaker.current_round_deadline();
            match self
                .sync_manager
                .sync_to(
                    deadline,
                    SyncInfo {
                        highest_ledger_info: timeout_msg.highest_ledger_info().clone(),
                        highest_quorum_cert: timeout_msg.highest_quorum_certificate().clone(),
                        peer: timeout_msg.author(),
                    },
                )
                .await
                {
                    Ok(()) => debug!(
                        "Successfully added new highest quorum certificate at round {} from old round {}",
                        new_round_highest_quorum_cert_round, current_highest_quorum_cert_round
                    ),
                    Err(e) => warn!(
                        "Unable to insert new highest quorum certificate {} from old round {} due to {:?}",
                        timeout_msg.highest_quorum_certificate(),
                        current_highest_quorum_cert_round,
                        e
                    ),
                }
        }
        self.pacemaker
            .process_remote_timeout(timeout_msg.pacemaker_timeout().clone())
            .await;
    }

//这个是说的收到了这个round超时的消息，这样的话就需要停止投票，保存当前的同步状态。
/// The replica stops voting for this round and saves its consensus state.  Voting is halted
    /// to ensure that the next proposer can make a proposal that can be voted on by all replicas.
    /// Saving the consensus state ensures that on restart, the replicas will not waste time
    /// on previous rounds.
    pub async fn process_outgoing_pacemaker_timeout(&self, round: Round) -> Option<TimeoutMsg> {
        // Stop voting at this round, persist the consensus state to support restarting from
        // a recent round (i.e. > the last vote round)  and then send the highest quorum
        // certificate known
        let consensus_state = self
            .safety_rules
            .write()
            .unwrap()
            .increase_last_vote_round(round);
        if let Some(consensus_state) = consensus_state {
            if let Err(e) = self.storage.save_consensus_state(consensus_state) {
                error!("Failed to persist consensus state after increasing the last vote round due to {:?}", e);
                return None;
            }
        }

        let last_vote_round = self
            .safety_rules
            .read()
            .unwrap()
            .consensus_state()
            .last_vote_round();
        warn!(
            "Round {} timed out and {}, expected round proposer was {:?}, broadcasting new round to all replicas",
            round,
            if last_vote_round == round { "already executed and voted at this round" } else { "will never vote at this round" },
            self.proposer_election.get_valid_proposers(round),
        );

        Some(TimeoutMsg::new(
            self.block_store.highest_quorum_cert().as_ref().clone(),
            self.block_store.highest_ledger_info().as_ref().clone(),
            PacemakerTimeout::new(round, self.block_store.signer()),
            self.block_store.signer(),
        ))
    }

```

#### process_winning_proposal： 从一系列proposal中选出了一个胜出的proposal，如果我们要投票就给他投票。这似乎说了一个round会有多个proposal，我们都会记录下来，并选出一个胜出的把它加入到block store之后尝试给他投票

```rust
 /// This function processes a proposal that was chosen as a representative of its round:
    /// 1. Add it to a block store.
    /// 2. Try to vote for it following the safety rules.
    /// 3. In case a validator chooses to vote, send the vote to the representatives at the next
    /// position.
    pub async fn process_winning_proposal(&self, proposal: ProposalInfo<T, P>) {
        let qc = proposal.proposal.quorum_cert();
        let update_res = self.safety_rules.write().unwrap().update(qc);
        if let Some(new_commit) = update_res {
            let finality_proof = qc.ledger_info().clone();
            self.process_commit(new_commit, finality_proof).await;
        }

        if let Some(time_to_receival) = duration_since_epoch()
            .checked_sub(Duration::from_micros(proposal.proposal.timestamp_usecs()))
        {
            counters::CREATION_TO_RECEIVAL_MS.observe(time_to_receival.as_millis() as f64);
        }
        let block = match self
            .sync_manager
            .execute_and_insert_block(proposal.proposal)
            .await
        {
            Err(e) => {
                debug!(
                    "Block proposal could not be added to the block store: {:?}",
                    e
                );
                return;
            }
            Ok(block) => block,
        };

        // Checking pacemaker round again, because multiple proposal can now race
        // during async block retrieval
        if self.pacemaker.current_round() != block.round() {
            debug!(
                "Skip voting for winning proposal {} rejected because round is incorrect. Pacemaker: {}, proposal: {}",
                block,
                self.pacemaker.current_round(),
                block.round()
            );
            return;
        }

        let current_round_deadline = self.pacemaker.current_round_deadline();
        if self.enforce_increasing_timestamps {
            match wait_if_possible(
                self.time_service.as_ref(),
                Duration::from_micros(block.timestamp_usecs()),
                current_round_deadline,
            )
            .await
            {
                Ok(waiting_success) => {
                    debug!("Success with {:?} for being able to vote", waiting_success);

                    match waiting_success {
                        WaitingSuccess::WaitWasRequired { wait_duration, .. } => {
                            counters::VOTE_SUCCESS_WAIT_MS
                                .observe(wait_duration.as_millis() as f64);
                            counters::VOTE_WAIT_WAS_REQUIRED_COUNT.inc();
                        }
                        WaitingSuccess::NoWaitRequired { .. } => {
                            counters::VOTE_SUCCESS_WAIT_MS.observe(0.0);
                            counters::VOTE_NO_WAIT_REQUIRED_COUNT.inc();
                        }
                    }
                }
                Err(waiting_error) => {
                    match waiting_error {
                        WaitingError::MaxWaitExceeded => {
                            error!(
                                "Waiting until proposal block timestamp usecs {:?} would exceed the round duration {:?}, hence will not vote for this round",
                                block.timestamp_usecs(),
                                current_round_deadline);
                            counters::VOTE_FAILURE_WAIT_MS.observe(0.0);
                            counters::VOTE_MAX_WAIT_EXCEEDED_COUNT.inc();
                            return;
                        }
                        WaitingError::WaitFailed {
                            current_duration_since_epoch,
                            wait_duration,
                        } => {
                            error!(
                                "Even after waiting for {:?}, proposal block timestamp usecs {:?} >= current timestamp usecs {:?}, will not vote for this round",
                                wait_duration,
                                block.timestamp_usecs(),
                                current_duration_since_epoch);
                            counters::VOTE_FAILURE_WAIT_MS
                                .observe(wait_duration.as_millis() as f64);
                            counters::VOTE_WAIT_FAILED_COUNT.inc();
                            return;
                        }
                    };
                }
            }
        }

        let vote_info = match self
            .safety_rules
            .write()
            .unwrap()
            .voting_rule(Arc::clone(&block))
        {
            Err(e) => {
                debug!("{}Rejected{} {}: {:?}", Fg(Red), Fg(Reset), block, e);
                return;
            }
            Ok(vote_info) => vote_info,
        };
        if let Err(e) = self
            .storage
            .save_consensus_state(vote_info.consensus_state().clone())
        {
            debug!("Fail to persist consensus state: {:?}", e);
            return;
        }
        let proposal_id = vote_info.proposal_id();
        let executed_state = self
            .block_store
            .get_state_for_block(proposal_id)
            .expect("Block proposal: no execution state found for inserted block.");

        let ledger_info_placeholder = self
            .block_store
            .ledger_info_placeholder(vote_info.potential_commit_id());
        let vote_msg = VoteMsg::new(
            proposal_id,
            executed_state,
            block.round(),
            self.author.get_author(),
            ledger_info_placeholder,
            self.block_store.signer(),
        );

        let recipients: Vec<Author> = self
            .proposer_election
            .get_valid_proposers(block.round() + 1)
            .iter()
            .map(ProposerInfo::get_author)
            .collect();
        debug!(
            "{}Voted for{} {}, potential commit {}",
            Fg(Green),
            Fg(Reset),
            block,
            vote_info
                .potential_commit_id()
                .unwrap_or_else(HashValue::zero)
        );
        self.network.send_vote(vote_msg, recipients).await;
    }

```

#### process_vote：接受并处理vote事件（在自己是当前round的proposer的时候）

```rust
 /// Upon new vote:
///
    /// 1. Filter out votes for rounds that should not be processed by this validator (to avoid
    /// potential attacks).
    /// 2. Add the vote to the store and check whether it finishes a QC.
    /// 3. Once the QC successfully formed, notify the Pacemaker.
    #[allow(clippy::collapsible_if)] // Collapsing here would make if look ugly
    pub async fn process_vote(&self, vote: VoteMsg, quorum_size: usize) {
        // Check whether this validator is a valid recipient of the vote.
        let next_round = vote.round() + 1;
        if self
            .proposer_election
            .is_valid_proposer(self.author, next_round)
            .is_none()
        {
            debug!(//自己不是当前round的valid proposer，直接结束
                "Received {}, but I am not a valid proposer for round {}, ignore.",
                vote, next_round
            );
            security_log(SecurityEvent::InvalidConsensusVote)
                .error("InvalidProposer")
                .data(vote)
                .data(next_round)
                .log();
            return;
        }
////自己是valid proposer
        
        let deadline = self.pacemaker.current_round_deadline();
        // TODO [Reconfiguration] Verify epoch of the vote message.
        // Add the vote and check whether it completes a new QC.
        match self
            .block_store
            .insert_vote(vote.clone(), quorum_size)
            .await
        {
            VoteReceptionResult::DuplicateVote => {
                // This should not happen in general.
                security_log(SecurityEvent::DuplicateConsensusVote)
                    .error(VoteReceptionResult::DuplicateVote)
                    .data(vote)
                    .log();
            }
            VoteReceptionResult::NewQuorumCertificate(qc) => {//如果加入这个vote后达到了2f+1，生成了一个合法的QC
                if self.block_store.need_fetch_for_quorum_cert(&qc) == NeedFetchResult::NeedFetch {
                    if let Err(e) = self
                        .sync_manager
                        .fetch_quorum_cert(qc.as_ref().clone(), vote.author(), deadline)
                        .await
                    {
                        error!("Error syncing to qc {}: {:?}", qc, e);
                        return;
                    }
                } else {
                    if let Err(e) = self
                        .block_store
                        .insert_single_quorum_cert(qc.as_ref().clone())
                        .await//将QC插入到本地的block store
                    {
                        error!("Error inserting qc {}: {:?}", qc, e);
                        return;
                    }
                }
                // Notify the Pacemaker about the new QC round.
                self.pacemaker
                    .process_certificates(vote.round(), None)
                    .await;////告诉pacemaker，新的QC产生了
            }
            // nothing interesting with votes arriving for the QC that has been formed
            _ => {}//QC早就集齐2f+1了，这个vote没啥实质作用
        };
    }
```

#### process_commit：

```rust
//英文注释很全面，值得注意的是prune the tree的操作真的很巧妙，他让同步变得简单，并且去除了过去的不再起作用的记录，起到了garbage collection的功能。

/// Upon new commit:
    /// 1. Notify state computer with the finality proof.
    /// 2. After the state is finalized, update the txn manager with the status of the committed
    /// transactions.
    /// 3. Prune the tree.
    async fn process_commit(
        &self,
        committed_block: Arc<Block<T>>,
        finality_proof: LedgerInfoWithSignatures,
    ) {
        // Verify that the ledger info is indeed for the block we're planning to
        // commit.
        assert_eq!(
            finality_proof.ledger_info().consensus_block_id(),
            committed_block.id()
        );

        // Update the pacemaker with the highest committed round so that on the next round
        // duration it calculates, the initial round index is reset
        self.pacemaker
            .update_highest_committed_round(committed_block.round());

        if let Err(e) = self.state_computer.commit(finality_proof).await {
            // We assume that state computer cannot enter an inconsistent state that might
            // violate safety of the protocol. Specifically, an executor service is going to panic
            // if it fails to persist the commit requests, which would crash the whole process
            // including consensus.
            error!(
                "Failed to persist commit, mempool will not be notified: {:?}",
                e
            );
            return;
        }
        // At this moment the new state is persisted and we can notify the clients.
        // Multiple blocks might be committed at once: notify about all the transactions in the
        // path from the old root to the new root.
        for committed in self
            .block_store
            .path_from_root(Arc::clone(&committed_block))
            .unwrap_or_else(Vec::new)
        {
            if let Some(time_to_commit) = duration_since_epoch()
                .checked_sub(Duration::from_micros(committed.timestamp_usecs()))
            {
                counters::CREATION_TO_COMMIT_MS.observe(time_to_commit.as_millis() as f64);
            }
            let compute_result = self
                .block_store
                .get_compute_result(committed.id())
                .expect("Compute result of a pending block is unknown");
            if let Err(e) = self
                .txn_manager
                .commit_txns(
                    committed.get_payload(),
                    compute_result.as_ref(),
                    committed.timestamp_usecs(),
                )
                .await
            {
                error!("Failed to notify mempool: {:?}", e);
            }
        }
        counters::LAST_COMMITTED_ROUND.set(committed_block.round() as i64);
        debug!("{}Committed{} {}", Fg(Blue), Fg(Reset), *committed_block);
        self.block_store.prune_tree(committed_block.id()).await;
    }
```

#### process_chunk_retrieval:

```rust
/// Retrieve the chunk from storage and send it back.
    /// We'll also try to add the QuorumCert into block store if it's for a existing block and
    /// potentially commit.
    pub async fn process_chunk_retrieval(&self, request: ChunkRetrievalRequest) {
        if self
            .block_store
            .block_exists(request.target.certified_block_id())
            && self
                .block_store
                .get_quorum_cert_for_block(request.target.certified_block_id())
                .is_none()
        {
            if let Err(e) = self
                .block_store
                .insert_single_quorum_cert(request.target.clone())
                .await
            {
                error!(
                    "Failed to insert QuorumCert {} from ChunkRetrievalRequest: {}",
                    request.target, e
                );
                return;
            }
            let update_res = self
                .safety_rules
                .write()
                .expect("[state synchronizer handler] unable to lock safety rules")
                .process_ledger_info(&request.target.ledger_info());

            if let Some(block) = update_res {
                self.process_commit(block, request.target.ledger_info().clone())
                    .await;
            }
        }

        let target_version = request.target.ledger_info().ledger_info().version();

        let response = self
            .sync_manager
            .get_chunk(request.start_version, target_version, request.batch_size)
            .await;

        if let Err(e) = request.response_sender.send(response) {
            error!("Failed to return the requested chunk: {:?}", e);
        }
    }
```

