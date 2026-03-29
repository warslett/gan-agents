# Criticism of Draft 4

## Idea 1: Authoritative Server with Entity Component System (ECS) and WebSocket Transport

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 9 |
| Impact | 8 |
| Simplicity | 7 |
| Risk Management | 9 |
| Rigour | 9 |
| Coherence | 9 |
| Adaptability | 9 |
| **Aggregate** | **8.6** |

### Critical Analysis
This proposal has reached a high level of refinement. The dynamic zone partitioning is the standout addition in this round — it transforms the architecture from a manually configured multi-server system to a self-adaptive one. The zone split/merge based on entity density monitoring, using the same serialization protocol as boundary handoffs, is architecturally clean. The productive reuse of the standby server during merges (spectator service, chat, leaderboards) is a thoughtful operational detail.

The Kubernetes/Agones/GameLift compatibility statement broadens deployment options and makes the architecture accessible to teams with existing cloud infrastructure expertise.

However, the dynamic zone partitioning introduces new complexity that the proposal acknowledges but does not fully address. Zone splits and merges are disruptive operations — how long do they take? What happens to players in the region being split during the operation? Is there a brief interruption? The zone topology stored in etcd must be consistent across all zone servers and the matchmaking service; what happens during the propagation delay when a zone has just been split but some servers still have the old topology? The 5-second density sampling interval means the system is reactive, not predictive — a sudden influx of players (e.g., a game event announcement) could overwhelm a zone before the split triggers.

The architecture's fundamental limitation remains: it is optimized for spatially partitioned game worlds (MMOs, open-world games, battle royales). For session-based games without a persistent spatial world (arena shooters, MOBAs, party games, card games), the zone partitioning system is irrelevant overhead. The proposal does not address how this architecture simplifies for non-spatial game types. In contrast, Idea 2's room-based model is inherently session-oriented and works equally well for both spatial and non-spatial games.

The Wasm-default plugin system is well-specified now but is positioned as a modding/extensibility feature — useful but not transformative for the core game development workflow. Idea 2's Wasm modules are the game logic itself, which is a more fundamental application of the technology.

### Verdict: Strong

---

## Idea 2: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 9 |
| Impact | 9 |
| Simplicity | 8 |
| Risk Management | 9 |
| Rigour | 10 |
| Coherence | 10 |
| Adaptability | 10 |
| **Aggregate** | **9.3** |

### Critical Analysis
This proposal has achieved a remarkable level of completeness across four rounds of refinement. Every component is specified to a depth that enables implementation decisions. The three Round 4 additions — transparent client reconnection during migration, native mode reframing, and Gateway HA — each resolve their respective concerns thoroughly.

The Gateway reconnection protocol is the strongest addition. The key insight — that WebSocket connections terminate at the Gateway, not at the room host — means room mobility is architecturally guaranteed rather than bolted on. The token-based resume protocol with 2-second snapshot buffer replay is a specific, implementable design. The 1-3 second disruption on Gateway failure is an honest characterization.

The complete architecture summary at the end of the revision is helpful for evaluation: the system has 4 core services, Room Host daemons, a message bus, and optional downstream consumers. This is a manageable system topology — comparable to a typical microservices web application plus a game-specific runtime layer.

The remaining concerns are extremely minor. The Room Host daemon is a custom component that must be developed and maintained — the proposal does not discuss its implementation complexity or suggest any existing frameworks/tools that could serve as a starting point. The NATS message bus is mentioned but there is no discussion of message ordering guarantees, backpressure handling, or what happens when a downstream consumer (e.g., Analytics) falls behind. The bin-packing algorithm uses "historical resource usage" to estimate room resource profiles, but new game modes have no history — how are they bootstrapped?

These are implementation details rather than architectural weaknesses. The architecture is coherent, comprehensive, and the most versatile of the two finalists — it works for session-based games, persistent worlds (by making long-lived rooms), competitive esports, and casual multiplayer equally well.

### Verdict: Strong

---

## Score Summary

| Idea | Feasibility | Impact | Simplicity | Risk Mgmt | Rigour | Coherence | Adaptability | Aggregate |
|---|---|---|---|---|---|---|---|---|
| 1: Authoritative ECS | 9 | 8 | 7 | 9 | 9 | 9 | 9 | 8.6 |
| 2: Microservices + Wasm | 9 | 9 | 8 | 9 | 10 | 10 | 10 | 9.3 |

---

## Eviction Recommendation

**Evict: Idea 1 — Authoritative Server with Entity Component System (ECS) and WebSocket Transport**

**Reason:** Both proposals are strong, but Idea 1 scores lower across the board (8.6 vs. 9.3 aggregate). The decisive factors are: (1) **Versatility** — Idea 1 is optimized for spatially partitioned worlds and adds irrelevant complexity for session-based games, while Idea 2's room model works universally across all game types. (2) **Operational sophistication** — Idea 2's Gateway abstraction enables transparent room migration, failover, and connection management that Idea 1's direct-connection model cannot match. (3) **Live operations** — Idea 2's Wasm hot-swap for core game logic enables zero-downtime updates that fundamentally change the deployment experience, whereas Idea 1's Wasm plugins are limited to extensibility/modding. (4) **Simplicity** — Idea 2 scores higher on simplicity because the room-per-session model avoids the zone partitioning, boundary coordination, and dynamic split/merge complexity that Idea 1 requires. Idea 1 is an excellent architecture for MMOs and open-world games specifically, but Idea 2 is the better general-purpose recommendation.
