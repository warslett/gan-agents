# Colyseus - Multiplayer Game Framework

**Source:** https://docs.colyseus.io/

## Extracted Information

### What It Is
Colyseus is an open-source Node.js framework for building authoritative game servers with real-time state synchronization, matchmaking, and effortless integration into any game engine or frontend.

### Key Features

1. **Rooms and Matchmaking** - From a single Room definition, clients are matched into multiple Room instances. Rooms are the primary organizational unit for games.

2. **State Synchronization** - Define your state on the server, and it is automatically synchronized to all connected clients. Uses a Schema-based system with TypeScript decorators. Employs automatic delta synchronization (only changes are sent).

3. **Scalability** - Built for horizontal and vertical scalability across multiple processes.

4. **Flexible Deployment** - Can be self-hosted or deployed on Colyseus Cloud for managed hosting.

### Architecture Patterns

**Server-Side Structure:**
- Rooms serve as the primary organizational unit
- Room state uses a Schema-based system with TypeScript decorators
- Message-driven communication between clients and server
- Lifecycle hooks: onCreate, onJoin, onLeave, onDispose

**State Management:**
Centralized server state model using Schema definitions with type annotations, enabling automatic delta synchronization to clients.

### Supported Client Platforms
- TypeScript/JavaScript (ideal for web-based games)
- Unity (C#)
- Defold (Lua)
- Haxe
- GameMaker, Godot, Construct 3, MonoGame, Cocos Creator
- Discord Activities and WeChat integration

### Turn-Based Gaming Support
The framework includes a "Turn Based Tanks Demo" in tutorials, suggesting native support for turn-based game architectures.

### Relevance to Rummikub
- Authoritative server model is ideal for preventing cheating in competitive games
- Room-based architecture naturally maps to game lobbies/tables
- State synchronization handles broadcasting board state to all players
- Matchmaking handles connecting players who want to play together
- Schema-based state would work well for representing tiles, racks, and board state
- JavaScript/TypeScript stack means same language for client and server
