# Multiplayer Game Architecture Patterns Overview

**Source:** Compiled from common game development architecture resources and patterns

## Extracted Information

### Core Architecture Choices

**1. Client-Server (Authoritative Server)**
- Server maintains the canonical game state
- Clients send actions/intents; server validates and applies them
- Server broadcasts updated state to all clients
- Pros: Cheat-proof, single source of truth, easy to add AI players server-side
- Cons: Higher server costs, added latency, server is a single point of failure

**2. Peer-to-Peer**
- One client acts as host or all clients share state
- Reduced server requirements (only signaling needed)
- Pros: Lower infrastructure costs, lower latency between peers
- Cons: Hard to prevent cheating, hard to add AI opponents, host migration is complex

**3. Client-Server with Client-Side Prediction**
- Client predicts outcomes immediately for responsiveness
- Server validates and corrects if needed
- Mainly useful for real-time games; less relevant for turn-based

### Communication Protocols

**WebSocket:**
- Persistent bidirectional connection
- Low overhead after initial handshake
- Ideal for real-time updates
- Supported by all modern browsers
- Libraries: Socket.IO, ws, uWebSockets

**HTTP REST + Polling:**
- Stateless request/response
- Simple to implement
- Higher latency, more overhead
- Can work for slow turn-based games (chess-by-email style)
- Could be combined with Server-Sent Events for push updates

**WebRTC Data Channels:**
- Peer-to-peer data transfer
- Lowest latency possible
- Complex connection setup
- Primarily useful for real-time competitive games

### State Synchronization Strategies

**Full State Transfer:**
- Send entire game state on every change
- Simple to implement, no desync issues
- Bandwidth-heavy for large states
- Good for turn-based games with small state

**Delta/Diff Synchronization:**
- Only send changes since last sync
- More bandwidth-efficient
- More complex to implement
- Colyseus uses this approach automatically

**Event Sourcing:**
- Send only the actions/moves
- Clients replay actions to reconstruct state
- Very bandwidth-efficient
- Enables replay, undo, and time-travel debugging
- boardgame.io uses this approach
- Risk of desync if logic differs between client and server

### Lobby and Matchmaking Patterns

**Simple Room/Lobby:**
- Players create or join named rooms
- First player becomes host/creator
- Suitable for friend-based play

**Queue-Based Matchmaking:**
- Players enter a queue
- Server matches players by skill, latency, or preferences
- Better for competitive/ranked play

**Invitation System:**
- Share game links or invite specific users
- Works well for social/casual games

### Scaling Patterns

**Single Process:**
- All game rooms in one server process
- Simplest deployment
- Limited by single machine resources
- Fine for small-scale or hobby projects

**Horizontal Scaling:**
- Multiple server processes/containers
- Load balancer distributes rooms
- Sticky sessions needed for WebSocket
- Redis or similar for shared state between processes

**Serverless/FaaS:**
- Each move is a function invocation
- State stored in database
- No persistent connections (HTTP-based)
- Very cost-effective for low-traffic games
- Higher latency per action
