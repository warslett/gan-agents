# Criticism of Draft 2

## Idea 1: Authoritative Server with Entity Component System (ECS) and WebSocket Transport

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 9 |
| Impact | 8 |
| Simplicity | 7 |
| Risk Management | 9 |
| Rigour | 8 |
| Coherence | 9 |
| Adaptability | 8 |
| **Aggregate** | **8.3** |

### Critical Analysis
The revisions meaningfully address the weaknesses identified in Round 1. The spatial partitioning strategy for multi-server scaling is well-described, and the two-phase entity handoff protocol with overlapping zones is a realistic approach used by production MMO engines. The blue-green deployment at the zone level and Redis-based crash recovery add genuine operational maturity.

However, the spatial partitioning approach introduces its own complexity. The Boundary Coordination Service is a potential bottleneck and single point of failure for cross-zone interactions. What happens when a large-scale event (a raid boss fight, a massive PvP battle) draws hundreds of players to a zone boundary? The overlap region simulation (both servers running the same entities) doubles the compute cost in boundary areas and introduces subtle consistency challenges: which server is authoritative for an entity in the overlap? The proposal mentions a "Boundary Service resolving authority" but does not specify the resolution mechanism. Additionally, the ECS plugin system for dynamic System registration is mentioned but not specified — how are plugins distributed, versioned, and sandboxed?

Despite these gaps, the revisions have lifted this proposal from a standard textbook description to a more production-ready architecture. The adaptability score improved significantly with the plugin system addition.

### Verdict: Strong

---

## Idea 2: Distributed Actor Model with Event Sourcing

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 8 |
| Impact | 8 |
| Simplicity | 6 |
| Risk Management | 7 |
| Rigour | 8 |
| Coherence | 9 |
| Adaptability | 9 |
| **Aggregate** | **7.9** |

### Critical Analysis
The revisions represent a substantial improvement. The tiered event persistence strategy is the strongest addition — by making Kafka writes asynchronous and accepting ~100ms of potential event loss on crash, the latency concern is effectively resolved for gameplay operations. This is a pragmatic engineering trade-off that demonstrates architectural maturity.

The phased adoption strategy is also excellent. Starting with room-level actors and room-level events, rather than per-entity actors and per-tick events, reduces the initial system complexity by an order of magnitude while preserving the ability to scale up later. This makes the architecture accessible to teams that are not distributed systems experts.

The genre positioning is honest and helpful, though it slightly narrows the proposal's appeal. The remaining concern is that even with the phased approach, the technology stack (Orleans/Akka + Kafka + CQRS projections + distributed actor cluster) is still significantly more complex than the competing proposals. A team choosing this architecture needs operational expertise in both actor frameworks and event streaming platforms — a relatively rare combination. The proposal would benefit from discussing the team skill requirements and learning curve more explicitly. Additionally, the ~100ms event loss window on crash could be problematic for transactional operations (trades, purchases) even in slower-paced games — this edge case needs a separate durability path.

### Verdict: Strong

---

## Idea 3: Serverless Edge Computing with Conflict-Free Replicated Data Types (CRDTs)

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 6 |
| Impact | 6 |
| Simplicity | 5 |
| Risk Management | 5 |
| Rigour | 6 |
| Coherence | 7 |
| Adaptability | 7 |
| **Aggregate** | **6.0** |

### Critical Analysis
The revisions represent a significant improvement over the original proposal. The three most impactful changes are: (1) switching from pure serverless to persistent edge processes (Durable Objects), which resolves the platform constraint issues; (2) the tiered consistency model, which honestly addresses the eventual consistency limitation; and (3) the explicit genre scoping to casual/social games.

However, the architecture still faces fundamental challenges. Durable Objects are a Cloudflare-specific technology, creating strong vendor lock-in. Fly.io and Railway are mentioned as alternatives, but they do not offer an equivalent abstraction — the architecture would need significant redesign for different platforms. The tiered consistency model, while theoretically sound, adds complexity: developers must correctly classify every piece of game state into the right tier, and mistakes lead to subtle bugs (e.g., an inventory race condition because an item interaction was placed in Tier 3 instead of Tier 1). There is no tooling or framework described to help developers make these classifications.

The CRDT implementations are now more specific and practical, which is good. However, the custom delta-state CRDT for world blocks is mentioned without detail — custom CRDTs are extremely difficult to implement correctly (proving convergence is non-trivial). The gossip protocol for cross-edge-node state dissemination is also underspecified: in a globally distributed system, gossip convergence time can be seconds, which may create jarring inconsistencies even in casual games.

This proposal has improved from "Weak" to a reasonable niche architecture, but it remains the weakest of the surviving ideas due to vendor lock-in risks, the complexity of correctly applying the consistency tiers, and the relative immaturity of CRDTs in game backend contexts.

### Verdict: Weak

---

## Idea 4: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 9 |
| Impact | 9 |
| Simplicity | 7 |
| Risk Management | 9 |
| Rigour | 9 |
| Coherence | 9 |
| Adaptability | 9 |
| **Aggregate** | **8.7** |

### Critical Analysis
This is the strongest revision across all four proposals. Every criticism from Round 1 has been directly and substantively addressed. The in-room networking stack is now fully specified with tick rate, input processing, delta compression, client prediction, and lag compensation — bringing it up to par with Idea 1's networking detail. The Wasm hot-swap safety mechanism is impressive: the pause-serialize-migrate-resume protocol with automatic rollback and build-time schema validation demonstrates serious engineering thinking. The cross-service latency solution (confining all latency-sensitive operations to the room process) is clean and correct.

The reduced service count (4 core services with incremental additions) is a smart pragmatic choice that acknowledges the operational cost of microservices. The specification that the game loop never blocks on external service calls, with async event emission via NATS for downstream services, is exactly the right pattern.

The remaining concerns are minor. The 1-2 tick pause during Wasm hot-swap (up to ~100ms at 20Hz) may be noticeable in competitive scenarios — could this be masked or eliminated with a double-buffering approach? The 1.2-2x Wasm performance overhead claim needs qualification: this holds for computational logic but Wasm's memory access patterns and lack of SIMD in some runtimes could make physics-heavy rooms more expensive. Finally, the proposal does not discuss how game rooms are allocated to physical servers — is there an orchestrator that bin-packs rooms onto machines based on CPU/memory availability?

These are refinement-level concerns, not structural weaknesses. This proposal is the most complete and operationally mature of the four.

### Verdict: Strong

---

## Score Summary

| Idea | Feasibility | Impact | Simplicity | Risk Mgmt | Rigour | Coherence | Adaptability | Aggregate |
|---|---|---|---|---|---|---|---|---|
| 1: Authoritative ECS | 9 | 8 | 7 | 9 | 8 | 9 | 8 | 8.3 |
| 2: Actor + Event Sourcing | 8 | 8 | 6 | 7 | 8 | 9 | 9 | 7.9 |
| 3: Serverless Edge + CRDTs | 6 | 6 | 5 | 5 | 6 | 7 | 7 | 6.0 |
| 4: Microservices + Wasm | 9 | 9 | 7 | 9 | 9 | 9 | 9 | 8.7 |

---

## Eviction Recommendation

**Evict: Idea 3 — Serverless Edge Computing with Conflict-Free Replicated Data Types (CRDTs)**

**Reason:** Despite meaningful improvements from Round 1 (score rose from 5.0 to 6.0), Idea 3 remains the weakest proposal by a significant margin (6.0 vs. 7.9+ for the others). The vendor lock-in risk with Durable Objects, the complexity of correctly classifying game state into consistency tiers, the underspecified custom CRDTs and gossip protocol, and the relative immaturity of CRDTs in game contexts all contribute to a proposal that is architecturally interesting but practically risky. The niche it serves (casual/social multiplayer) is better addressed by simpler architectures from the other proposals.
