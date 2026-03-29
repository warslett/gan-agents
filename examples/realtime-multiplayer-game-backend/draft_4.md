# Draft 4 — Real-Time Multiplayer Game Backend Architecture

## Idea 1: Authoritative Server with Entity Component System (ECS) and WebSocket Transport

### Description
A centralized authoritative game server built on an Entity Component System architecture with spatial partitioning for multi-server scaling, decentralized boundary authority, and a sandboxed ECS plugin system.

**Revisions addressing Round 3 criticism:**

**Plugin system default recommendation:** Wasm is now the default and recommended plugin format for all environments. Wasm provides sandboxing, portability, and predictable resource usage out of the box. Native shared libraries are supported ONLY as an opt-in performance override for first-party Systems developed by the core team — never for third-party plugins or mods. This eliminates the decision-point confusion while preserving the escape hatch for performance-critical first-party code.

**Dynamic zone partitioning:** Zone boundaries are no longer static. A Zone Manager service monitors entity density per zone via periodic sampling (every 5 seconds). When a zone's entity count exceeds a configurable high-water mark, the Zone Manager triggers a zone split: the zone's spatial region is divided along the axis with the greatest entity spread, and a new zone server is spun up for the new partition. Conversely, when adjacent zones fall below a low-water mark, they are merged. Split and merge operations use the same serialization and handoff protocol as entity boundary transfers, ensuring consistency. The zone topology is stored in a shared configuration store (etcd) that all zone servers and the matchmaking service read. This makes the architecture reactive to player distribution patterns rather than requiring manual configuration.

**Zone merge cost acknowledgment:** During a zone merge for mass events, the standby server's resources are not wasted — it serves as a read replica for spectator clients and handles non-gameplay queries (leaderboard lookups, chat) for players in the merged zone. This repurposes the standby capacity productively. The merge operation has a pre-configured resource budget; if the merged entity count exceeds the budget, the system falls back to a "boundary event" mode where the two servers coordinate via the overlap protocol rather than fully merging, accepting slightly degraded cross-boundary interactions.

**Operational model comparison:** This architecture requires managing long-lived server processes, but the zone management system (dynamic splitting/merging, entity handoff, crash recovery via Redis snapshots) automates the most complex operational tasks. The operational model is comparable to managing a stateful distributed database cluster — familiar to infrastructure teams experienced with systems like CockroachDB, Cassandra, or Elasticsearch. For teams that prefer managed infrastructure, the zone server model is compatible with container orchestration (Kubernetes StatefulSets with persistent volumes) and cloud game server hosting (Agones on GKE, GameLift).

### Advantages
- Cheat resistance via authoritative server model
- ECS provides cache-friendly, parallelizable game state management
- Dynamic zone partitioning adapts to player distribution automatically
- Decentralized boundary authority with no coordinator bottleneck
- Zone merge with productive standby utilization for mass events
- Wasm-default plugin system with sandboxing and resource budgets
- Compatible with Kubernetes/Agones/GameLift for managed operations
- Blue-green zone deployments, Redis crash recovery, comprehensive monitoring
- Three rounds of refinement addressing scaling, operations, and adaptability

### Why This Idea Has Merit
After three rounds of adversarial refinement, this architecture has evolved from a standard ECS game server description into a comprehensive, production-ready system with dynamic zone management, decentralized coordination, and a specified plugin system. It addresses the full lifecycle: development (plugin system, ECS modularity), deployment (blue-green, Kubernetes compatibility), runtime (dynamic zones, crash recovery), and operations (monitoring, spectator support). Its primary strength is the combination of a battle-tested architectural foundation with explicit solutions for every scaling and operational challenge raised during the adversarial process.

---

## Idea 2: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Description
A microservices-based architecture with dedicated game room containers, seamless double-buffered WebAssembly hot-swap, intelligent room bin-packing with live migration, and a full in-room authoritative networking stack.

**Revisions addressing Round 3 criticism:**

**Client reconnection during live migration:** When a room is live-migrated to a new host, the Gateway Service handles reconnection transparently. The migration sequence: (1) Room Manager notifies Gateway that room R is migrating from Host A to Host B. (2) Gateway begins buffering incoming client messages for room R. (3) Room state is serialized on Host A and deserialized on Host B (50-200ms depending on state size). (4) Room Manager confirms room R is live on Host B. (5) Gateway updates its routing table and flushes buffered messages to Host B. (6) Clients experience a brief latency spike (the buffered messages arrive slightly late) but never disconnect — the WebSocket connection is between the client and the Gateway, not between the client and the room host directly. The Gateway's connection-level abstraction makes room mobility invisible to clients.

**Native mode framing:** Native mode is now explicitly framed as an exceptional measure, not a standard option. It is documented as: "For rooms requiring >500 concurrent entities with physics simulation at 60Hz, where Wasm overhead is measured (not assumed) to exceed tick budget, a native compilation target is available. This requires separate build, test, and deployment pipelines and should be treated as a performance-critical exception, not the default." All game modes start as Wasm; native mode is only considered after profiling demonstrates a concrete need. This prevents teams from prematurely optimizing into native mode and losing the hot-swap capability.

**Gateway scaling and high availability:** The Gateway Service is a stateless WebSocket termination layer — it holds no game state, only a routing table mapping (room ID -> host address). This makes it horizontally scalable: multiple Gateway instances run behind a Layer 4 load balancer (e.g., AWS NLB or HAProxy). The routing table is shared via a lightweight distributed cache (Redis or a gossip-based in-process cache). If a Gateway instance crashes, clients reconnect to another Gateway instance via the load balancer; since the routing table is shared, the new Gateway immediately routes them to the correct room. WebSocket reconnection uses a token-based resume protocol: the client presents a session token, and the new Gateway retrieves the room assignment from the cache, re-establishes the route, and the room process replays missed state updates from its snapshot buffer (last 2 seconds of state). Total client-visible disruption on Gateway failure: 1-3 seconds of reconnection time with no state loss.

**Complete architecture summary:**
- **Gateway Service (stateless, horizontally scaled):** WebSocket termination, routing, session management, token-based reconnection
- **Room Management Service:** Room lifecycle (create/destroy/migrate), bin-packing allocator, health monitoring
- **Matchmaking Service:** Player queue management, skill-based matching, geographic affinity, room creation requests
- **Auth Service:** JWT-based authentication, session tokens, permission validation
- **Room Host Daemons (per physical server):** Host Wasm runtime, execute game loops, report resource usage, handle room serialization/deserialization
- **NATS Message Bus:** Async event emission from rooms to downstream consumers
- **Downstream Services (added incrementally):** Leaderboard, Analytics, Chat, Replay — all consume from NATS

### Advantages
- Seamless double-buffered Wasm hot-swap with automatic rollback
- Transparent client reconnection during room live-migration via Gateway abstraction
- Stateless, horizontally scalable Gateway with token-based resume on failure
- Intelligent room bin-packing with geographic affinity and live migration
- Full authoritative in-room networking: prediction, lag compensation, delta compression
- Characterized Wasm performance with SIMD, native-mode escape hatch (exceptional only)
- 4 core services + Room Host daemons, incremental growth via NATS consumers
- Feature flags and canary deployments for safe zero-downtime gameplay updates
- Three rounds of refinement addressing every identified weakness

### Why This Idea Has Merit
After three rounds of adversarial refinement, this architecture is the most operationally complete proposal. It addresses not only the core game networking (authoritative server, client prediction, lag compensation) but the entire operational lifecycle: deployment (Wasm hot-swap, canary releases), scaling (room bin-packing, Gateway horizontal scaling), fault tolerance (Gateway reconnection, room migration, automatic rollback), and live operations (feature flags, NATS-based analytics pipeline). The Gateway abstraction layer is the key architectural enabler — by terminating client connections at a stateless proxy, the backend gains room mobility, transparent failover, and connection management flexibility that a direct-connection architecture cannot match. This is the architecture that answers not just "how do I build a game backend?" but "how do I run a game backend in production?"
