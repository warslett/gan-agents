# boardgame.io - Framework for Turn-Based Games

**Source:** https://boardgame.io/documentation/#/

## Extracted Information

Note: The documentation page rendered with limited content. The following is compiled from the framework's well-known public documentation and GitHub repository.

### What It Is
boardgame.io is an open-source framework specifically designed for building turn-based multiplayer board games. It is built on top of React (for UI) and uses a Redux-like state management approach. The server component runs on Node.js.

### Core Concepts

**Game Definition:**
Games are defined as plain JavaScript objects specifying:
- `setup`: Function returning the initial game state (G)
- `moves`: Object mapping move names to functions that modify G
- `endIf`: Function that checks if the game is over
- `turn`/`phases`: Configuration for turn order and game phases

**State Management:**
- Game state (G) is a plain JavaScript object
- Moves are pure functions: (G, ctx, ...args) => newG
- State transitions are logged, enabling undo/redo and time-travel debugging
- All state changes go through the framework, enabling automatic synchronization

### Multiplayer Architecture
- Client-server architecture with authoritative server
- Server validates all moves
- Automatic state synchronization to all clients
- Supports both real-time (WebSocket via Socket.IO) and asynchronous play
- Lobby system for creating and joining games

### AI / Bot Support
- Built-in bot framework
- `RandomBot`: Makes random valid moves
- `MCTSBot`: Monte Carlo Tree Search AI that explores game trees
- Bots run as simulated players, using the same move interface as human players
- AI can be run server-side or client-side

### Phases and Turn Order
- Games can be split into phases with different rules
- Configurable turn order (round-robin, custom, etc.)
- `endTurn`, `endPhase` helpers
- Stages allow multiple players to act simultaneously within a turn

### Additional Features
- Secret state: Different players can see different parts of the state
- Logging and replays
- Plugin system for extending functionality

### Relevance to Rummikub
- Purpose-built for exactly this kind of game
- Turn-based architecture matches Rummikub's gameplay
- Phases could represent: initial meld phase vs. normal play
- Secret state support is essential (each player's rack is private)
- Built-in AI (MCTS) could serve as the AI opponent
- Move validation on server prevents cheating
- Undo support could help with tile manipulation (try arrangements, undo if invalid)
- React integration simplifies building the drag-and-drop tile UI
