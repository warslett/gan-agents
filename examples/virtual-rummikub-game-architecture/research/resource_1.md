# Rummikub - Game Rules and Mechanics (Wikipedia)

**Source:** https://en.wikipedia.org/wiki/Rummikub

## Extracted Information

### Basic Game Structure
Rummikub is a tile-based game for two to four players, combining elements of the card game rummy and mahjong. The game contains 106 tiles total: 104 numbered tiles (values 1-13 in four colors with two of each) plus two jokers.

### Players and Setup
- Each player begins with 14 tiles on individual racks (hidden from other players).
- Players draw tiles to determine who starts, with the highest value beginning play proceeding clockwise.
- Remaining tiles form the pool (draw pile).

### Core Gameplay Mechanics

**Initial Meld Requirement:**
The first player must play a set worth minimum 30 points before contributing to existing formations. If unable, they draw one tile from the pool.

**Valid Set Types:**
- **Runs**: Three or more same-colored consecutive tiles (e.g., 9-10-11-12)
- **Groups**: Three or four same-value tiles in different colors

**Invalid sets include:**
- Non-consecutive numbers in runs
- Mixed colors within runs
- Repeated colors in groups

### Advanced Mechanics - Tile Manipulation

Players may modify previously-placed tiles by:
- Shifting run endpoints and removing opposite ends
- Splitting long runs and inserting matching values
- Substituting group tiles with missing colors
- Removing end tiles from runs or single tiles from four-tile groups
- Replacing jokers with actual numbered tiles

**Joker Rules:**
Jokers cannot be retrieved before initial meld. They are worth 30 points and can be swapped out when the correct numbered tile becomes available.

### Scoring System
Losers sum remaining tile values in their racks. These scores subtract from cumulative totals, while the winner gains points equal to all opponents' scores combined.

### Winning Condition
A player declares "Rummikub" upon emptying their rack completely, becoming the winner.

### Variations
- **French Rules**: Include vertical group placement similar to Scrabble, enabling faster gameplay.
- **Draw Two/River variant**: Allows drawing two tiles while discarding one to a "River" row, with options to retrieve River tiles.

### Key Architectural Implications
- Game state must track: each player's rack (private), the table/board (public), and the draw pool (hidden count only).
- Tile manipulation is the most complex mechanic - a player can rearrange the entire board on their turn as long as all sets remain valid at the end.
- Turn validation must verify that all tiles on the table form valid sets, no tiles have disappeared, and any new tiles came from the active player's rack.
- The 30-point initial meld requirement adds a phase/state distinction per player.
