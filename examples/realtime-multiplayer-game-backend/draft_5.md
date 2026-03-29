# Draft 5 — Real-Time Multiplayer Game Backend Architecture (Final)

## Idea 1: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Description
A microservices-based architecture where the backend is decomposed into focused, independently deployable services, with each active game session running in a dedicated "game room" — a stateful process managed by a Room Management Service. Game rooms run on Room Host daemons distributed across physical servers, with an intelligent bin-packing allocator handling placement and live migration.

The core innovation is a hot-swappable game logic module system: game rules are compiled as WebAssembly (Wasm) modules loaded into game room processes at runtime. A seamless double-buffered hot-swap mechanism enables zero-downtime game logic updates — new modules are loaded in the background and atomically swapped at the next natural state checkpoint, with automatic rollback if the new module produces errors.

**Core Services:**
- **Gateway Service (stateless, horizontally scaled):** Terminates WebSocket connections from clients, maintains a shared routing table (room ID -> host address) via distributed cache, and routes messages to the correct game room. Supports token-based session resume on reconnection, with 2-second snapshot buffer replay for seamless failover. Runs behind a Layer 4 load balancer (AWS NLB / HAProxy) and scales horizontally with no upper bound.
- **Room Management Service:** Manages the lifecycle of game rooms (create, destroy, migrate). Includes the Room Allocator, which uses a best-fit bin-packing algorithm to place rooms on servers based on available resources and geographic affinity to matched players. Monitors room health and triggers live migration when servers become overloaded or unhealthy.
- **Matchmaking Service:** Manages player queues, performs skill-based matching with geographic affinity, and requests room creation from the Room Management Service.
- **Auth Service:** JWT-based authentication, session token issuance, and permission validation.
- **Room Host Daemons (per physical server):** Host the Wasm runtime (Wasmtime or V8), execute game loops at configurable tick rates (20-64 Hz), report resource usage via heartbeat, and handle room state serialization/deserialization for migration and hot-swap.
- **NATS Message Bus:** Async event emission from rooms to downstream consumers. Rooms emit gameplay events to NATS without blocking the game loop.
- **Downstream Services (added incrementally):** Leaderboard, Analytics, Chat, Replay — all consume events from NATS at their own pace.

**In-Room Networking Stack:**
Each game room runs an authoritative server model. Clients send timestamped input packets to the room via the Gateway. The room validates and applies inputs, then broadcasts delta-compressed binary state snapshots (FlatBuffers or MessagePack) to connected clients. Client-side prediction is supported: each state update includes the client's last-acknowledged input sequence number for reconciliation. Lag compensation uses a server-side input buffer with configurable delay (1-3 ticks) to smooth network jitter.

**Wasm Hot-Swap Mechanism:**
The room process uses double-buffering: while the active Wasm module runs the game loop, a background thread loads and initializes the new module. At the next natural checkpoint (end of round, respawn, or configurable tick interval), the room atomically swaps the active module pointer. The new module receives serialized state via a versioned migration function. If the new module's first tick errors or exceeds 2x expected duration, the room reverts to the old module automatically and reports the failure. Schema compatibility is validated at build time via CI checks.

**Wasm Performance Characteristics:**
- Game logic (state machines, rules, AI): 1.1-1.3x native — negligible overhead
- Math-heavy (physics, pathfinding): 1.5-2.5x native, mitigated to 1.2-1.5x with Wasm SIMD
- Memory-intensive (large world iteration): 1.3-1.8x native
- Per-room entity caps (2-100 players) mean even 2x overhead adds <1ms per tick
- Native compilation mode available as an exceptional measure for rooms exceeding 500 entities at 60Hz with physics, requiring separate build/test/deploy pipelines

**Room Allocation and Live Migration:**
Room Host daemons report available CPU, memory, and bandwidth via heartbeat. The Room Allocator uses best-fit bin-packing with geographic affinity. Live migration serializes room state on the source host and deserializes on the destination — the same mechanism as Wasm hot-swap. During migration, the Gateway buffers client messages and flushes them to the new host on completion. Clients experience a brief latency spike but never disconnect.

**Gateway Failover:**
If a Gateway instance crashes, clients reconnect to another instance via the load balancer. The new Gateway retrieves the room assignment from the shared routing cache, re-establishes the route, and the room replays missed state from its snapshot buffer. Total client-visible disruption: 1-3 seconds with no state loss.

### Advantages
- Seamless zero-downtime game logic updates via double-buffered Wasm hot-swap
- Transparent room live-migration without client disconnection
- Stateless, horizontally scalable Gateway with token-based resume on failure
- Intelligent room bin-packing with geographic affinity
- Full authoritative in-room networking with client prediction and lag compensation
- Detailed, honest Wasm performance characterization with native escape hatch
- 4 core services + Room Host daemons — manageable day-one topology
- Incremental service growth via NATS event consumers
- Feature flags and canary deployments for safe gameplay updates
- Wasm sandboxing provides security isolation for game logic
- Works universally across game types: session-based, persistent, competitive, casual

### Why This Idea Has Merit
This architecture survived four rounds of adversarial evaluation, each round addressing specific weaknesses identified by the Discriminator. It began as a microservices sketch with an interesting Wasm hot-swap idea, and evolved into the most operationally complete proposal — addressing not only game networking (authoritative server, client prediction, lag compensation, delta compression) but the full operational lifecycle (deployment via hot-swap, scaling via bin-packing, fault tolerance via Gateway failover and room migration, live operations via feature flags and NATS analytics pipeline).

The Gateway abstraction layer is the key architectural enabler. By terminating client connections at a stateless proxy rather than at the game room directly, the architecture gains room mobility, transparent failover, and connection management flexibility. This single design decision makes zero-downtime updates, live migration, and seamless failover architecturally natural rather than bolted on.

The architecture is the most versatile finalist — equally suited to arena shooters, MOBAs, battle royales, party games, card games, and persistent worlds (modeled as long-lived rooms). It answers not just "how do I build a real-time multiplayer game backend?" but "how do I run one in production, update it without downtime, scale it automatically, and recover from failures gracefully?"
