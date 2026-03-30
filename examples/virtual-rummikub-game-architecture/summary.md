# Summary — Virtual Rummikub Game Architecture

## 1. Original Problem

> "What software architecture should I use for a virtual rummikub game allowing multiple players to play against each other or against an AI over the web?"

[Research undertaken first](research/index.md)

**Scoring Dimensions:**
- **Feasibility** — Can this architecture realistically be built with commonly available tools, libraries, and developer skills?
- **Simplicity** — Is the architecture easy to understand and explain, with minimal moving parts?
- **Fault Tolerance** — Can the system continue functioning, even partially, when components fail?
- **Maintainability** — How easy is the architecture to monitor, trace, debug and audit?
- **Adaptability** — How easily can new game modes, rules variants, or features be added without major rework?
- **Security** — How well does the system defend against malicious external actors?
- **Scalability** — How many simultaneous games and connected players can the system handle before performance degrades?

## 2. Final Idea

### Event-Sourced Game Actors with Declarative Move Protocol

This architecture models each Rummikub game as a lightweight, event-sourced actor running in a stateful serverless environment. The design is deliberately platform-portable: the game actor is a plain TypeScript class that receives messages, manages an event log with co-located storage, and pushes updates over WebSockets. This class can be deployed on Cloudflare Durable Objects, on AWS Lambda with DynamoDB and API Gateway WebSockets, on Rivet Actors, or even on a self-hosted Node.js process with in-memory or Redis-backed state. A thin platform adapter layer (under 200 lines) maps the actor interface to the host environment's storage and WebSocket APIs, keeping the entire game engine portable.

Each game actor maintains its state as an ordered log of moves (events). To avoid replay latency growing with game length, the actor persists periodic snapshots—for example, every 10 moves—so that state reconstruction only replays events since the last snapshot. When a player submits a move, the actor loads the most recent snapshot, replays subsequent events, validates the move, appends the event to the log, and pushes updated state to connected clients. Serverless deployments only execute when moves are made, matching the bursty nature of turn-based play where minutes may pass between actions.

The heart of the system is a declarative move protocol. During a player's turn, all tile manipulation happens entirely on the client side within an isolated "sandbox" copy of the board state. When the player commits their turn, the client computes a declarative move description: a structured diff specifying which tiles moved from rack to board, which tiles were rearranged between sets, and the final composition of every set on the board. This single atomic message is sent to the server actor, which independently validates it by checking that all sets are legal, no tiles have appeared or vanished, the initial meld requirement is met if applicable, and at least one new tile was played from the rack. If validation fails, the server rejects the move and the client reverts to the pre-turn snapshot.

The game validation logic is written in TypeScript as a pure-function module with no runtime-specific dependencies, ensuring it runs identically on any server environment and in the browser. The client uses this shared module to provide instant feedback during tile placement before the server round-trip, but the server's validation is always authoritative.

AI opponents are implemented as difficulty-tiered functions behind a simple internal API contract. A "beginner" AI uses a fast greedy heuristic, while an "expert" AI uses a backtracking solver written in pure TypeScript to avoid native dependency issues across environments. These AI functions are invoked directly by the game actor on the server side, not as separate HTTP microservices, eliminating network latency and external failure points. If an AI computation exceeds a time budget (e.g., 500ms), the game actor falls back to the greedy heuristic, ensuring turns always complete.

For fault tolerance, the event log is the system's primary durability mechanism. Each game actor periodically flushes its event log and latest snapshot to an external durable store (such as S3, R2, or a database) as a background operation. If the actor host crashes or becomes unavailable, a new actor instance can be spun up and reconstruct full game state by loading the latest backup and replaying events since the last flush. For connection resilience, the client persists the current sandbox state to local storage during a turn, so that if a WebSocket connection drops mid-manipulation, the player can reconnect and resume editing without losing work.

Authentication and authorization are enforced at the actor entry point via JWT tokens (issued by a separate auth service with standard token rotation and short expiry times). WebSocket subscriptions are scoped so that each player receives only the public board state and their own rack—never another player's rack. Spectators receive board-only state updates. For games that attract many spectators, the actor can delegate fan-out to a lightweight pub-sub relay rather than pushing to every WebSocket directly, keeping the actor's own workload bounded.

The event-sourced log provides built-in support for full game replay, spectator mode, undo/redo within a turn (client-side), post-game analytics, and debugging. Events are versioned with a schema version field, and the replay logic includes lightweight migration transforms that upcast old event formats to the current schema during reconstruction.

**Why it survived:** This idea won because it balanced multiple competing concerns more effectively than the alternatives throughout the three-round selection process. In Round 1, it scored 5.3—better than both the CRDT-based architecture (4.1) and WebRTC P2P (4.0)—because it traded architectural sophistication for practical simplicity. While it stacked multiple patterns, the event-sourced declarative move protocol proved more tractable than CRDTs with WASM ILP solvers or peer-to-peer mesh networks.

In Round 2, the idea was refined (scoring 6.4) by explicitly making it platform-portable rather than Cloudflare-specific, and by clarifying that AI solvers should be pure TypeScript backtracking rather than native ILP libraries. This portability became crucial: it meant the architecture could run on any major platform (Cloudflare, AWS, GCP, self-hosted) with a thin adapter, dramatically reducing vendor lock-in and broadening its applicability. The competing WASM-shared-core architecture (6.1) introduced the implementation burden of Rust/WASM compilation, which the jury rated as a disadvantage in feasibility and maintainability.

In Round 3, the final refinement (scoring 6.7) addressed the most pressing criticisms: explicitly discussing external backup semantics for fault tolerance, detailing spectator fan-out delegation for scalability, and acknowledging that the thin adapter layer had limits but that pragmatic acceptance of platform-specific tradeoffs was reasonable. The idea's aggregated trajectory—5.3 → 6.4 → 6.7—showed consistent improvement through targeted architectural refinement. It survived because it offered a legitimate path to production: it required no exotic technologies (CRDTs, WASM compilers, WebRTC mesh), could be implemented by standard full-stack JavaScript teams, and scaled to thousands of games while keeping each game as a bounded, testable unit.

## 3. Ideas Overview

3 ideas were generated in Round 1:

1. **CRDT-Based Optimistic Board Manipulation with WASM-Shared Game Logic** -- A three-layer architecture using Conflict-free Replicated Data Types for real-time board spectating, a shared Rust/WASM game core for validation, and an event-sourced log, with an ILP-based AI solver.

2. **Serverless Event-Sourced Actor Architecture with Pluggable AI Microservices and Offline-First PWA Shell** -- A serverless architecture where each game is an event-sourced actor, with AI as decoupled HTTP microservices and an offline-first PWA client using client-side WASM-compiled solvers for practice.

3. **WebRTC Peer-to-Peer Mesh with Ephemeral Validation Relay and Client-Side Tiered AI** -- A peer-to-peer architecture using WebRTC data channels for direct player-to-player communication, a stateless validation relay for turn boundaries, and client-side WASM ILP solvers for AI.

## 4. Evolution Through Rounds

### Event-Sourced Game Actors with Declarative Move Protocol (Idea 2) — WINNER

The idea evolved from a Cloudflare Durable Objects-specific architecture in Round 2 to a platform-portable actor model in Round 3, with refinements to fault tolerance semantics (explicit external backups), scalability (spectator fan-out delegation), and realistic tradeoff acknowledgment.

| Round | Aggregate | Idea | Criticism |
|---|---|---|---|
| 1 | 5.3 | [round_1/ideas/idea_2.md](round_1/ideas/idea_2.md) | [round_1/criticism/idea_2_criticism.md](round_1/criticism/idea_2_criticism.md) |
| 2 | 6.4 | [round_2/ideas/idea_2.md](round_2/ideas/idea_2.md) | [round_2/criticism/idea_2_criticism.md](round_2/criticism/idea_2_criticism.md) |
| 3 | 6.7 | [round_3/ideas/idea_2.md](round_3/ideas/idea_2.md) | [round_3/criticism/idea_2_criticism.md](round_3/criticism/idea_2_criticism.md) |

### WASM-Shared Game Logic with Real-Time Board Spectating (Idea 1) — Evicted Round 3

This idea proposed a two-layer architecture using Rust/WASM for the game core and WebSocket throttled broadcasts for spectating, but was eliminated because the Rust/WASM toolchain complexity and narrower hiring pool were seen as disadvantages compared to the pure-TypeScript portable actor approach.

| Round | Aggregate | Idea | Criticism |
|---|---|---|---|
| 1 | 4.1 | [round_1/ideas/idea_1.md](round_1/ideas/idea_1.md) | [round_1/criticism/idea_1_criticism.md](round_1/criticism/idea_1_criticism.md) |
| 2 | 6.1 | [round_2/ideas/idea_1.md](round_2/ideas/idea_1.md) | [round_2/criticism/idea_1_criticism.md](round_2/criticism/idea_1_criticism.md) |

### WebRTC Peer-to-Peer Mesh with Ephemeral Validation Relay and Client-Side Tiered AI (Idea 3) — Evicted Round 2

This architecture relied on WebRTC peer-to-peer connections with a stateless validation relay, but was eliminated early because the fundamental security flaw (clients hold game state, validation only at turn boundaries), combined with NAT traversal unreliability and extreme operational complexity for debugging distributed peer systems, outweighed its cost advantages.

| Round | Aggregate | Idea | Criticism |
|---|---|---|---|
| 1 | 4.0 | [round_1/ideas/idea_3.md](round_1/ideas/idea_3.md) | [round_1/criticism/idea_3_criticism.md](round_1/criticism/idea_3_criticism.md) |

## 5. Process Statistics

- **Total rounds:** 3 (1 ideation + 2 eviction/revision rounds)
- **Total ideas generated:** 3
- **Ideas evicted:** 2
- **Final survivor:** Event-Sourced Game Actors with Declarative Move Protocol (Idea 2)
