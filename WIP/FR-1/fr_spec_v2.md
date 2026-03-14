# Frontend Implementation Specification

> Authoritative reference for AI implementors building the Phaser-based frontend for the hex auto-battler. This document covers everything needed to render, animate, and wire UI to the existing game logic — without reasoning about logic internals.

---

## 1. Technology & Constraints

- **Renderer:** Phaser 3 (loaded via CDN `<script>` tag)
- **Output:** Single `.html` file — all JS inline, no build tools, no npm, no TypeScript
- **Assets:** NONE. All visuals are vector graphics (Phaser shapes) and colored Unicode symbols rendered as text
- **Game logic:** Already implemented in `src/00_enums.js` through `src/10_state_machine.js`. The frontend **must not modify** modules 00–10. Only `src/11_renderer_stub.js` and `src/12_main.js` are frontend territory.

---

## 2. Game Controller API

The frontend interacts with game logic **exclusively** through the `GameController` object returned by `createGameController(renderer)`.

### 2.1 Controller Methods

| Method | Signature | When to Call |
|--------|-----------|-------------|
| `dispatch` | `(command: Command, payload?) → void` | Send FSM commands |
| `castSpell` | `(spellIndex: number, targetZone: 'ALLY'\|'ENEMY') → void` | During SPELLCAST phase only |
| `getState` | `() → GameState` | Read current state at any time |
| `getBackup` | `() → GameState` | Read backup state (pre-round) |

### 2.2 Commands (the `Command` enum)

| Command | When Available | Effect |
|---------|---------------|--------|
| `Command.START` | ENDGAME or initial boot | Creates default game, runs first round, lands in AFTERMATH |
| `Command.CONFIRM` | AFTERMATH | If `state.finished` → triggers FINISH. Otherwise saves backup, runs next round, lands in AFTERMATH |
| `Command.REVERT` | AFTERMATH | Restores backup state, enters SPELLCAST phase |
| `Command.FINISH` | (auto-dispatched by CONFIRM when `finished=true`) | Runs endgame pipeline, enters ENDGAME phase |

### 2.3 Spell Casting

```
game.castSpell(spellIndex, targetZone)
```
- Only callable when `state.phase === Phase.SPELLCAST`
- `spellIndex`: index into `state.remainingSpells[]`
- `targetZone`: `'ALLY'` or `'ENEMY'` — determines which zone the spell overlays onto
- After casting: the spell is removed from `remainingSpells`, a full round executes, phase returns to AFTERMATH

### 2.4 Revert Button Guard

The REVERT button should be **disabled** when `state.remainingSpells.length === 0` (no spells left to cast — reverting would be pointless).

---

## 3. Renderer Interface Contract

The renderer object passed to `createGameController()` must implement:

### 3.1 `playEvents(eventGroups)`

Called by the controller after each pipeline. Receives the graphic events to animate.

**Parameter:** `eventGroups` — `Array<Array<GraphicEvent>>`

Structure: outer array = sequential steps. Inner array = events that play **in parallel** within that step. The renderer must play step 0 fully, then step 1, etc.

```
eventGroups = [
  [event, event],   // step 0: these two animate simultaneously
  [event],          // step 1: plays after step 0 completes
  [event, event, event],  // step 2: three simultaneous animations
]
```

### 3.2 `onPhaseChange(phase, state)`

Called when the game phase transitions. Use this to update UI chrome (enable/disable buttons, show panels, update status text).

**Parameters:**
- `phase`: one of `Phase.COMBAT`, `Phase.AFTERMATH`, `Phase.SPELLCAST`, `Phase.ENDGAME`
- `state`: the current `GameState` — read-only snapshot for rendering

### 3.3 `renderState(state, canvas)` *(optional)*

Called to do a full re-render of the board. Use for initial draw and after any state update.

---

## 4. Game State Shape (Read-Only for Frontend)

The frontend reads `game.getState()` to render. Here is the complete shape:

```javascript
{
  roundNumber: Number,       // current round (starts at 0, incremented each round)
  coinStack: Boolean[],      // internal — frontend ignores
  allyParty: {
    side: 'ALLY',
    units: [Unit, ...]       // up to 5
  },
  enemyParty: {
    side: 'ENEMY',
    units: [Unit, ...]       // up to 5
  },
  remainingSpells: [Spell, ...],  // shrinks as spells are used
  phase: String,             // Phase enum value
  finished: Boolean,         // true when a party is wiped
  stats: {
    woundsReceived: Number,  // ally wounds taken
    woundsDealt: Number,     // enemy wounds inflicted
    unitsKilled: Number,     // enemy units killed
    unitsLost: Number,       // ally units lost
    score: Number,           // final computed score (only meaningful in ENDGAME)
    rating: String,          // 'S'|'A'|'B'|'C'|'D' (only meaningful in ENDGAME)
  }
}
```

### 4.1 Unit Shape

```javascript
{
  id: String,               // e.g. "ally_0", "enemy_2"
  name: String,             // "Footman" | "Archer"
  party: String,            // "ALLY" | "ENEMY"
  attackTemplate: String,   // "MELEE_SINGLE" | "MISSILE"
  attackPower: Number,      // base value
  nAttacks: Number,         // base value
  toughness: Number,        // wound threshold = 2 + toughness
  speed: Number,
  drPhys: Number,           // base Damage Reduction

  // Dynamic — these change during gameplay:
  dead: Boolean,
  positionCellId: Number,   // 0–11, maps to grid cell
  protection: Number,       // remaining shield HP
  wounds: Number,           // current wound count
  statusEffects: Set<String> // Set of StatusEffect values: 'EMPOWERED'|'WEAKENED'|'LOCKED'
}
```

**Effective stats** (after buffs) — use the helper functions:
- `getEffectiveAttackPower(unit)` — doubled if EMPOWERED, halved if WEAKENED
- `getEffectiveNAttacks(unit)` — 0 if LOCKED
- `getEffectiveDR(unit)` — base + 10 if LOCKED
- `getWoundThreshold(unit)` — `2 + unit.toughness`

### 4.2 Spell Shape

```javascript
{
  id: String,                // e.g. "spell_1"
  name: String,              // e.g. "Healing Light"
  hexes: {                   // sparse map: spell-hex-index → SpellSymbol
    [index: Number]: String  // index 0–6, value is SpellSymbol enum
  }
}
```

**SpellSymbol enum values:** `'HP'`, `'POS'`, `'LOCK'`, `'DMG'`

### 4.3 Phase Enum Values

| Value | Meaning |
|-------|---------|
| `'SETUP'` | Initializing (transient — frontend won't see this) |
| `'COMBAT'` | Round is executing (transient — logic runs synchronously) |
| `'AFTERMATH'` | Round resolved, awaiting player input |
| `'SPELLCAST'` | Player is choosing a spell to cast |
| `'ENDGAME'` | Game over, showing final score |

---

## 5. Graphic Event Catalog

Every event the renderer receives has a `type` field and additional data fields. This is the **complete** list.

### 5.1 Combat Events

#### `NEW_ROUND`
```javascript
{ type: 'NEW_ROUND', roundNumber: Number }
```
Signals the start of a new round. Use for round title animation / status update.

#### `ATTACK`
```javascript
{ type: 'ATTACK', attackerId: String, targetId: String }
```
An attack is being initiated. Animate the attacker's approach.

#### `BLOCKED`
```javascript
{ type: 'BLOCKED', targetId: String }
```
Attack fully absorbed by Damage Reduction. Defensive bounce animation.

#### `HIT`
```javascript
{ type: 'HIT', targetId: String, damage: Number, protRemaining: Number }
```
Protection absorbed damage. Show impact but no wound. `protRemaining` is the new protection value.

#### `WOUNDED`
```javascript
{ type: 'WOUNDED', targetId: String, wounds: Number, threshold: Number }
```
Target took wound(s). `wounds` = current total, `threshold` = max before death.

#### `DEATH`
```javascript
{ type: 'DEATH', targetId: String }
```
Target killed. Replaces WOUNDED when wound threshold exceeded.

#### `PASS`
```javascript
{ type: 'PASS', attackerId: String }
```
Attacker found no valid target. Skip animation or show "pass" indicator.

### 5.2 Spell Events

#### `SPELL`
```javascript
{ type: 'SPELL', spellId: String, targetZone: 'ALLY'|'ENEMY' }
```
A spell is being cast. Show spell cast animation before individual FX.

#### `HEAL`
```javascript
{ type: 'HEAL', targetId: String }
```
Unit's wounds set to 0.

#### `MOVE`
```javascript
{ type: 'MOVE', unitId: String, fromCellId: Number, toCellId: Number }
```
Unit relocated to a different cell.

#### `SWAP`
```javascript
{ type: 'SWAP', unitAId: String, unitBId: String }
```
Two units exchanged positions.

#### `BUFF`
```javascript
{ type: 'BUFF', targetId: String, effect: 'EMPOWERED'|'WEAKENED'|'LOCKED' }
```
Status effect applied/active on a unit.

---

## 6. Grid Geometry — Rendering the Hex Board

### 6.1 Grid Structure

The board is **12 hex cells** arranged as 2 overlapping super-hexes (flat-topped hexagons drawn with `add.polygon(6)`):

```
          ALLY SIDE                      ENEMY SIDE
     Back    Front   Middle   Front    Back

TOP   [5]     [0]     [1]     [7]     [8]

               [6]                [11]

BOT   [4]     [3]     [2]    [10]     [9]
```

- Cells 6 and 11 are **center cells** of the ally/enemy super-hexes
- Cells 1 and 2 are the **shared Middle cells** (both zones overlap here)
- All other cells are rim cells

### 6.2 Cell Metadata for Positioning

The renderer needs to place each hex. Use these static lookups to determine zone coloring and layout:

| Cell ID | Zone | Row | Vertical Position |
|---------|------|-----|-------------------|
| 0 | ALLY | ALLY_FRONT | top |
| 1 | MIDDLE | MIDDLE | top |
| 2 | MIDDLE | MIDDLE | bottom |
| 3 | ALLY | ALLY_FRONT | bottom |
| 4 | ALLY | ALLY_BACK | bottom |
| 5 | ALLY | ALLY_BACK | top |
| 6 | ALLY | ALLY_FRONT | center |
| 7 | ENEMY | ENEMY_FRONT | top |
| 8 | ENEMY | ENEMY_BACK | top |
| 9 | ENEMY | ENEMY_BACK | bottom |
| 10 | ENEMY | ENEMY_FRONT | bottom |
| 11 | ENEMY | ENEMY_FRONT | center |

Available as constants: `CELL_TO_ZONE`, `CELL_TO_ROW`, `CELL_VERTICAL` (from `01_grid_constants.js`).

### 6.3 Hex Pixel Coordinates

The renderer must compute pixel positions for each cell. Since the grid is small and fixed, a hardcoded position table is recommended over computed hex math.

**Approach:** Define a `CELL_POSITIONS` lookup mapping cell ID → `{x, y}` pixel coordinates. Derive positions from the grid diagram above, using consistent hex spacing. See §12.1 for a reference computation.

### 6.4 Hex Rendering

- Use Phaser's `add.polygon()` with 6 vertices for flat-topped hexagons
- Middle cells (1, 2) should use a **neutral/distinct color** since they belong to both zones
- Empty cells: outlined hex only
- Occupied cells: filled hex + unit icon

---

## 7. Unit Rendering

### 7.1 Unit Icon Composition

Each unit on the grid is rendered as:
1. **Colored circle** (background) — party-colored
2. **Unicode symbol** (foreground) — centered inside the circle, rendered as Phaser text

### 7.2 Unit Type Icons

| Unit Type | Unicode Symbol | Notes |
|-----------|---------------|-------|
| Footman (MELEE_SINGLE) | ⚔ (U+2694) | Crossed swords |
| Archer (MISSILE) | 🏹 (U+1F3F9) | Bow and arrow |

### 7.3 Color Scheme

| Element | Ally | Enemy |
|---------|------|-------|
| Circle background | Cyan (`#00FFFF`) | Magenta (`#FF00FF`) |
| Symbol foreground | White (`#FFFFFF`) | Black (`#000000`) |

### 7.4 Dead Unit Rendering

When a unit dies:
- Replace unit icon with a skull symbol (💀 or ☠)
- Fade the circle to a desaturated/dark version
- The cell remains occupied (dead units are not removed from the grid for visual clarity)

### 7.5 Status Effect Indicators

Overlay small icons or colored markers on the unit circle when status effects are active:

| Status | Visual Indicator |
|--------|-----------------|
| LOCKED | 🔒 Chain/lock overlay, DR aura glow |
| EMPOWERED | ⬆ Up-arrow or golden glow |
| WEAKENED | ⬇ Down-arrow or grey tint |

### 7.6 Unit Info (Hover Panel Content)

When a unit is hovered or clicked, display this info table in the Hover Panel:

| Field | Source | Format |
|-------|--------|--------|
| Name | `unit.name` | String |
| Attack | `getEffectiveAttackPower(unit)` | `"3 × 2"` (power × nAttacks) |
| Attack Type | `unit.attackTemplate` | `"Melee"` or `"Missile"` |
| Protection | `unit.protection` | Number, show bar if desired |
| Wounds | `unit.wounds` / `getWoundThreshold(unit)` | `"1/3"` |
| DR | `getEffectiveDR(unit)` | Number |
| Speed | `unit.speed` | Number |
| Status | `unit.statusEffects` | List of active effects or "None" |
| State | `unit.dead` | "DEAD" or "Alive" |

---

## 8. Spell Panel Rendering

### 8.1 Layout

The bottom strip shows 6 spell slots arranged horizontally (see mockup). Slots 1–3 are **Ally spells**, slots 4–6 are **Enemy spells** (see §15 Clarifications — this pairing may need revision).

Each slot is a card containing:
- Spell name
- A mini super-hex (7 cells) showing which positions have symbols
- Hotkey number (1–6)

### 8.2 Spell Mini-Hex

Each spell has a `hexes` map (index 0–6 → SpellSymbol). Render a small 7-hex diagram following the same super-hex structure as the grid:

```
TOP:   [5]  [0]  [1]
CENTER:     [6]
BOTTOM:[4]  [3]  [2]
```

Color each hex by its symbol:

| Symbol | Color | Meaning |
|--------|-------|---------|
| HP | Green | Heal |
| POS | Blue | Move |
| LOCK | Yellow/Gold | Lock (disable) |
| DMG | Red | Direct damage |
| (empty) | Grey/transparent | No effect on this cell |

### 8.3 Spell Symbol FX Descriptions (for hover tooltips)

| Symbol | Description |
|--------|------------|
| HP | "Heals all wounds" |
| POS | "Relocates unit to a nearby cell" |
| LOCK | "Locks unit: cannot attack, but gains +10 DR" |
| DMG | "Inflicts 1 wound (bypasses armor and protection)" |

### 8.4 Spell Overlay Visualization

When a spell is hovered, highlight the grid cells it would affect:

| Target Zone | Spell Index → Grid Cell |
|-------------|------------------------|
| ALLY | 0→0, 1→1, 2→2, 3→3, 4→4, 5→5, 6→6 |
| ENEMY | 0→7, 1→8, 2→9, 3→10, 4→2, 5→1, 6→11 |

Note: Enemy spells at indices 4 and 5 target the shared Middle cells (2 and 1). This means enemy-targeted spells can affect units on Middle cells regardless of their party.

### 8.5 Spell State

| Condition | Visual State |
|-----------|-------------|
| `phase === SPELLCAST` + spell exists | **Enabled** — clickable, bright |
| `phase === AFTERMATH` | **Disabled** — greyed out, tooltip: "Use Revert to go back and cast a Spell before this round resolves!" |
| Spell already used (not in `remainingSpells`) | **Spent** — removed or crossed out, tooltip: "You've already used this Spell!" |
| `phase === ENDGAME` | Hidden or fully disabled |

### 8.6 Spell Activation Flow

1. Player clicks REVERT button → phase changes to SPELLCAST
2. Spell slots become enabled
3. Player clicks a spell slot (or presses hotkey 1–6)
4. Frontend calls `game.castSpell(spellIndex, targetZone)`
5. Phase returns to AFTERMATH with new state

---

## 9. UI Layout

Based on the mockup image, the layout is a fixed-size panel arrangement:

```
┌──────────────┬──────────────────────┬──────────────┐
│              │                      │              │
│   LOG PANEL  │     GRID PANEL       │  HOVER PANEL │
│              │                      │              │
│              │                      │              │
├─[BTN][BTN]───┼──── STATUS BAR ──────┤              │
├──────────────┴──────────────────────┴──────────────┤
│  [1:S-Ally] [2:S-Ally] [3:S-Ally] [4:S-Enemy] [5:S-Enemy] [6:S-Enemy] │
└───────────────────────────────────────────────────┘
```

### 9.1 Panel Descriptions

| Panel | Content | Size Notes |
|-------|---------|------------|
| **Log Panel** | Scrollable text log of combat events. Two buttons docked at bottom: CONFIRM ("INDEED") and REVERT ("NEVER SO") | Left column, ~1/4 screen width (300–500px) |
| **Grid Panel** | The 12-cell hex battle grid with unit icons | Center, largest area |
| **Hover Panel** | Unit info table or spell FX description (context-dependent) | Right column, ~1/4 screen width (300–500px) |
| **Status Bar** | Single-line text showing current game status | Below grid, spans grid width |
| **Spell Panel** | 6 spell slot cards in a horizontal row | Full width bottom strip |

### 9.2 Panel Sizing

- **Static sizing** — panels are sized once based on display dimensions, nothing resizes dynamically
- Width range for side panels: 300–500px (quarter of screen width)
- Spell cards: equal width, filling the bottom strip

### 9.3 Button Labels

| Button | Label Text | Hotkeys |
|--------|-----------|---------|
| Confirm | "INDEED" | Space, Enter |
| Revert | "NEVER SO" | Esc, Tab |
| Restart | (shown in ENDGAME only) | — |

---

## 10. Tween Animation Specifications

All animations use Phaser tweens. Events arrive in sequential groups — play each group fully before starting the next.

### 10.1 Combat Tweens

#### ATTACK (on source unit)
- Sway forward halfway toward target position, then back
- Duration: 0.2s total
- Easing: ease-out then ease-in

#### BLOCKED (on target unit)
- Elastic bounce (scale oscillation)
- Brief white flash overlay
- Duration: 0.3s

#### HIT (on target unit)
- Impact flash
- Show floating damage number (the `damage` field)
- Update protection bar/number
- Duration: 0.2s

#### WOUNDED (on target unit)
- Flash red tint
- Yank diagonally (brief displacement + return)
- Flash-in a wound icon overlay (attached to unit)
- Duration: 0.4s

#### DEATH (on target unit)
- Color inversion flash + abrupt downward yank, then return
- 0.5s delay
- Sine fade-out of unit icon
- Fade-in skull icon on the cell
- Total duration: ~1.2s

#### PASS (on attacker)
- Subtle shake or dim flash
- Duration: 0.2s

#### NEW_ROUND
- Round number title animation (flash in "Round N")
- Duration: 0.5s

### 10.2 Spell Tweens

#### SPELL (cast effect)
- Highlight affected cells with spell color
- Cast animation on the spell card
- Duration: 0.5s

#### HEAL
- Green pulse glow on target
- Wounds counter resets visually
- Duration: 0.3s

#### MOVE
- Slide unit icon from `fromCellId` to `toCellId`
- Duration: 0.3s

#### SWAP
- Simultaneous slide of both units to each other's positions
- Duration: 0.3s

#### BUFF
- Status effect icon appears on unit
- Color tint corresponding to effect type
- Duration: 0.3s

### 10.3 Animation Queue Pattern

```javascript
async function playEventGroups(eventGroups) {
  for (const group of eventGroups) {
    // Play all events in this group simultaneously
    const tweenPromises = group.map(evt => playOneEvent(evt));
    await Promise.all(tweenPromises);
  }
}
```

---

## 11. Status Bar Content

The status bar shows context-dependent single-line messages:

| Phase | Status Text |
|-------|-------------|
| COMBAT | `"Round {N} — Combat..."` |
| AFTERMATH | `"Round {N} complete — CONFIRM or REVERT?"` |
| AFTERMATH (finished) | `"Battle concluded — CONFIRM to see results"` |
| SPELLCAST | `"Choose a spell to cast (1-6)"` |
| ENDGAME | `"GAME OVER — Score: {score} — Rating: {rating}"` |

The status bar should have a **cyclic alpha animation** (gentle pulsing opacity) to feel dynamic without being distracting.

---

## 12. Supplementary Implementation Details

### 12.1 Hex Position Computation Reference

For a flat-topped hex with radius `r`, the 6 vertices are at angles 0°, 60°, 120°, 180°, 240°, 300° from center:

```javascript
function hexVertices(cx, cy, r) {
  const verts = [];
  for (let i = 0; i < 6; i++) {
    const angle = (Math.PI / 180) * (60 * i);
    verts.push(cx + r * Math.cos(angle), cy + r * Math.sin(angle));
  }
  return verts;
}
```

Cell center positions can be derived from the super-hex layout. With hex radius `r`, the horizontal spacing between adjacent columns is `1.5 * r` and vertical spacing is `sqrt(3) * r`. Recommended: precompute a lookup table.

### 12.2 Log Panel Content Generation

The log panel converts graphic events into human-readable text. Use the same mapping as the renderer stub:

| Event | Log Text |
|-------|----------|
| NEW_ROUND | `"═══ Round {N} ═══"` |
| ATTACK | `"⚔ {attackerId} attacks {targetId}"` |
| BLOCKED | `"🛡 Attack on {targetId} BLOCKED"` |
| HIT | `"💥 {targetId} hit for {damage} (prot: {protRemaining})"` |
| WOUNDED | `"🩸 {targetId} WOUNDED ({wounds}/{threshold})"` |
| DEATH | `"💀 {targetId} KILLED"` |
| HEAL | `"💚 {targetId} healed"` |
| MOVE | `"➡ {unitId} moved to cell {toCellId}"` |
| SWAP | `"🔄 {unitAId} swapped with {unitBId}"` |
| PASS | `"⏭ {attackerId} has no target — PASS"` |
| SPELL | `"✨ Spell {spellId} cast on {targetZone} zone"` |
| BUFF | `"⚡ {targetId} gained {effect}"` |

### 12.3 Hover Panel Behavior

- **Hover over unit** → show unit info table (§7.6)
- **Hover over spell** → show spell FX description list (§8.3) composed from the distinct symbols in the spell
- **Click unit** → lock info display until clicking elsewhere
- **Click elsewhere** → clear locked display

### 12.4 Mouse Reactions on Units

On hover:
- Highlight the hex cell (brighten border or add glow)
- Enlarge ("pop") the unit circle icon slightly (scale to 1.1–1.2×)

On hover exit:
- Return to normal

### 12.5 Button Hover Effects

"Impactful gradient-based recoloring animations on button hover" — apply a gradient tween that shifts the button's fill color across a range on mouse enter, and reverses on mouse leave.

---

## 13. Default Content Data Reference

### 13.1 Unit Templates

| Template | Atk Type | Atk Power | N Attacks | Protection | Toughness | Speed | DR |
|----------|----------|-----------|-----------|------------|-----------|-------|----|
| Footman | Melee | 3 | 2 | 14 | 1 | 3 | 1 |
| Archer | Missile | 5 | 1 | 8 | 0 | 4 | 0 |

### 13.2 Default Party Setup

**Ally Party (3 units):**

| Unit | Template | Cell ID | Position Name |
|------|----------|---------|---------------|
| ally_0 | Footman | 0 | Ally Front Top |
| ally_1 | Footman | 3 | Ally Front Bottom |
| ally_2 | Archer | 4 | Ally Back Bottom |

**Enemy Party (5 units):**

| Unit | Template | Cell ID | Position Name |
|------|----------|---------|---------------|
| enemy_0 | Footman | 7 | Enemy Front Top |
| enemy_1 | Footman | 10 | Enemy Front Bottom |
| enemy_2 | Footman | 11 | Enemy Front Center |
| enemy_3 | Archer | 8 | Enemy Back Top |
| enemy_4 | Archer | 9 | Enemy Back Bottom |

### 13.3 Default Spells (4 spells)

| Index | ID | Name | Active Hex Indices | Symbol |
|-------|----|------|--------------------|--------|
| 0 | spell_1 | Healing Light | 0, 2, 4 | HP |
| 1 | spell_2 | Displacement | 0, 2, 4 | POS |
| 2 | spell_3 | Binding Chains | 0, 2, 4 | LOCK |
| 3 | spell_4 | Arcane Blast | 0, 2, 4 | DMG |

### 13.4 Score Ratings

| Score Threshold | Rating |
|----------------|--------|
| ≥ 300 | S |
| ≥ 200 | A |
| ≥ 100 | B |
| ≥ 0 | C |
| < 0 | D |

### 13.5 Score Weights

| Event | Points Per Occurrence |
|-------|--------------------|
| Wound dealt (to enemy) | +20 |
| Wound received (by ally) | -10 |
| Enemy unit killed | +50 |
| Ally unit lost | -100 |

---

## 14. FSM Flow Diagram (Frontend Perspective)

```
                    ┌──────────┐
                    │  BOOT    │
                    └────┬─────┘
                         │ dispatch(START)
                         ▼
              ┌─────────────────────┐
              │  AFTERMATH          │ ◄──────────────────────────┐
              │  (show round result │                            │
              │   + await input)    │                            │
              └──┬──────────┬───────┘                            │
                 │          │                                    │
    dispatch     │          │ dispatch                           │
    (CONFIRM)    │          │ (REVERT)                           │
                 │          │                                    │
      ┌──────────┘          └──────────┐                        │
      ▼                                ▼                        │
 ┌─────────┐                   ┌────────────┐                   │
 │ finished?│                   │ SPELLCAST  │                   │
 │  Y / N   │                   │ (pick spell│                   │
 └─┬─────┬──┘                   │  + zone)   │                   │
   │     │                      └──────┬─────┘                   │
   │     │                             │ castSpell(idx, zone)    │
   │     │ N: runs next round          │ runs spell + round      │
   │     └─────────────────────────────┴─────────────────────────┘
   │ Y
   ▼
 ┌─────────┐         dispatch(START)         ┌──────────┐
 │ ENDGAME │ ────────────────────────────► │  (restart)│
 │ (score) │                                └──────────┘
 └─────────┘
```

---

## 15. REQUIRED CLARIFICATIONS

The following items are **ambiguous or underspecified** in the current design documents and must be resolved before implementation:

### 15.1 Spell Slot Layout — 6 Slots vs 4 Spells

The mockup shows **6 spell slots** (labeled 1–6), and the WIP overview mentions "3×2 linked spells — when Ally spell X (slots 1-3) is used, Enemy spell X+3 (slots 4-6) is also spent." However, the game logic only has **4 spells** in `DEFAULT_SPELLS`, and `castSpell()` takes a single index + target zone.

**Questions:**
- Are there meant to be 6 spells total (3 ally-targeted, 3 enemy-targeted)?
- Or are the 4 existing spells shown twice (once for each target zone), with the "linked" mechanic meaning: casting spell 1 on allies also consumes the enemy version (slot 4)?
- Does `castSpell` need to be called with a specific target zone, or is the zone determined by which slot (1–3 = ALLY, 4–6 = ENEMY) the player clicks?
- If linked: should both spell slots visually grey out when one is used?

### 15.2 Spell Target Zone Selection

The current `castSpell(spellIndex, targetZone)` API accepts both index and zone as separate parameters. The `12_main.js` stub currently hardcodes `'ENEMY'` as the zone.

**Question:** How does the player choose the target zone? Options:
- (a) Fixed by slot position (1–3 = ALLY, 4–6 = ENEMY) — no choice needed
- (b) Player picks zone via a sub-menu or toggle after selecting a spell
- (c) Each spell can only target one zone (zone is inherent to the spell)

### 15.3 Unit Icons for Additional Unit Types

Only Footman and Archer exist currently. If more unit types are planned:
**Question:** Should the frontend be built to extensibly map `attackTemplate` → icon, or hardcode just these two?

### 15.4 Hex Cell Visual Style

**Questions:**
- What specific hex colors should be used for: ally cells, enemy cells, middle cells, empty cells?
- Should cells have visible borders always, or only on hover?
- What is the hex size (radius in pixels) relative to the game canvas?

### 15.5 Canvas / Game Resolution

**Question:** What is the target resolution? Fixed pixel size, or responsive to window? The mockup appears to be ~960×540 (16:9 at low res). Confirm the target canvas dimensions.

### 15.6 Sound

The spec mentions: "On CONFIRM: Wait 1 second, play a generic battle sound."
**Question:** Are sounds in scope for this implementation? If so, what sound assets (or should procedural/Web Audio be used)?

### 15.7 "Wait for Input" Timing

The WIP overview notes: "General 'wait for input' — it's no good that we start immediately with round played out!"
**Question:** Should there be a delay or animation-complete gate before enabling CONFIRM/REVERT buttons? Currently the logic runs synchronously and immediately produces results. The animation queue (§10.3) would provide natural gating — should buttons be disabled until all tweens complete?

### 15.8 Endgame Screen Content

**Question:** Beyond score and rating, what should the endgame screen display? Options:
- Stats breakdown (wounds dealt, wounds received, units killed, units lost)
- Surviving units
- Restart button only
- Any transition animation from battlefield to score screen?

### 15.9 Log Panel — Unit ID vs Display Name

The log currently uses unit IDs (e.g., "ally_0"). Should it use display names (e.g., "Footman") with position context, or are IDs acceptable?

**Question:** Preferred log format — `"ally_0 attacks enemy_2"` or `"Footman (cell 0) attacks Footman (cell 10)"`?
