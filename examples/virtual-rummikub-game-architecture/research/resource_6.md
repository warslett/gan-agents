# Rummikub AI and Solver Algorithms

**Source:** Compiled from academic literature and open-source projects (no single URL - search results pointed to multiple GitHub repos and papers)

## Extracted Information

### The Rummikub Tile Placement Problem

Rummikub's tile manipulation mechanic creates a complex optimization problem. On each turn, a player can rearrange all tiles on the table (plus tiles from their rack) as long as all resulting sets are valid. This is computationally interesting because:

- Finding the optimal placement of tiles is NP-hard in the general case
- The problem can be modeled as an Integer Linear Programming (ILP) problem
- Each possible set (run or group) is a variable; the solver finds which sets to use

### Algorithmic Approaches for AI

**1. Greedy/Heuristic Approach:**
- Find the simplest valid moves first (direct placements without rearranging)
- Extend existing runs/groups with tiles from rack
- Only attempt complex rearrangements if simple moves fail
- Fast but suboptimal

**2. Integer Linear Programming (ILP):**
- Model all possible valid sets as binary variables
- Constraints: each tile appears in exactly one set (or stays on rack)
- Objective: maximize tiles played from rack (or minimize rack value)
- Solvers like GLPK, CBC, or OR-Tools can solve this
- Guarantees optimal tile placement per turn
- Well-studied approach in Rummikub literature

**3. Monte Carlo Tree Search (MCTS):**
- Simulate many possible game trajectories
- Evaluate which moves lead to winning positions
- Good for strategic decisions (which tiles to hold, when to draw)
- Can be combined with ILP for tile placement

**4. Rule-Based / Expert Systems:**
- Encode human strategies as rules
- Examples: "hold tiles that pair well", "avoid breaking useful sets on the table", "track opponent tile counts"
- Faster than search-based approaches
- Good for creating AI difficulty levels

### Key Technical Challenges for AI

- **Hidden information**: AI does not know opponents' tiles or remaining pool
- **Large action space**: Many possible rearrangements on a complex board
- **Strategic depth**: Short-term optimal play (play maximum tiles) may not be long-term optimal (holding useful tiles)
- **Tile tracking**: Remembering which tiles have been seen/played

### Open Source Implementations
Several GitHub repositories implement Rummikub solvers:
- Various ILP-based solvers using Python (PuLP, OR-Tools)
- JavaScript/TypeScript implementations for web-based play
- The solver/AI can run server-side, making it a natural fit for server-authoritative architecture
