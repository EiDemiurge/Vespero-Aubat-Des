# Frontend Implementation Specification

> Authoritative reference for implementing the visual/interactive frontend layer.
> The frontend consumes the game logic API and must NOT reason about internal pipeline mechanics.

---

## 1. Architecture Boundary

```
 GAME LOGIC (modules 00-10)          FRONTEND (modules 11-12+)
 ─────────────────────────           ─────────────────────────
 Pure functions, no DOM              DOM, Canvas, Input, Audio

 GameController ──────────────────►  Renderer
   .dispatch(command)                  .playEvents(eventGroups)
   .castSpell(index, zone)             .onPhaseChange(phase, state)
   .getState() → GameState             .renderState(state, canvas)
```

The frontend implements a **Renderer** object and passes it to `createGameController(renderer)`.
The frontend also wires up UI controls that call `game.dispatch()` and `game.castSpell()`.

Game logic modules (00-10) must NEVER be modified for rendering changes.

---

## 2. Renderer Interface Contract

The renderer must expose exactly these three methods:

### 2.1 `playEvents(eventGroups)`

Called by the game controller after each pipeline execution.

**Parameter:** `eventGroups` — `Array<Array<GraphicEvent>>`

Structure: `[[parallel group 0], [parallel group 1], ...]`
- Each inner array is a set of events that play **simultaneously** (as parallel tweens/animations).
- Outer array elements play **sequentially** — group 1 starts only after group 0 finishes.

The renderer must process all groups in order, with appropriate delays/tweens between them.

### 2.2 `onPhaseChange(phase, state)`

Called when the game transitions between phases. This is the primary signal for UI mode changes.

**Parameters:**
- `phase` — One of: `'SETUP'`, `'COMBAT'`, `'AFTERMATH'`, `'SPELLCAST'`, `'ENDGAME'`
- `state` — The current `GameState` object (read-only reference; do not mutate)

### 2.3 `renderState(state, canvas)`

Called by `12_main.js` after every UI update to draw the current board state.

**Parameters:**
- `state` — Current `GameState`
- `canvas` — The `<canvas>` DOM element

---

## 3. Game Controller API (Frontend → Logic)

The frontend sends commands to the game via the controller object returned by `createGameController(renderer)`:

### 3.1 `dispatch(command)`

| Command     | When to Call                                | Effect                                |
|-------------|---------------------------------------------|---------------------------------------|
| `'START'`   | On page load, or when player clicks Restart | Creates new game, runs first round    |
| `'CONFIRM'` | Player accepts round result (AFTERMATH)     | Saves backup, runs next round         |
| `'REVERT'`  | Player wants to undo and cast a spell       | Restores backup, enters SPELLCAST     |
| `'FINISH'`  | (Internal — auto-dispatched by CONFIRM when `state.finished === true`) | Computes score, enters ENDGAME |

### 3.2 `castSpell(spellIndex, targetZone)`

| Parameter    | Type     | Values              |
|-------------|----------|---------------------|
| `spellIndex`| `number` | Index into `state.remainingSpells` array (0-based) |
| `targetZone`| `string` | `'ALLY'` or `'ENEMY'` |

Only callable when `state.phase === 'SPELLCAST'`. Applies the spell, re-runs the round, transitions to AFTERMATH.

### 3.3 `getState()` → `GameState`

Returns the current game state for rendering. Call after any dispatch/cast to get fresh state.

### 3.4 `getBackup()` → `GameState`

Returns the backup state (pre-round). Useful if the frontend needs to show a "before" comparison.

---

## 4. Game State Shape (Read-Only for Frontend)

```javascript
GameState = {
  roundNumber:     Number,        // Current round (starts at 0 during SETUP, incremented each round)
  coinStack:       Boolean[100],  // (Not relevant to rendering — internal use only)
  allyParty:       Party,
  enemyParty:      Party,
  remainingSpells: Spell[],       // Spells available to cast (shrinks as spells are used)
  phase:           String,        // 'SETUP'|'COMBAT'|'AFTERMATH'|'SPELLCAST'|'ENDGAME'
  finished:        Boolean,       // true = game over condition met
  stats:           Stats,
}

Party = {
  side:  String,     // 'ALLY' or 'ENEMY'
  units: Unit[],     // Up to 5 units (includes dead units — check .dead flag)
}
```

### 4.1 Unit Shape

```javascript
Unit = {
  // Identity
  id:               String,    // e.g. "ally_0", "enemy_2"
  name:             String,    // Display name: "Footman" or "Archer"
  party:            String,    // 'ALLY' or 'ENEMY'

  // Static combat stats (base values — do NOT use for display of effective values)
  attackTemplate:   String,    // 'MELEE_SINGLE' or 'MISSILE'
  attackPower:      Number,    // Base damage per hit
  nAttacks:         Number,    // Base number of attacks per turn
  toughness:        Number,    // Wound threshold = 2 + toughness
  speed:            Number,    // Determines action order
  drPhys:           Number,    // Base damage reduction

  // Dynamic (changes during gameplay)
  dead:             Boolean,   // true = unit is dead
  positionCellId:   Number,    // Current hex cell (0-11)
  protection:       Number,    // Remaining shield HP (starts from template value)
  wounds:           Number,    // Current wound count
  statusEffects:    Set<String>, // Active effects: 'EMPOWERED', 'WEAKENED', 'LOCKED'
}
```

### 4.2 Spell Shape

```javascript
Spell = {
  id:    String,                // e.g. "spell_1"
  name:  String,                // Display name: "Healing Light", "Displacement", etc.
  hexes: { [index]: Symbol },   // Map of spell-hex index (0-6) to symbol string
}
// Symbol values: 'HP', 'POS', 'LOCK', 'DMG'
```

### 4.3 Stats Shape

```javascript
Stats = {
  woundsReceived: Number,  // Total ally wounds taken
  woundsDealt:    Number,  // Total enemy wounds inflicted
  unitsKilled:    Number,  // Total enemy units killed
  unitsLost:      Number,  // Total ally units killed
  score:          Number,  // Final computed score (populated at ENDGAME)
  rating:         String,  // 'S', 'A', 'B', 'C', or 'D' (populated at ENDGAME)
}
```

**Score formula:** `score = (woundsReceived * -10) + (woundsDealt * 20) + (unitsKilled * 50) + (unitsLost * -100)`

**Rating thresholds:** S >= 300, A >= 200, B >= 100, C >= 0, D < 0

---

## 5. Graphic Event Catalog

Every graphic event has a `type` string and associated data fields. The renderer's `playEvents` method receives these.

### 5.1 Combat Events

| Type        | Data Fields                                       | Animation Guidance                     |
|-------------|---------------------------------------------------|----------------------------------------|
| `ATTACK`    | `{ attackerId, targetId }`                        | Show attacker lunging/shooting toward target |
| `BLOCKED`   | `{ targetId }`                                    | Shield flash / "blocked" indicator     |
| `HIT`       | `{ targetId, damage, protRemaining }`             | Impact effect, update protection bar   |
| `WOUNDED`   | `{ targetId, wounds, threshold }`                 | Blood effect, update wound counter     |
| `DEATH`     | `{ targetId }`                                    | Death animation, mark unit as dead     |
| `PASS`      | `{ attackerId }`                                  | Skip indicator (no valid target found) |

### 5.2 Spell Events

| Type        | Data Fields                                       | Animation Guidance                     |
|-------------|---------------------------------------------------|----------------------------------------|
| `SPELL`     | `{ spellId, targetZone }`                         | Spell cast overlay on target zone      |
| `HEAL`      | `{ targetId }`                                    | Green glow, reset wound display        |
| `MOVE`      | `{ unitId, fromCellId, toCellId }`                | Slide unit from old cell to new cell   |
| `SWAP`      | `{ unitAId, unitBId }`                            | Two units exchange positions           |
| `BUFF`      | `{ targetId, effect }`                            | Status icon appears; `effect` is `'EMPOWERED'`, `'WEAKENED'`, or `'LOCKED'` |

### 5.3 Round Events

| Type        | Data Fields                                       | Animation Guidance                     |
|-------------|---------------------------------------------------|----------------------------------------|
| `NEW_ROUND` | `{ roundNumber }`                                 | Round counter update, transition banner |

### 5.4 Event Sequencing Example

A typical attack sequence from `playEvents` looks like:

```javascript
[
  [{ type: 'ATTACK', attackerId: 'ally_0', targetId: 'enemy_0' }],      // Step 0: lunge
  [{ type: 'HIT', targetId: 'enemy_0', damage: 2, protRemaining: 12 }], // Step 1: impact
]
```

A lethal hit:
```javascript
[
  [{ type: 'ATTACK', attackerId: 'ally_0', targetId: 'enemy_2' }],
  [{ type: 'DEATH', targetId: 'enemy_2' }],
]
```

Parallel events (rare but possible):
```javascript
[
  [{ type: 'WOUNDED', targetId: 'enemy_0' }, { type: 'DEATH', targetId: 'enemy_1' }],
]
```

---

## 6. Phase-Driven UI Behavior

### 6.1 Phase Transition Table

| Phase       | Entered Via                     | UI State                                     |
|-------------|--------------------------------|----------------------------------------------|
| `SETUP`     | Internal only (boot)           | No user-facing UI for this phase             |
| `COMBAT`    | START, CONFIRM, castSpell      | Animations playing; all buttons disabled     |
| `AFTERMATH` | After round completes          | Show result; enable CONFIRM and REVERT       |
| `SPELLCAST` | REVERT command                 | Show spell selection panel; disable CONFIRM  |
| `ENDGAME`   | FINISH (auto after last round) | Show final score/rating; show Restart button |

### 6.2 Button Enable/Disable Rules

| Button   | Enabled When                                                           |
|----------|------------------------------------------------------------------------|
| CONFIRM  | `phase === 'AFTERMATH'`                                                |
| REVERT   | `phase === 'AFTERMATH' && state.remainingSpells.length > 0`           |
| Restart  | `phase === 'ENDGAME'` (visible only then)                             |
| Spell N  | `phase === 'SPELLCAST'` (one button per remaining spell)              |

### 6.3 SPELLCAST Flow (User Interaction)

1. Player clicks REVERT → phase becomes `SPELLCAST`
2. Frontend shows available spells (from `state.remainingSpells`)
3. Player selects a spell (index) **and a target zone** (`'ALLY'` or `'ENEMY'`)
4. Frontend calls `game.castSpell(spellIndex, targetZone)`
5. Phase transitions through COMBAT → AFTERMATH automatically
6. Frontend calls `game.getState()` and re-renders

---

## 7. Grid Layout — Rendering Geometry

### 7.1 Grid Structure

The battlefield is two overlapping pointy-top super-hexes sharing 2 middle cells. Total: 12 cells.

```
          ALLY SIDE                      ENEMY SIDE
     Back    Front   Middle   Front    Back
     col     col     col      col      col

TOP   [5]     [0]     [1]     [7]     [8]

               [6]                [11]

BOT   [4]     [3]     [2]    [10]     [9]
```

### 7.2 Cell ID Quick Reference

| Cell ID | Name               | Zone   | Row          | Vertical |
|---------|--------------------|--------|--------------|----------|
|  0      | Ally Front Top     | Ally   | Ally Front   | Top      |
|  1      | Middle Top         | Shared | Middle       | Top      |
|  2      | Middle Bottom      | Shared | Middle       | Bottom   |
|  3      | Ally Front Bottom  | Ally   | Ally Front   | Bottom   |
|  4      | Ally Back Bottom   | Ally   | Ally Back    | Bottom   |
|  5      | Ally Back Top      | Ally   | Ally Back    | Top      |
|  6      | Ally Front Center  | Ally   | Ally Front   | Center   |
|  7      | Enemy Front Top    | Enemy  | Enemy Front  | Top      |
|  8      | Enemy Back Top     | Enemy  | Enemy Back   | Top      |
|  9      | Enemy Back Bottom  | Enemy  | Enemy Back   | Bottom   |
| 10      | Enemy Front Bottom | Enemy  | Enemy Front  | Bottom   |
| 11      | Enemy Front Center | Enemy  | Enemy Front  | Center   |

### 7.3 Zone Membership

| Zone         | Cell IDs         |
|--------------|------------------|
| Ally (excl.) | 0, 3, 4, 5, 6   |
| Enemy (excl.)| 7, 8, 9, 10, 11 |
| Shared       | 1, 2             |

### 7.4 Position Mapping Guidance

The frontend must define pixel coordinates for all 12 cell centers. Cells 6 and 11 are the super-hex centers (vertically centered between the top and bottom rows of their respective side). The two super-hexes overlap at cells 1 and 2 (the middle column).

Suggested approach: define the 12 cell positions as a static lookup table of `{x, y}` pairs, then render hex tiles at each position. Use pointy-top hex orientation matching the mockup.

---

## 8. Content Requirements for Rendering

### 8.1 Unit Types and Visual Identity

| Unit Type | Attack Style | Icon/Visual Cue            |
|-----------|-------------|----------------------------|
| Footman   | Melee       | Sword icon (crossed swords in mockup) |
| Archer    | Missile     | Bow/arrow or crosshair icon |

Each unit on the board must display:
- **Unit icon** identifying its type (Footman vs Archer)
- **Party color** — Ally units tinted blue/cyan, Enemy units tinted red/pink (per mockup)
- **Death state** — Dead units show skull icon (as seen in mockup center hex)
- **Status effects** — Visual indicator for LOCKED, EMPOWERED, WEAKENED when active

### 8.2 Unit Stat Display (Hover Info / Detail Panel)

When a unit is hovered or selected, display:

| Stat              | Source Field       | Notes                                  |
|-------------------|--------------------|----------------------------------------|
| Name              | `unit.name`        | "Footman" or "Archer"                 |
| Attack Power      | `unit.attackPower` | Show base value; show effective if buffed |
| N of Attacks      | `unit.nAttacks`    | Show base; show 0 if LOCKED           |
| Protection        | `unit.protection`  | Current remaining (decreases in combat)|
| Wounds            | `unit.wounds`      | Current / threshold (threshold = 2 + toughness) |
| Speed             | `unit.speed`       | Static                                 |
| Damage Reduction  | `unit.drPhys`      | Show base; show +10 if LOCKED         |
| Attack Type       | `unit.attackTemplate` | "Melee" or "Missile"                |
| Status Effects    | `unit.statusEffects`  | List active effects                  |
| Dead              | `unit.dead`        | Show death state clearly              |

### 8.3 Spell Card Display

Each spell card in the spell selection panel must show:
- **Spell name** (`spell.name`)
- **Mini hex grid** — 7 hexes (indices 0-6) showing which hexes have symbols
- **Symbol icons** at populated hex positions:
  - `HP` — Heal (cross/heart icon)
  - `POS` — Move (arrow icon)
  - `LOCK` — Lock (lock/chain icon)
  - `DMG` — Damage (skull/explosion icon)
- Empty hex positions (no symbol) shown as blank/dimmed hexes

#### Default Spell Content

| Spell           | Hex 0 | Hex 1 | Hex 2 | Hex 3 | Hex 4 | Hex 5 | Hex 6 |
|-----------------|-------|-------|-------|-------|-------|-------|-------|
| Healing Light   | HP    | —     | HP    | —     | HP    | —     | —     |
| Displacement    | POS   | —     | POS   | —     | POS   | —     | —     |
| Binding Chains  | LOCK  | —     | LOCK  | —     | LOCK  | —     | —     |
| Arcane Blast    | DMG   | —     | DMG   | —     | DMG   | —     | —     |

### 8.4 Spell Overlay Visualization

When a spell targets a zone, the frontend should show which grid cells are affected.

**Spell hex index → Grid cell mapping:**

| Spell Index | Ally Zone Target | Enemy Zone Target |
|-------------|------------------|-------------------|
| 0           | Cell 0           | Cell 7            |
| 1           | Cell 1           | Cell 8            |
| 2           | Cell 2           | Cell 9            |
| 3           | Cell 3           | Cell 10           |
| 4           | Cell 4           | Cell 2 (shared!)  |
| 5           | Cell 5           | Cell 1 (shared!)  |
| 6           | Cell 6           | Cell 11           |

Key: When targeting enemy zone, spell indices 4 and 5 hit shared middle cells (2 and 1), potentially affecting ally units positioned there.

### 8.5 Default Game Setup (Initial Board)

**Ally Party (3 units, blue):**
| Cell | Unit    | ID      |
|------|---------|---------|
| 0    | Footman | ally_0  |
| 3    | Footman | ally_1  |
| 4    | Archer  | ally_2  |

**Enemy Party (5 units, red):**
| Cell | Unit    | ID       |
|------|---------|----------|
| 7    | Footman | enemy_0  |
| 10   | Footman | enemy_1  |
| 11   | Footman | enemy_2  |
| 8    | Archer  | enemy_3  |
| 9    | Archer  | enemy_4  |

---

## 9. UI Layout (From Mockup Analysis)

The mockup shows a landscape layout with these panels:

```
┌────────────┬──────────────────────────┬─────────────┐
│            │                          │             │
│  LOG       │     BATTLEFIELD          │  HOVER      │
│  PANEL     │     (hex grid)           │  INFO       │
│            │                          │  PANEL      │
│            │                          │             │
├────────────┼──────────────────────────┤             │
│ [CONFIRM]  │       STATUS:            │             │
│ [REVERT ]  │                          │             │
├────────────┴──────────────────────────┴─────────────┤
│  [1 Spell] [2 Spell] [3 Spell] [4 Spell] [5] [6]  │
│    (Spell cards row — each shows mini hex grid)     │
└─────────────────────────────────────────────────────┘
```

### 9.1 Panel Descriptions

| Panel          | Content                                                           |
|----------------|-------------------------------------------------------------------|
| **Log Panel**  | Scrollable text log of combat events (left side)                  |
| **Battlefield**| The 12-cell hex grid with units rendered on it (center)           |
| **Hover Info** | Detail panel showing stats of hovered/selected unit (right side)  |
| **Buttons**    | CONFIRM ("INDEED") and REVERT ("NEVER SO") below the log panel   |
| **Status Bar** | "STATUS:" area below the battlefield                              |
| **Spell Row**  | Bottom row: up to 6 spell cards, each showing a mini hex pattern  |

### 9.2 Visual Style (From Mockup)

- **Color palette:** Dark grey background, purple/magenta accent borders and highlights
- **Hex tiles:** Dark colored hexes with colored icons inside
- **Ally units:** Blue/cyan tinted icons on hex tiles
- **Enemy units:** Red/pink tinted icons on hex tiles
- **Dead units:** Skull symbol on darkened hex
- **Empty hexes:** Visible but dimmed/darker
- **Spell cards:** Grid of small circles (representing spell hex slots) with symbol icons
- **Button labels:** The mockup shows "INDEED" and "NEVER SO" as CONFIRM and REVERT respectively
- **Font style:** Pixel/retro style consistent with the overall aesthetic
- **Borders:** Prominent borders around each panel area

---

## 10. Animation Timing Guidance

The event groups from `playEvents` should be animated with appropriate timing:

| Event Type  | Suggested Duration | Notes                                    |
|-------------|-------------------|------------------------------------------|
| `NEW_ROUND` | 500-800ms         | Round transition banner                  |
| `ATTACK`    | 300-500ms         | Attacker moves toward target             |
| `BLOCKED`   | 200-300ms         | Shield flash                             |
| `HIT`       | 200-300ms         | Impact flash, protection bar update      |
| `WOUNDED`   | 300-500ms         | Blood splash, wound counter update       |
| `DEATH`     | 500-800ms         | Death animation, unit removed/greyed out |
| `HEAL`      | 300-500ms         | Green particles/glow                     |
| `MOVE`      | 400-600ms         | Unit slides to new position              |
| `SWAP`      | 400-600ms         | Two units slide past each other          |
| `BUFF`      | 300-400ms         | Status icon appears with flash           |
| `SPELL`     | 500-800ms         | Spell overlay/cast animation             |
| `PASS`      | 200-300ms         | Brief "no target" indicator              |

Between sequential groups (outer array): add a ~100-200ms gap.
Within a parallel group (inner array): all animations start at the same time.

---

## 11. Log Panel Content

The log panel should display a human-readable running commentary. Suggested format per event type:

| Event Type  | Log Text Template                                         |
|-------------|-----------------------------------------------------------|
| `NEW_ROUND` | `"═══ Round {roundNumber} ═══"`                           |
| `ATTACK`    | `"{attackerName} attacks {targetName}"`                   |
| `BLOCKED`   | `"Attack on {targetName} BLOCKED"`                        |
| `HIT`       | `"{targetName} hit for {damage} (prot: {protRemaining})"` |
| `WOUNDED`   | `"{targetName} WOUNDED ({wounds}/{threshold})"`           |
| `DEATH`     | `"{targetName} KILLED"`                                   |
| `HEAL`      | `"{targetName} healed"`                                   |
| `MOVE`      | `"{unitName} moved to {cellName}"`                        |
| `SWAP`      | `"{unitAName} swapped with {unitBName}"`                  |
| `PASS`      | `"{attackerName} has no target — PASS"`                   |
| `SPELL`     | `"Spell cast: {spellName} on {targetZone} zone"`          |
| `BUFF`      | `"{targetName} gained {effect}"`                          |

Note: Events contain unit IDs, not names. The renderer must resolve `unit.id` → `unit.name` from the current game state. If the unit is no longer findable (edge case), fall back to displaying the ID.

---

## 12. Endgame Screen

When `phase === 'ENDGAME'`, display:

- **Final Score:** `state.stats.score`
- **Rating:** `state.stats.rating` (S/A/B/C/D)
- **Breakdown:**
  - Wounds Dealt: `state.stats.woundsDealt` (x20 each = positive)
  - Units Killed: `state.stats.unitsKilled` (x50 each = positive)
  - Wounds Received: `state.stats.woundsReceived` (x-10 each = negative)
  - Units Lost: `state.stats.unitsLost` (x-100 each = negative)
- **Restart Button** visible

---

## 13. Required Clarifications

The following points are ambiguous in the current spec/mockup and need design decisions before frontend implementation:

### 13.1 Spell Target Zone Selection
**Current state:** `castSpell(index, zone)` requires a `targetZone` parameter ('ALLY' or 'ENEMY'), but the current `12_main.js` hardcodes `'ENEMY'`. The mockup shows no UI for zone selection.
**Needs answer:** How does the player choose which zone to target? Options:
  - (a) Two buttons per spell card (Ally / Enemy)
  - (b) A toggle or radio button for zone before casting
  - (c) Click the spell, then click a zone on the battlefield
  - (d) Some spells are always one zone (e.g., heal = ally, damage = enemy) — implicit from spell type

### 13.2 Spell Card Slots vs Remaining Spells
**Current state:** The mockup shows 6 numbered spell slots (1-6) in the bottom row, but the default content only has 4 spells. Some slots appear to show a mini hex grid of circles.
**Needs answer:**
  - Are the extra slots reserved for future spells, or should the UI always show exactly `remainingSpells.length` cards?
  - When a spell is consumed, does its slot disappear, grey out, or show empty?

### 13.3 Button Labels
**Current state:** The mockup shows "INDEED" and "NEVER SO" as button text.
**Needs answer:** Are these the final button labels, or should they be "CONFIRM" / "REVERT" (or something else)?

### 13.4 Status Bar Content
**Current state:** The mockup shows "STATUS:" text below the battlefield. No spec defines what goes here.
**Needs answer:** What information should the status bar display? Possibilities:
  - Current phase name
  - Current round number
  - Score-in-progress
  - "Waiting for player..." / "Combat in progress..." contextual text
  - All of the above

### 13.5 Hover Info Panel — Trigger and Content
**Current state:** The mockup shows a "HOVER INFO" panel on the right. The spec defines unit stats but doesn't specify interaction model.
**Needs answer:**
  - Does info show on hover (mouseover) or click/tap?
  - Can the player inspect enemy units too, or only allies?
  - Does hovering a spell card show spell details in this panel?
  - What is shown when nothing is hovered? (Empty panel, general game stats, tips?)

### 13.6 Animation vs Instant Resolution
**Current state:** The game logic runs the entire round synchronously and returns all events at once. The renderer receives the full event array after the round is complete.
**Needs answer:**
  - Should the frontend animate events one-by-one with delays (the round "plays out" visually)?
  - Or should it show the final state immediately and let the player scroll the log?
  - If animated: should there be a "skip animation" button?
  - Does the CONFIRM/REVERT button become enabled only after all animations finish?

### 13.7 Sound Design
**Current state:** The spec mentions "play a generic battle sound" on CONFIRM, but there is no sound system or sound asset list.
**Needs answer:**
  - Is sound in scope for the initial frontend?
  - If yes: what sounds are needed per event type?
  - Audio format preference? (Web Audio API, `<audio>` tags, Howler.js, etc.)

### 13.8 Responsive Layout / Target Resolution
**Current state:** Mockup appears to be ~640x360 or similar low-res pixel art style.
**Needs answer:**
  - Target canvas resolution?
  - Should the game scale to fit the browser window or use a fixed size?
  - Is pixel-perfect rendering important (nearest-neighbor scaling)?

### 13.9 Rendering Technology
**Current state:** The project mentions Phaser as a future possibility. The stub uses `canvas.getContext('2d')`.
**Needs answer:**
  - Should the frontend use Canvas 2D API directly?
  - Or integrate Phaser 3?
  - Or use plain DOM/CSS for layout with canvas only for the hex grid?

### 13.10 Unit Health Bar Visualization
**Current state:** Units have both Protection (shield) and Wounds. These are separate systems.
**Needs answer:**
  - Should Protection be shown as a numerical value, a bar, or both?
  - Should Wounds be shown as pips (dots), a fraction like "1/3", or a bar?
  - Are these shown on the hex tile directly, or only in the hover panel?

### 13.11 Dead Unit Display
**Current state:** The mockup shows a skull icon on one hex. Dead units remain in the `units` array with `dead: true`.
**Needs answer:**
  - Does the dead unit remain visible on its cell (greyed out with skull)?
  - Or does the cell become empty after a death animation?
  - Can dead units be hovered for info?

### 13.12 Middle Row Ownership Visualization
**Current state:** Cells 1 and 2 are "shared" — they belong to neither zone exclusively. Units can be moved there by spells.
**Needs answer:**
  - Should middle cells have a neutral color (neither blue nor red)?
  - When a unit occupies a middle cell, does the cell tint to that unit's party color?
  - Should spell targeting visualization highlight middle cells specially to show cross-zone bleed?

### 13.13 Combat Log History
**Needs answer:**
  - Does the log persist across rounds, or clear each round?
  - Is there a max log length / auto-scroll behavior?
  - Can the player scroll back through the log during AFTERMATH?

---

## Appendix A: Enum Values Reference

For convenience, all enum string values the frontend may encounter:

```
Phase:          SETUP, COMBAT, AFTERMATH, SPELLCAST, ENDGAME
Command:        START, CONFIRM, REVERT, FINISH
AttackTemplate: MELEE_SINGLE, MISSILE
StatusEffect:   EMPOWERED, WEAKENED, LOCKED
SpellSymbol:    HP, POS, LOCK, DMG
GraphicEvent:   ATTACK, BLOCKED, HIT, WOUNDED, DEATH, HEAL, MOVE, SWAP, BUFF, PASS, SPELL, NEW_ROUND
LogicEvent:     WOUND_DEALT, WOUND_RECEIVED, UNIT_KILLED, UNIT_LOST, BLOCKED, PASS
```

## Appendix B: Cell Name Lookup (for Log/Display)

```javascript
const CELL_NAMES = {
  0:  'Ally Front Top',      1:  'Middle Top',
  2:  'Middle Bottom',       3:  'Ally Front Bottom',
  4:  'Ally Back Bottom',    5:  'Ally Back Top',
  6:  'Ally Front Center',   7:  'Enemy Front Top',
  8:  'Enemy Back Top',      9:  'Enemy Back Bottom',
  10: 'Enemy Front Bottom',  11: 'Enemy Front Center',
};
```
