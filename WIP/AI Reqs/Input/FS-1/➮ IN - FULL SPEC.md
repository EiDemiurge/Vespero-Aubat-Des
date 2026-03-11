Turn-based auto-battler on a tiny hex grid with player casting spells to change round outcomes. 
## 1. Finite State Machine

### 1.1 States

| State     | Description                             |
| --------- | --------------------------------------- |
| SETUP     | Initialize game from launch data        |
| COMBAT    | Main round loop executes                |
| AFTERMATH | Round resolved; awaiting player command |
| SPELLCAST | Player is applying a spell              |
| ENDGAME   | Final score display                     |
*For clarity, prefix S: can be applied to these names, e.g. S:COMBAT*

### 1.2 Commands

| Command | Behavior                                                                                                                                                                                                               |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| START   | Issued on game launch or when the player hits Restart from the End Game screen.                                                                                                                                        |
| CONFIRM | Overwrites the backup state with current state. Proceeds with the Round Loop to S:COMBAT, yielding the next game state and setting game phase to S:AFTERMATH. If the `finished` flag is true, issues C:FINISH instead. |
| REVERT  | Transitions to SPELLCAST state.                                                                                                                                                                                        |
| FINISH  | Transitions to ENDGAME state, passing Stats data.                                                                                                                                                                      |
*For clarity, prefix C: can be applied to these names, e.g. C:CONFIRM* 

---

### 1.3 Transitions  

| State     | Transitions To             |
| --------- | -------------------------- |
| SETUP     | COMBAT                     |
| COMBAT    | AFTERMATH                  |
| AFTERMATH | SPELLCAST, COMBAT, ENDGAME |
| SPELLCAST | COMBAT                     |
| ENDGAME   | SETUP                      |

---
## 2. Data Models

### 2.1 Party
A map of up to 5 units to cells on a 5-hexagon grid. Each unit is mapped to one or more unique hex cells without overlap.

### 2.2 Unit
A data map of static and dynamic values. 
(^ marks dynamic values that can be changed directly by pipelines)

| Name: [String]
| Attack Template: [Enum Container]
| Attack Power: [int]
| N of Attacks  : [int] 
| Toughness     : [int] 
| Speed             : [int]
| DR-Phys        : [int]

| ^Dead: [Boolean]
| ^Position Cell ID: [int] 
| ^Protection     : [int]
| ^Wounds         : [int]
| ^Status Effects: [Enum Container]

*Status Effects Enum: [Empowered, Weakened, Locked]*
*Attack Template Enum: [Melee Single, Missile*] 
### 2.3 Spell
A super-hex mini-grid of 7 hexes. Each hex has a Symbol mapped to a unique coordinate. Symbols are Enum consts: [HP, Pos, Lock, Dmg]

### 2.4 Game State

| Field            | Type / Shape              |
| ---------------- | ------------------------- |
| Round Number     | Integer                   |
| Coin Stack       | 100 Booleans              |
| Player Party     | Party                     |
| Enemy Party      | Party                     |
| Remaining Spells | List of Spell             |
| State Phase      | State Enum, e.g. S_COMBAT |
| Stats            | Dynamic object            |


### 2.5 Pipeline Result

| Field          | Description                                      |
| -------------- | ------------------------------------------------ |
| Game State     | New state to replace the previous one            |
| Pipeline Type  | Enum identifier (for debugging)                  |
| Graphic Events | Nested list with parallel groups (for animation) |
| Logic Events   | Flat list (for logging / Stats)                  |


### [2.6] Stats 

When a pipeline returns the Result, the [Result Processor] looks for these Logic Events to use for creating a new modified copy of Game State's Stats data object and pass it to the Game State that will be set as current. 

Aggregated in Stats:  ( `Event (Score delta multiplier)` )
	Wounds received (-10)
	Wounds dealt (20) 
	Units killed (50)
	Units lost (-100)


---

## 3. Pipeline Architecture

All game logic is processed via **Pipeline functions**. These are idempotent, operate on a read-only copy of Game State, and return a Result containing: a new Game State, Graphic Events, and Logic Events.

Results are processed in two ways:
1. **Game State** is applied immediately.
2. **Graphic + Logic Events** propagate upward and are batched for animation and logging on the frontend.

Event sequences from a Pipeline are transformed into a matching sequence of tween animations. Visualization is optional per pipeline.

---

## 4. Combat Loop Structure

### Nesting Overview

```
Round Loop (L1)
 └─ Party Turn (L2)  ×2
     └─ Unit Attack (L3)  per alive/unlocked unit
         └─ Target Acquisition + Damage Pipeline  per attack count
```

### 4.1 L1 — Round Loop

The main outer loop. On exit, execute the Endgame Pipeline.

| Step | Action |
|------|--------|
| 1    | Run **New Round Pipeline** (clean-up, refresh dynamic values). |
| 2    | Determine party turn order (party with the fastest unit goes first). |
| 3    | Execute L2 for the first-acting party. |
| 4    | Execute L2 for the second-acting party. |
| 5    | Log aggregate round result. |
| 6    | Wait for player command: **CONFIRM** or **REVERT**. |

**On CONFIRM:** Wait 1 second, play a generic battle sound, proceed to next round — unless exit condition is met (a party has no living units).

**On REVERT:** Restore the backup Game State. Unblock the Spell Panel. Wait for numeric input to select a spell. *(This is effectively "step 0" of the Round Loop.)*

**On Spell Applied:** Run the Apply Spell Pipeline, then restart the Round Loop from step 1.

### 4.2 L2 — Party Turn Loop

For each unit that is not DEAD, perform its action. The default action is Attack (enter L3).

### 4.3 L3 — Unit Attack Loop

A counted loop where `i` runs from 1 to the unit's `N of Attacks` parameter.

Each iteration:
1. Run **Find Target Pipeline** using the attacker's Attack Template to acquire the next unique target. If none found → exit with a PASS result event.
2. Run **Damage Pipeline** for the acquired target. Return to step 1.

---

## 5. Pipeline Definitions

Below are functions that operate on a given Game State and return a new modified one without modifying anything else (idempotent).
### 5.1 Setup Pipeline

Given launch data (Party 1, Party 2, Spells), create the initial Game State. If no data is provided, use default data defined in code (see Content FS-1).

### 5.2 New Round Pipeline

Takes the previous round's game state and returns one for the next round after applying following modifications:
1. Re-Initialize the Coin Stack with 100 new random booleans.
2. Increment Round Number.
3. Iterate over all units and remove temporary status effects. 


### 5.3 Find Target Pipeline

Given an [Attack Template] and a source unit reference, find the first valid target not yet targeted during this attack's processing. Runs in a nested while loop until the result set is empty, then exits. *(Example: Melee Wide may find 1, 2, or 3 targets sequentially.)*

Particular algorithms for acquiring targets are in [Content Details] section.

### 5.4 Damage Pipeline

| Step | Action                                                            | Exit Condition                                       |
| ---- | ----------------------------------------------------------------- | ---------------------------------------------------- |
| 1    | Subtract target's Damage Reduction from incoming damage.          | If result ≤ 0 → exit with **BLOCKED** event.         |
| 2    | Subtract remaining damage from target's Protection.               | If Protection remains > 0 → exit with **HIT** event. |
| 3    | Inflict a Wound. Check target's total Wounds vs. (2 + Toughness). | If within limit → exit with **WOUNDED** event.       |
| 4    | Kill the unit (set DEAD to true).                                 | Exit with **DEATH** event.                           |

### 5.5 Apply Spell Pipeline

Apply the spell's super-hex grid directly onto the target zone (Ally or Enemy, and always the 2 Middle Cells). Each symbol's coordinate maps to a cell in the zone. For each cell containing a unit, apply that symbol's FX.
Concrete logic for symbols is defined in Content Details. 

Note: since the two hex cells between the original party positions are available to both sides, a spell may in theory afflict some allies and some enemies at the same time. 

Each FX being applied may run its own Pipeline, so we must aggregate its Result to pass it upward to the Result Processor. 

At the end, run Apply Buffs Pipeline.
### 5.6 Swap Positions Pipeline

For two given units, swap their Position values. 

### 5.7 Apply Buffs Pipeline

For all units, iterate over their Status Effects set and apply modifications. 

Currently supported: 

LOCK status - Unit's N of Attacks is set to 0 and Damage Reduction to 10.
EMPOWER status - Unit's  Attack Power is doubled.
WEAKEN status - Unit's  Attack Power is halved, rounding down.

### 5.7 Endgame Pipeline

Computes final score and picks matching rating, setting them in the Stats data object. Then sets GameState's Phase to S:ENDGAME and returns. 

May produce animation effects for transition to EndGameScreen. 

### 5.8 Game End Check Pipeline

Checks if either party has no units that are alive (Dead == false) 
Returns a copy of the game state with FINISHED flag set to true.



---

# Content Details 

#### Combat Grid 
 
 Grid is composed of 2 super-hexes overlapping horizontally (Ally Zone and Enemy Zone respectively). Each has 5 cells exclusive to it and 2 shared cells in the middle, with total of 5+5+2=12 cells. 

Being tiny, the Grid it does not use conventional hex coordinate space. 
Instead, each cell has an ID (from 0 to 11) assigned using this algorithm for incrementing: 
Starting with the left super-hex, we pick the top hex and move clockwise until we traversed all hexes (next hex already has id assigned), then pick the center hex next. Same for the right super-hex. 

Following are static mappings of Cell IDs to facilitate geometric operations and target-acquisition: 

ID sets for Rows: 
(for Bottom to Top order, reverse for Top to Bottom)
Ally Back =  [4, 5]
Ally Front = [3, 6, 0]
Middle =      [2, 1]
Enemy Back =  [9, 8]
Enemy Front = [10, 11, 7]

Cell names to id: 
	Ally Back Bottom = [4] 
	Ally Back Top = [5] 
	Ally Front Bottom = [3] 
	Ally Front Center = [6] 
	Ally Front Top = [0] 
	Middle Bottom = [2] 
	Middle Top = [1] 
	Enemy Front Bottom = [10] 
	Enemy Front Center = [11] 
	Enemy Front Top = [7] 
	Enemy Back Bottom = [9] 
	Enemy Back Top = [8]


#### FX definitions 
*FX = One-shot Effects from Symbols* 

##### Heal FX 

Sets target's Wounds parameter to 0. 
##### Wound FX 

Uses parts of the Damage Pipeline, bypassing Damage Reduction, immediately increments Wounds parameter by X (default = 1) and runs the same check for DEATH. 
Returned Result has either WOUNDED or DEATH Logic/Graphic event pair.

##### Move FX 
Target must move to another cell in the same row or in the Middle Row. Prefer free cells in the same row as target. 
Pop a bool from Coin Stack if must decide between two cells from the same row. 
If there are no free cells, run the same search for cells occupied by single-cell units from the same party as FX target with same prioritization rules. 
If a valid unit is found, run Swap Positions pipeline for these two. 

##### Status Effects
Weaken, Empower and Locked are status effects that a unit may receive from an FX. They are cleared at the end of the round. 

LOCK status - Unit is unable to act, but takes 10 less damage from Attacks. 
EMPOWER status - Unit's effective Attack Power is doubled.
WEAKEN status - Unit's effective Attack Power is halved, rounding down.

Rule: Weaken and Empower are mutually exclusive - one overrides the other, so only the one applied last can ever remain. 


---
#### Attack Templates 

##### Melee
*Checks row by row to find the closest enemy in the closest row.*

Protocol: 
In each row, check indices sequentially to find the first enemy unit.

Define search direction based on source position: 
Bottom => Bottom to Top
Top => Top to Bottom

*If source is on the center-cell of the Front Row (id 6 or 11), pop a boolean from Coin Stack to determine the search direction.*

##### Missile 
*Hits the first enemy target in its horizontal flight path*

Protocol: 
Creates a sorted set of cells for the missile’s flight path.
Loops through them, if there is a unit on current cell, acquires them and returns. 

4 cells in horizontal line from source - 1 Middle and 3 Enemy cells.
Example - for Source at cell [4], the flight path is [2, 10, 11, 9]

Similar to Melee, uses Coin Stack to define vertical search direction, which determines which opposite front-row cell is checked first. 

--- 

### Default Data Preset:
#### Ally Party:
[0] = FOOTMAN
[3] = FOOTMAN
[4] = ARCHER

#### Enemy Party:
[7] = FOOTMAN
[10] = FOOTMAN
[11] = FOOTMAN
[8] = ARCHER
[9] = ARCHER

#### Spells:
Spell_1:
	[0] = HP
	[2] = HP
	[4] = HP

Spell_2:
	[0] = MOVE
	[2] = MOVE
	[4] = MOVE

Spell_3:
	[0] = LOCK
	[2] = LOCK
	[4] = LOCK

Spell_3:
	[0] = DMG
	[2] = DMG
	[4] = DMG

### FS-1 Unit Content Data 

##### FOOTMAN
| Attack Type  : [Melee Single]
| Attack Power: [3]
| N of Attacks  : [2] 

| Protection     : [12]
| Toughness     : [1] 
| Speed             : [3]
| Damage Reduction        : [1]

##### ARCHER
| Attack Type  : [Missile]
| Attack Power: [5]
| N of Attacks  : [1] 

| Protection     : [8]
| Toughness     : [0] 
| Speed             : [4]
| Damage Reduction       : [0]