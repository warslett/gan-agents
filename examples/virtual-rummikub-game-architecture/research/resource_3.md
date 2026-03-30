# Socket.IO - Real-Time Communication Library

**Source:** https://socket.io/get-started/chat

## Extracted Information

### What It Is
Socket.IO enables real-time, bidirectional communication between clients and servers. It is a lower-level library compared to full game frameworks like Colyseus or boardgame.io, providing the communication layer on which a game server can be built.

### Key Features

**Real-Time Push Capabilities:**
The framework allows servers to actively send messages to connected clients, rather than requiring clients to continuously poll for updates.

**Flexible Event System:**
Socket.IO operates on custom event emission. You can send and receive any events you want, with any data you want. Any objects that can be encoded as JSON will do, and binary data is supported too.

### Architecture

**Two-Part Structure:**
- **Server component**: Integrates with Node.js HTTP servers
- **Client library**: Loads in browsers (socket.io-client)

During development, the server automatically serves the client code.

### Communication Methods
- `io.emit()` - Sends events to all connected sockets
- `socket.broadcast.emit()` - Sends to everyone except the sender
- `socket.on()` - Listens for incoming events
- Rooms and namespaces for organizing connections into groups

### Additional Capabilities (from broader Socket.IO docs)
- **Rooms**: Built-in concept of rooms/channels that sockets can join and leave. Useful for game lobbies.
- **Namespaces**: Separate communication channels on the same connection.
- **Acknowledgements**: Callback-based confirmation that a message was received.
- **Fallback transport**: Automatically falls back from WebSocket to HTTP long-polling if needed.
- **Auto-reconnection**: Built-in reconnection logic.

### Relevance to Rummikub
- Provides the real-time communication layer needed for multiplayer
- Rooms concept maps directly to game tables/lobbies
- Event-based architecture suits turn-based game actions (play tiles, draw tile, end turn)
- More flexible than a full game framework but requires building game logic from scratch
- Very mature and widely used library with large ecosystem
- Would need to implement state synchronization, matchmaking, and game logic manually
