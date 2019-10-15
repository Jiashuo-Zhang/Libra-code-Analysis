# safety_rules

**关于safety_rules ,Libra官方的关于LibraBFT的论文中有详细论述，这一部分对外提供各种接口，用于安全的voting和committng，这一部分比较清楚，里面实现的逻辑就是论文里提到的逻辑，如果理解起来又困难可以看Libra-BFT的论文。**

```rust
/// SafetyRules is responsible for two things that are critical for the safety of the consensus:
/// 1) voting rules,
/// 2) commit rules.
/// SafetyRules is NOT THREAD SAFE (should be protected outside via e.g., RwLock).
/// The commit decisions are returned to the caller as result of learning about a new QuorumCert.
pub struct SafetyRules {
    // Keeps the state.
    state: ConsensusState,
}

impl SafetyRules {
    /// Constructs a new instance of SafetyRules given the BlockTree and ConsensusState.
    pub fn new(state: ConsensusState) -> Self {
        Self { state }
    }

    /// Learn about a new quorum certificate. Several things can happen as a result of that:
    /// 1) update the preferred block to a higher value.
    /// 2) commit some blocks.
    /// In case of commits the last committed block is returned.
    /// Requires that all the ancestors of the block are available for at least up to the last committed block, might panic otherwise.
    /// The update function is invoked whenever a system learns about a potentially high QC.
  ///当收到一个可能更高round的QC的时候，会调用这个
    pub fn update(&mut self, qc: &QuorumCert) {
        // Preferred block rule: choose the highest 2-chain head.
        if qc.parent_block_round() > self.state.preferred_block_round() {
            self.state
                .set_preferred_block_round(qc.parent_block_round());
        }
    }

    /// Check if a one-chain at round r+2 causes a commit at round r and return the committed
    /// block id at round r if possible
  ///收到了一个QC，要commit之前某个block，这里就规定了commit哪个
    fn commit_rule_for_certified_block(
        &self,
        block_parent_qc: &QuorumCert,
        block_round: u64,
    ) -> Option<HashValue> {
        // We're using a so-called 3-chain commit rule: B0 (as well as its prefix)
        // can be committed if there exist certified blocks B1 and B2 that satisfy:
        // 1) B0 <- B1 <- B2 <--
        // 2) round(B0) + 1 = round(B1), and
        // 3) round(B1) + 1 = round(B2).

        if block_parent_qc.parent_block_round() + 1 == block_parent_qc.certified_block_round()
            && block_parent_qc.certified_block_round() + 1 == block_round
        {
            return Some(block_parent_qc.parent_block_id());
        }
        None
    }

    /// Clones the up-to-date state of consensus (for monitoring / debugging purposes)
    pub fn consensus_state(&self) -> ConsensusState {
        self.state.clone()
    }

    /// Attempts to vote for a given proposal following the voting rules.
    /// The returned value is then going to be used for either sending the vote or doing nothing.
    /// In case of a vote a cloned consensus state is returned (to be persisted before the vote is
    /// sent).
    /// Requires that all the ancestors of the block are available for at least up to the last
    /// committed block, might panic otherwise.
  ///这里决定是不是要为收到的块vote
    pub fn voting_rule<T: Payload>(
        &mut self,
        proposed_block: &Block<T>,
    ) -> Result<VoteInfo, ProposalReject> {
        if proposed_block.round() <= self.state.last_vote_round() {
            return Err(ProposalReject::OldProposal {
                proposal_round: proposed_block.round(),
                last_vote_round: self.state.last_vote_round(),
            });
        }

        let respects_preferred_block = proposed_block.quorum_cert().certified_block_round()
            >= self.state.preferred_block_round();
        if respects_preferred_block {
            self.state.set_last_vote_round(proposed_block.round());

            // If the vote for the given proposal is gathered into QC, then this QC might eventually
            // commit another block following the rules defined in
            // `commit_rule_for_certified_block()` function.
            let potential_commit_id = self.commit_rule_for_certified_block(
                proposed_block.quorum_cert(),
                proposed_block.round(),
            );

            Ok(VoteInfo {
                proposal_id: proposed_block.id(),
                proposal_round: proposed_block.round(),
                consensus_state: self.state.clone(),
                potential_commit_id,
                parent_block_id: proposed_block.quorum_cert().certified_block_id(),
                parent_block_round: proposed_block.quorum_cert().certified_block_round(),
            })
        } else {
            Err(ProposalReject::ProposalRoundLowerThenPreferredBlock {
                preferred_block_round: self.state.preferred_block_round(),
            })
        }
    }
}

```

