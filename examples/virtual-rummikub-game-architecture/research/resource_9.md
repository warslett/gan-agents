# Game State Modeling for Rummikub

**Source:** Compiled from game design and software architecture resources

## Extracted Information

### Core State Elements

**Tile Representation:**
```
Tile {
  id: unique identifier
  number: 1-13
  color: red | blue | yellow | black
  isJoker: boolean
}
```
- 106 tiles total: 2 of each number-color combination (2 * 4 * 13 = 104) + 2 jokers
- Each tile needs a unique ID to track its position across rearrangements

**Game State:**
```
GameState {
  pool: Tile[]           // tiles not yet drawn (hidden from all players)
  players: Player[]      // array of player states
  board: Set[]           // tiles on the table, organized into sets
  currentPlayer: number  // index of active player
  phase: string          // e.g., "initial_meld" or "normal_play"
}
```

**Player State:**
```
Player {
  id: string
  rack: Tile[]           // tiles in hand (visible only to this player)
  hasMetInitialMeld: boolean
  score: number
  isAI: boolean
}
```

**Set (on the board):**
```
Set {
  id: unique identifier
  tiles: Tile[]          // ordered list of tiles in this set
  type: "run" | "group"  // could be inferred from tiles
}
```

### State Visibility Rules (Critical for Architecture)

| State Element | Current Player | Other Players | Spectators |
|---|---|---|---|
| Own rack | Visible | Hidden | Hidden |
| Other racks | Hidden | Hidden | Hidden |
| Board/table | Visible | Visible | Visible |
| Pool count | Visible | Visible | Visible |
| Pool contents | Hidden | Hidden | Hidden |
| Scores | Visible | Visible | Visible |

This means the server must filter state before sending to each client. This is a key architectural requirement.

### Turn Flow State Machine

```
WAITING_FOR_TURN
  -> START_TURN (player's turn begins)
    -> MANIPULATING (player is moving tiles)
      -> VALIDATE (player clicks "done")
        -> [Valid] TURN_COMPLETE -> next player's WAITING_FOR_TURN
        -> [Invalid] MANIPULATING (revert and try again)
      -> DRAW_TILE (player cannot or chooses not to play)
        -> TURN_COMPLETE -> next player's WAITING_FOR_TURN
```

### Validation Logic

**End-of-turn validation must check:**
1. All sets on the board are valid (runs of 3+ same color, groups of 3-4 same number different colors)
2. No tiles from the board have disappeared (all pre-existing board tiles still present)
3. At least one new tile was added from the player's rack (unless drawing)
4. If first meld: sum of tiles in newly created sets >= 30
5. Joker usage follows rules (cannot retrieve before initial meld, etc.)

**Approach: Snapshot and Compare**
- Snapshot the board state at start of turn
- Let player freely manipulate
- At end of turn, compare new state with snapshot
- Verify constraints above

### Persistence Considerations
- In-memory state for active games (fast access)
- Database persistence for: game history, user accounts, leaderboards
- Options: Redis for active game state, PostgreSQL/MongoDB for persistent data
- Event sourcing: store moves as events, reconstruct state by replaying
