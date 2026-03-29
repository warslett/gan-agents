# Draft 3 — Real-Time Multiplayer Game Backend Architecture

## Idea 1: Authoritative Server with Entity Component System (ECS) and WebSocket Transport

### Description
A centralized authoritative game server built on an Entity Component System architecture, using WebSocket connections for bidirectional real-time communication. The server owns the canonical game state represented as entities with composable components, processed by systems at a fixed tick rate. Multi-server scaling is achieved through spatial partitioning with overlapping zone boundaries.

**Revisions addressing Round 2 criticism:**

**Boundary Coordination resilience:** The Boundary Coordination Service is no longer a single centralized service. Instead, zone boundary authority is resolved locally between the two adjacent zone servers using a deterministic rule: the server with the lower zone ID is authoritative for entities in the overlap region. This eliminates the need for a third-party coordinator entirely. When an entity moves fully into a new zone (exits the overlap), authority transfers via a direct two-phase handoff between the two zone servers. For large-scale cross-boundary events (boss fights, mass PvP), a "zone merge" operation is triggered: the two adjacent zone servers temporarily merge into a single logical server, with one server handling the combined state and the other becoming a hot standby. This is pre-configured for known hot-spot areas in the game world and can be triggered dynamically based on entity density metrics.

**Overlap region efficiency:** Entities in the overlap region are simulated by the authoritative server only (not both servers). The non-authoritative server receives replicated state for overlap entities via a dedicated inter-zone replication channel (binary, UDP-based, running at the same tick rate). This eliminates the doubled compute cost while maintaining seamless player experience across boundaries.

**ECS plugin system specification:** Plugins are packaged as shared libraries (.so/.dll) or WebAssembly modules, loaded via a plugin registry service. Each plugin declares its Component types and System registrations with a semantic version. Plugins are sandboxed: they can only access Components they declare in their manifest, and system execution is monitored for CPU budget overruns (systems that exceed their tick budget are disabled with a warning). Plugin hot-loading is supported for development environments; production deployments use a restart-on-update model for safety. Plugins are distributed via an artifact repository (similar to a package registry) and versioned with the game server release.

### Advantages
- Cheat resistance: authoritative server model
- ECS provides cache-friendly, parallelizable game state management
- Decentralized boundary resolution eliminates coordinator as bottleneck/SPOF
- Zone merge capability handles mass events at boundaries
- Efficient overlap replication avoids doubled compute costs
- Specified plugin system with sandboxing and budget enforcement
- Blue-green zone deployments, Redis crash recovery, comprehensive monitoring
- Mature, industry-proven architectural foundation

### Why This Idea Has Merit
This architecture has progressively addressed every criticism raised across two rounds. The boundary coordination is now fully decentralized with a deterministic authority rule, eliminating the bottleneck concern. The overlap region is simulated once (not twice), resolving the efficiency concern. The ECS plugin system is now specified with versioning, sandboxing, and budget enforcement. This proposal represents the most refined version of the industry-standard approach, enhanced with explicit solutions for the scaling and operational challenges that the standard approach typically leaves unaddressed.

---

## Idea 2: Distributed Actor Model with Event Sourcing

### Description
An architecture based on the Actor Model where game entities and rooms are represented as virtual actors with encapsulated state and asynchronous message passing. A tiered event persistence strategy provides event sourcing benefits without latency penalties. A phased adoption strategy enables incremental complexity.

**Revisions addressing Round 2 criticism:**

**Transactional durability path:** A separate durability tier is introduced for transactional operations. Player-to-player trades, currency transactions, and item transfers use a synchronous write path: the event is written to a local WAL (Write-Ahead Log) on the actor's host node AND acknowledged by at least one Kafka replica before the actor responds to the client. This adds ~5-15ms of latency to transactional operations but guarantees zero data loss. The async batch flush (~100ms window) remains for non-transactional gameplay events where the latency trade-off is appropriate. Game developers annotate commands as "transactional" or "gameplay" in the actor's message handler, and the framework routes them to the appropriate persistence path automatically.

**Team skill requirements and learning curve:** The proposal now explicitly addresses the human side. The recommended team profile includes at least one engineer with distributed systems experience (actor frameworks or event streaming), and the phased approach is specifically designed to limit the learning curve. Phase 1 (room-level actors, traditional state) requires only basic Orleans/Akka knowledge — roughly equivalent to learning a new web framework. Phase 2 (room-level event sourcing) requires understanding event sourcing patterns but not distributed CRDT theory. Phase 3 (per-entity actors) is only needed for massive worlds and requires deep distributed systems expertise. Estimated ramp-up time: Phase 1 is 2-4 weeks for a mid-level team, Phase 2 is 1-2 months, Phase 3 is 3-6 months. The proposal includes a recommended learning path and reference implementations.

**Reduced technology stack:** The CQRS projection layer is simplified. Instead of a separate projection service, Orleans grain state IS the read model for real-time queries (actors serve reads directly from memory). Kafka projections are only used for offline analytics, leaderboards, and replay — systems that can tolerate seconds of delay. This eliminates one entire service layer from the hot path and reduces the day-one technology requirements to: Orleans + a persistent store (PostgreSQL) + Kafka (added in Phase 2). The PostgreSQL backing store is familiar to most teams, reducing the learning curve further.

### Advantages
- Natural concurrency model with sequential per-actor message processing
- Tiered persistence: async for gameplay events, synchronous WAL+Kafka for transactions
- Simplified stack: Orleans + PostgreSQL + Kafka (Phase 2), no separate CQRS projection service
- Phased adoption with explicit team skill requirements and learning timelines
- Actors serve as both write and read models for real-time queries
- Powerful operational tooling: replay, audit, analytics from the event log
- Automatic distribution, failover, and rebalancing via Orleans' silo system
- Honest genre positioning for MMOs, sandbox worlds, strategy games

### Why This Idea Has Merit
The revisions address the two remaining Round 2 concerns: transactional durability and team accessibility. The dual-path persistence (async for gameplay, synchronous for transactions) provides the right durability guarantee for each operation type without penalizing the hot gameplay path. The explicit team skill requirements and phased learning path make the architecture accessible to teams that are not distributed systems experts. The simplified read path (actors serve reads directly) removes an entire service layer. This architecture remains uniquely powerful for games with complex, persistent, long-lived state, and is now more honestly positioned about the investment required to adopt it.

---

## Idea 3: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Description
A microservices-based architecture with dedicated game room containers, hot-swappable WebAssembly game logic modules, and a full in-room authoritative networking stack. The architecture starts with 4 core services and grows incrementally.

**Revisions addressing Round 2 criticism:**

**Seamless Wasm hot-swap via double-buffering:** The hot-swap mechanism is refined to eliminate the perceptible pause. The room process now uses a double-buffer approach: while the current Wasm module continues running the game loop, a background thread loads and initializes the new module. At the next natural state checkpoint (end of a round, respawn, or configurable tick interval), the room atomically swaps the active module pointer. The new module receives the current serialized state (using the migration function) and begins processing the next tick. From the client's perspective, this is invisible — no paused ticks, no stutter. The rollback mechanism remains: if the new module's first tick produces an error or takes >2x the expected duration, the room reverts to the old module and reports the failure to the deployment system.

**Wasm performance characterization:** The 1.2-2x performance overhead is now qualified with specifics. For game logic (state machines, rule evaluation, AI decision trees): 1.1-1.3x native, effectively negligible. For math-heavy operations (physics, pathfinding): 1.5-2.5x native, mitigated by using Wasm SIMD proposals (now supported in Wasmtime and V8) which bring physics-heavy workloads to 1.2-1.5x. For memory-intensive operations (large world state iteration): 1.3-1.8x due to Wasm's linear memory model vs. native pointer indirection. The per-room model caps entity count (typically 2-100 players per room), which means even 2x overhead results in <1ms additional tick time — well within budget for a 20-60Hz loop. For exceptionally demanding scenarios, the architecture supports "native mode" rooms that run compiled Rust/C++ directly (bypassing Wasm), sacrificing hot-swap for performance.

**Room allocation and bin-packing:** A Room Allocator component within the Room Management Service handles physical server placement. Each game server machine runs a Room Host daemon that reports available resources (CPU, memory, network bandwidth) to the Room Allocator via heartbeat. When a new room is needed, the Allocator uses a best-fit bin-packing algorithm: it selects the server with the most available resources that exceeds the room's resource profile (estimated from the game mode's historical resource usage). Rooms that exceed their resource estimates trigger an alert and can be live-migrated to a less loaded server: the room state is serialized, sent to the new host, and deserialized — the same mechanism used for Wasm hot-swap, repurposed for room mobility. Geographic affinity is also considered: rooms are preferentially placed on servers closest to the matched players' median geographic location.

### Advantages
- Independent scaling of each service based on specific load profile
- Seamless Wasm hot-swap via double-buffering with automatic rollback
- Characterized Wasm performance with SIMD support and native-mode escape hatch
- Intelligent room allocation with bin-packing, geographic affinity, and live migration
- Full in-room authoritative networking stack with prediction and lag compensation
- No cross-service calls in the latency-sensitive game loop
- 4 core services initially, incremental growth for additional capabilities
- Feature flags and canary deployments for safe gameplay updates
- Wasm sandboxing provides security isolation

### Why This Idea Has Merit
This proposal has now addressed three rounds of criticism and has become the most operationally complete architecture of the three remaining ideas. The double-buffered hot-swap eliminates the last UX concern. The performance characterization is honest and detailed, with a native-mode escape hatch for demanding scenarios. The room allocation system with bin-packing and live migration demonstrates that the architecture handles not just the game logic but the full operational lifecycle. This architecture is the most versatile — suitable for shooters, MOBAs, battle royales, party games, and any session-based multiplayer format — while also being the most operationally sophisticated with its zero-downtime update capability.
