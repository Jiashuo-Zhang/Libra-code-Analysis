# Network

## 目录：

```
network
├── benches                       # network benchmarks
├── memsocket                     # In-memory transport for tests
├── netcore
│   └── src
│       ├── multiplexing          # substream multiplexing over a transport
│       ├── negotiate             # protocol negotiation
│       └── transport             # composable transport API
├── noise                         # noise framework for authentication and encryption
└── src
    ├── channel                    # mpsc channel wrapped in IntGauge
    ├── connectivity_manager       # component to ensure connectivity to peers
    ├── interface                  # generic network API
    ├── peer_manager               # component to dial/listen for connections
    ├── proto                      # protobuf definitions for network messages
    ├── protocols                  # message protocols
    │   ├── direct_send            # protocol for fire-and-forget style message delivery
    │   ├── discovery              # protocol for peer discovery and gossip
    │   ├── health_checker         # protocol for health probing
    │   └── rpc                    # protocol for remote procedure calls
    ├── sink                       # utilities over message sinks
    └── validator_network          # network API for consensus and mempool
```

## 结构：

```rust
                             +---------------------+---------------------+
                             |      Consensus      |       Mempool       |
                             +---------------------+---------------------+
                             |            Validator Network              |
                             +---------------------+---------------------+
                             |            NetworkProvider                |
+------------------------------------------------+-----------------+     |
| Discovery, health, etc     |            RPC    |  DirectSend     |     |
+--------------+---------------------------------------------------------+
|                                         Peer Manager                   |
+------------------------------------------------------------------+-----+
```

## 功能：

**netwotk维护了node之间的通信，维护了peer们，还给出了访问validator的mempool，consensus等模块的功能**

