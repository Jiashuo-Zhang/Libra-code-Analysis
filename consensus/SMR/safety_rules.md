# safety_rules

####关于safety_rules ,Libra官方的关于LibraBFT的论文中有详细论述，这一部分对外提供各种接口，用于安全的voting和committng，这一部分比较清楚，建议阅读论文和safety_rules.rs

```rust
/// SafetyRules is responsible for two things that are critical for the safety of the consensus:
/// 1) voting rules,
/// 2) commit rules.
/// The only dependency is a block tree, which is queried for ancestry relationships between
/// the blocks and their QCs.
/// SafetyRules is NOT THREAD SAFE (should be protected outside via e.g., RwLock).
/// The commit decisions are returned to the caller as result of learning about a new QuorumCert.
```

