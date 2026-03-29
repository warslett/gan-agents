# Criticism of Draft 3

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
| Adaptability | 8 |
| **Aggregate** | **8.4** |

### Critical Analysis
The Round 3 revisions resolve the remaining structural concerns from Round 2. The deterministic boundary authority rule (lower zone ID is authoritative) is elegant and eliminates the coordinator bottleneck entirely. The zone merge capability for mass events is a pragmatic solution, though the "hot standby" model during a merge means one server's resources are wasted — a cost trade-off that should be acknowledged. The single-server overlap simulation with UDP replication to the non-authoritative server is correct and efficient.

The ECS plugin system is now adequately specified, though the choice between shared libraries and Wasm modules introduces a decision point that could confuse adopters — the proposal should recommend one approach as default (Wasm for safety, native for performance) rather than presenting both equally.

The architecture is extremely well-refined at this point. Its primary limitation compared to the other two proposals is inherent rather than fixable: this is fundamentally a stateful server architecture that requires managing server processes, dealing with server crashes, and handling zone coordination. The operational model is "run and manage game servers" — which is the right model for many games, but less operationally elegant than fully managed game room orchestration (Idea 3). The proposal also does not address how zone boundaries are configured and updated — is the world partitioning static or dynamic? Can zones be split or merged based on load?

This is a strong, production-ready architecture with few remaining weaknesses.

### Verdict: Strong

---

## Idea 2: Distributed Actor Model with Event Sourcing

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 8 |
| Impact | 8 |
| Simplicity | 7 |
| Risk Management | 8 |
| Rigour | 8 |
| Coherence | 9 |
| Adaptability | 9 |
| **Aggregate** | **8.1** |

### Critical Analysis
The revisions continue to strengthen this proposal. The dual-path persistence model (async for gameplay, synchronous for transactions) is well-designed and addresses the durability gap cleanly. The explicit team skill requirements and phased learning timeline are a standout addition — this is the only proposal that honestly discusses the human investment required, which is critical for real-world adoption decisions.

The simplified read model (actors serve reads directly) is a significant improvement that reduces both complexity and latency. This means the day-one stack is genuinely just Orleans + PostgreSQL, which is accessible.

However, this proposal has a structural gap that becomes more apparent in comparison with Idea 3: it does not adequately address the real-time networking layer. How do game clients connect to actors? What transport protocol is used? How is client-side prediction handled when game state is spread across multiple actors? If a player interacts with entities managed by different actors, how is the composite state update delivered to the client as a coherent frame? The proposal focuses heavily on the backend state management (where it excels) but is thin on the client-facing networking protocol — which is arguably the most important part of a "real-time multiplayer game backend." Idea 1 and Idea 3 both specify this layer in detail; Idea 2 does not.

Additionally, while the phased approach reduces day-one complexity, the Phase 1 architecture (room-level actors with traditional state) is essentially just "game rooms as actors" — which does not strongly differentiate from Idea 3's game room model. The unique value of this architecture (per-entity actors, event sourcing, replay) only materializes in Phases 2-3, meaning the initial deployment delivers less value per unit of complexity compared to Idea 3.

### Verdict: Weak

---

## Idea 3: Microservices with Dedicated Game Rooms and Hot-Swappable Logic Modules

### Scores

| Dimension | Score (1-10) |
|---|---|
| Feasibility | 9 |
| Impact | 9 |
| Simplicity | 7 |
| Risk Management | 9 |
| Rigour | 9 |
| Coherence | 9 |
| Adaptability | 10 |
| **Aggregate** | **8.9** |

### Critical Analysis
This proposal continues to lead and has further extended its advantage in Round 3. The double-buffered hot-swap mechanism is a genuine engineering refinement — eliminating the perceptible pause makes zero-downtime updates truly seamless rather than just "minimal disruption." The automatic rollback on error or timeout is a robust safety net.

The Wasm performance characterization is the most honest and detailed technical analysis across all three proposals. Breaking down overhead by operation type (logic, math, memory), citing SIMD mitigations, and providing the native-mode escape hatch demonstrates deep technical understanding. The argument that per-room entity caps make even 2x overhead negligible is persuasive and quantified.

The room allocation system with bin-packing and live migration is a significant addition that no other proposal matches. The ability to live-migrate rooms using the same serialization mechanism as hot-swap is an elegant reuse of infrastructure.

Remaining concerns are truly minor at this point. The proposal does not discuss how clients reconnect if a room is live-migrated — is there a brief disconnect, or does the Gateway handle the transparent re-routing? The "native mode" escape hatch, while pragmatic, means there are now two execution modes to test and maintain — this should be framed as an exceptional measure rather than a standard option. The Gateway Service is a potential bottleneck and single point of failure for all player connections — what is its scaling and HA strategy?

This is the most complete and operationally sophisticated proposal across all dimensions.

### Verdict: Strong

---

## Score Summary

| Idea | Feasibility | Impact | Simplicity | Risk Mgmt | Rigour | Coherence | Adaptability | Aggregate |
|---|---|---|---|---|---|---|---|---|
| 1: Authoritative ECS | 9 | 8 | 7 | 9 | 9 | 9 | 8 | 8.4 |
| 2: Actor + Event Sourcing | 8 | 8 | 7 | 8 | 8 | 9 | 9 | 8.1 |
| 3: Microservices + Wasm | 9 | 9 | 7 | 9 | 9 | 9 | 10 | 8.9 |

---

## Eviction Recommendation

**Evict: Idea 2 — Distributed Actor Model with Event Sourcing**

**Reason:** Idea 2 has the lowest aggregate score (8.1) and a critical gap: it lacks a specified real-time networking layer for client-facing communication — the most important aspect of a "real-time multiplayer game backend." While its backend state management model (actors + event sourcing) is powerful, the Phase 1 deployment (room-level actors) does not strongly differentiate from Idea 3's game room model, meaning the unique value only materializes in later phases at greater complexity cost. Both competing ideas specify their networking layers in detail; Idea 2 does not. The dual-path persistence and phased adoption are excellent features, but they do not compensate for the missing client-facing networking specification in an architecture evaluation focused on real-time multiplayer gameplay.
