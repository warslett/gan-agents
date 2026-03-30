# WebRTC Data Channels for Multiplayer Games (MDN)

**Source:** https://developer.mozilla.org/en-US/docs/Games/Techniques/WebRTC_data_channels

## Extracted Information

### What WebRTC Data Channels Offer for Games
WebRTC data channels enable peer-to-peer communication between game clients, allowing players to send:
- Text chat messages
- Game status information
- Real-time game state updates

### Two Channel Types

**Reliable Channels:**
- Guarantees messages arrive at the peer in the same order sent
- Analogous to TCP socket behavior
- Use case: Critical game events, chat, scores

**Unreliable Channels:**
- No guarantees - messages may arrive out of order or not at all
- Analogous to UDP socket behavior
- Use case: Frequent position updates, non-critical real-time data (lossy data acceptable)

### Key Points
1. WebRTC is primarily known for audio/video, but data channels are equally valuable for games
2. The documentation recommends using libraries to trivialize implementation work, abstract browser differences, and handle cross-browser compatibility
3. References "WebRTC Data Channels for Great Multiplayer" from Mozilla Hacks as authoritative source material

### Architecture Implications

**Peer-to-Peer Model:**
- No central server needed for data transfer (reduces hosting costs)
- Lower latency since data goes directly between players
- But: harder to implement authoritative game state (no server to validate moves)
- Challenge: One peer must act as "host" or all peers must agree on state

**Hybrid Model:**
- Use a signaling server for connection establishment
- Use WebRTC for actual game data once connected
- Server can still validate moves but data flows P2P

### Relevance to Rummikub
- P2P architecture could reduce server costs for a tile game
- Reliable channels are sufficient for turn-based gameplay (no need for unreliable/fast channels)
- However, P2P makes AI opponents harder (AI would need to run on one client or a server)
- Harder to prevent cheating without an authoritative server
- Connection management is more complex than WebSocket-based approaches
- May not be ideal for Rummikub specifically due to the need for server-side AI and move validation
