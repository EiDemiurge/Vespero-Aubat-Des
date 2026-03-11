# Milestone 1 — Damage + Targeting Pipelines

> Focused task brief. This file contains everything needed to implement Milestone 1.
> Do NOT read or modify any files not listed here.

---

## Files to Read (in order)

| File | What to look for |
|------|-----------------|
| `src/00_enums.js` | GraphicEventType, LogicEventType, PipelineType, AttackTemplate, StatusEffect |
| `src/02_data_factories.js` | cloneGameState, createPipelineResult, mergeResult, gfxEvent, logEvent, popCoin |
| `src/04_helpers.js` | getEffectiveDR, getEffectiveAttackPower, getWoundThreshold, getAllUnits, getUnitAtCell, isAlly, isEnemy, canAct |
| `src/01_grid_constants.js` | ROW, CELL_VERTICAL, CELL_TO_ROW, ALLY_ATTACK_ROW_ORDER, ENEMY_ATTACK_ROW_ORDER, MISSILE_PATHS_ALLY, MISSILE_PATHS_ENEMY |
| `src/05_pipeline_damage.js` | Stub to implement |
| `src/06_pipeline_target.js` | Stub to implement |
| `tests/test_damage.js` | Acceptance tests for damage |
| `tests/test_targeting.js` | Acceptance tests for targeting |

## Files to Modify

- `src/05_pipeline_damage.js` — implement `runDamagePipeline` and `runWoundFX`
- `src/06_pipeline_target.js` — implement `runFindTargetPipeline`
- Create `tests/test_damage_edge.js` — additional edge case tests

## Do NOT Read or Modify

- `src/07_pipeline_spell.js` through `src/12_main.js`
- `docs/scenarios/GOLDEN_TEST.md`, `GOLDEN_TEST_SPELLS.md` (full-round traces, not needed yet)
- `docs/FULL_SPEC.md`, `docs/DATA_SCHEMA.md` (relevant sections are extracted below)
- `SCHEME.md`

---

## Hard Rules (from AGENTS.md — must follow)

- **No classes, no `this`.** Plain objects and standalone functions.
- **Never mutate input state.** First line: `const state = cloneGameState(inputState);`
- **Always return a PipelineResult.** Use `createPipelineResult(state, PipelineType.X)`.
- **Use `getEffective*()`** helpers for DR, attack power, nAttacks. Never read base stats directly.
- **Use `popCoin(state.coinStack)`** for every random decision. Never `Math.random()`.
- **Consume coins consistently.** If a center-cell code path is reached, pop a coin even if only one target exists.
- **GraphicEvents** are nested arrays: `[[parallel group], [parallel group], ...]`
- **LogicEvents** are a flat array: `[{type, ...data}, ...]`
- Use `gfxEvent()` and `logEvent()` factories. Do not construct event objects by hand.
- Use `const` by default, `let` only when reassignment is needed. No `var`.

---

## Part A: Damage Pipeline

### Signature

```javascript
function runDamagePipeline(inputState, attackerId, targetId, rawDamage) -> PipelineResult
function runWoundFX(inputState, targetId, woundCount) -> PipelineResult
```

### Algorithm for `runDamagePipeline`

```
1. Clone state. Find attacker and target units by ID in the clone.

2. effectiveDamage = rawDamage - getEffectiveDR(target)
   If effectiveDamage <= 0:
     -> Emit BLOCKED graphic + logic events. Return.

3. oldProtection = target.protection
   target.protection = max(0, oldProtection - effectiveDamage)

   If oldProtection > 0 AND effectiveDamage <= oldProtection:
     -> Protection absorbed all damage (may now be 0 with 0 excess).
     -> Emit HIT graphic event. Return.

   If oldProtection > 0 AND effectiveDamage > oldProtection:
     -> Protection just broke. excess = effectiveDamage - oldProtection.
     -> newWounds = min(excess, 1)     // CAPPED AT 1 WOUND when breaking prot
     -> Go to wound check.

   If oldProtection === 0:
     -> Protection was already gone.
     -> newWounds = effectiveDamage     // FULL DAMAGE becomes wounds
     -> Go to wound check.

4. WOUND CHECK:
   target.wounds += newWounds
   If target.wounds >= getWoundThreshold(target):
     -> target.dead = true
     -> Emit DEATH graphic + logic events. Return.
   Else:
     -> Emit WOUNDED graphic + logic events. Return.
```

### Concrete Examples (for verifying your logic)

| Scenario | Raw | DR | Eff | Old Prot | New Prot | Excess | Wounds | Result |
|----------|-----|----|-----|----------|----------|--------|--------|--------|
| DR blocks all | 5 | 10 | -5 | 14 | 14 | — | 0 | BLOCKED |
| Prot absorbs | 5 | 1 | 4 | 20 | 16 | — | 0 | HIT |
| Prot breaks exactly | 3 | 1 | 2 | 2 | 0 | 0 | 0 | HIT |
| Prot breaks with excess | 5 | 1 | 4 | 2 | 0 | 2 | min(2,1)=1 | WOUNDED |
| Prot already 0 | 3 | 1 | 2 | 0 | 0 | — | 2 | WOUNDED (or DEATH) |
| Kill | 5 | 0 | 5 | 0 | 0 | — | 5 | DEATH (if wounds >= threshold) |

### Event Contract

**Graphic events** (push as `[gfxEvent(...)]` — each is its own parallel group):

| Exit | Event | Data fields |
|------|-------|-------------|
| BLOCKED | `GraphicEventType.BLOCKED` | `{ targetId }` |
| HIT | `GraphicEventType.HIT` | `{ targetId, damage: effectiveDamage, protRemaining: target.protection }` |
| WOUNDED | `GraphicEventType.WOUNDED` | `{ targetId, wounds: target.wounds, threshold: getWoundThreshold(target) }` |
| DEATH | `GraphicEventType.DEATH` | `{ targetId }` |

**Logic events** (push to flat `logicEvents` array):

| Condition | Event type | Notes |
|-----------|-----------|-------|
| Blocked | `LogicEventType.BLOCKED` | Any target party |
| Wound dealt to **enemy** | `LogicEventType.WOUND_DEALT` | Per wound point (emit once with amount, or once per wound — check test expectations) |
| Wound dealt to **ally** | `LogicEventType.WOUND_RECEIVED` | Same |
| Enemy killed | `LogicEventType.UNIT_KILLED` | |
| Ally killed | `LogicEventType.UNIT_LOST` | |

Use `isAlly(target)` / `isEnemy(target)` from helpers to determine party.

**Important for test compatibility:** Look at what `test_damage.js` asserts. It checks:
- `result.logicEvents.some(e => e.type === LogicEventType.BLOCKED)` for the blocked case
- `result.logicEvents.some(e => e.type === LogicEventType.UNIT_KILLED)` for the death case
So emit at least one logic event of the right type per exit path.

### Algorithm for `runWoundFX`

Used by spell DMG FX (Milestone 3), but implement now since `test_damage_edge.js` will test it.

```
1. Clone state. Find target by ID.
2. target.wounds += woundCount
3. Same wound check as step 4 above (wounds >= threshold -> DEATH, else WOUNDED).
4. Emit matching graphic + logic events.
```

No DR, no protection involved. Bypasses everything.

---

## Part B: Find Target Pipeline

### Signature

```javascript
function runFindTargetPipeline(inputState, attackerId, alreadyTargeted)
  -> { targetId: string|null, state: GameState }
```

Returns the updated `state` because coin pops may have occurred (the caller needs the modified coin stack).

### Branch: MELEE_SINGLE

Row-by-row scan for the nearest enemy.

```
1. Get attacker's vertical position: CELL_VERTICAL[attacker.positionCellId]
   - 'bottom' -> scanDirection = 'BTT' (Bottom-to-Top: use ROW arrays as-is)
   - 'top'    -> scanDirection = 'TTB' (Top-to-Bottom: reverse ROW arrays)
   - 'center' -> popCoin(state.coinStack): true = 'BTT', false = 'TTB'

2. Get row search order based on attacker's party:
   - Ally attacker  -> ALLY_ATTACK_ROW_ORDER  = ['MIDDLE', 'ENEMY_FRONT', 'ENEMY_BACK']
   - Enemy attacker -> ENEMY_ATTACK_ROW_ORDER = ['MIDDLE', 'ALLY_FRONT', 'ALLY_BACK']

3. MIDDLE ROW RULE: If attacker is on the Middle row (cell 1 or 2),
   skip 'MIDDLE' in the row search order. They're already there — no self-targeting.

4. For each rowName in search order:
     cells = ROW[rowName]   // e.g. ROW.ENEMY_FRONT = [10, 11, 7]
     If scanDirection === 'TTB': cells = [...cells].reverse()
     For each cellId in cells:
       unit = getUnitAtCell(state, cellId)
       If unit exists AND unit is enemy of attacker AND unit.id not in alreadyTargeted:
         -> return { targetId: unit.id, state }

5. No target found -> return { targetId: null, state }
```

"Enemy of attacker" means: if attacker is ALLY, target must be ENEMY (or on enemy side). Use the unit's `party` field — any alive unit whose `party !== attacker.party`.

### Branch: MISSILE

Flight path lookup — missile hits first occupied enemy cell.

```
1. Get attacker's vertical position: CELL_VERTICAL[attacker.positionCellId]

2. Determine lookup key:
   - If NOT 'center': key = attacker.positionCellId (as number)
   - If 'center': popCoin(state.coinStack)
     - true  -> key = `${attacker.positionCellId}_bot`
     - false -> key = `${attacker.positionCellId}_top`

3. Look up flight path:
   - Ally attacker  -> path = MISSILE_PATHS_ALLY[key]
   - Enemy attacker -> path = MISSILE_PATHS_ENEMY[key]

4. For each cellId in path:
     unit = getUnitAtCell(state, cellId)
     If unit exists AND unit.party !== attacker.party AND unit.id not in alreadyTargeted:
       -> return { targetId: unit.id, state }

5. No target found -> return { targetId: null, state }
```

**Note:** Paths from Middle row sources (cells 1, 2) are 3 cells long instead of 4. This is already handled in the lookup tables — no special logic needed.

### Melee Search Order Quick Reference

These are the exact cell sequences from `docs/GRID_REFERENCE.md`. Use them to verify your logic against the lookup tables.

**Ally attackers → searching for enemies:**

| Source | Vertical | Direction | Full search sequence |
|--------|----------|-----------|---------------------|
| 4 (AB-Bot) | bottom | BTT | [2,1] → [10,11,7] → [9,8] |
| 5 (AB-Top) | top | TTB | [1,2] → [7,11,10] → [8,9] |
| 3 (AF-Bot) | bottom | BTT | [2,1] → [10,11,7] → [9,8] |
| 0 (AF-Top) | top | TTB | [1,2] → [7,11,10] → [8,9] |
| 6 (AF-Ctr) | center | coin | true: [2,1]→[10,11,7]→[9,8] / false: [1,2]→[7,11,10]→[8,9] |

**Enemy attackers → searching for allies:**

| Source | Vertical | Direction | Full search sequence |
|--------|----------|-----------|---------------------|
| 10 (EF-Bot) | bottom | BTT | [2,1] → [3,6,0] → [4,5] |
| 7 (EF-Top) | top | TTB | [1,2] → [0,6,3] → [5,4] |
| 11 (EF-Ctr) | center | coin | true: [2,1]→[3,6,0]→[4,5] / false: [1,2]→[0,6,3]→[5,4] |
| 9 (EB-Bot) | bottom | BTT | [2,1] → [3,6,0] → [4,5] |
| 8 (EB-Top) | top | TTB | [1,2] → [0,6,3] → [5,4] |

**Middle row sources (skip Middle in search):**

| Source | Vertical | Ally searching enemy | Enemy searching ally |
|--------|----------|---------------------|---------------------|
| 2 (M-Bot) | bottom | [10,11,7] → [9,8] | [3,6,0] → [4,5] |
| 1 (M-Top) | top | [7,11,10] → [8,9] | [0,6,3] → [5,4] |

---

## Implementation Order

1. **Implement `runDamagePipeline` in `src/05_pipeline_damage.js`.**
2. **Implement `runWoundFX` in `src/05_pipeline_damage.js`.**
3. **Run `node tests/test_damage.js`.** Fix until all 4 cases pass.
4. **Create `tests/test_damage_edge.js`** with these cases:
   - Prot breaks exactly (damage = prot, excess 0 → HIT, 0 wounds)
   - Prot already 0 (full damage → wounds)
   - Prot breaks with excess → wound capped at 1
   - WoundFX bypasses DR and protection
5. **Run `node tests/test_damage_edge.js`.** Fix until all pass.
6. **Implement `runFindTargetPipeline` in `src/06_pipeline_target.js`.**
7. **Run `node tests/test_targeting.js`.** Fix until all 4 cases pass.

### Running Tests

Tests load code via `eval(require('fs').readFileSync('dist/game_logic.js', 'utf8'))`. Before running tests, build the concatenated JS:

```bash
./build.sh
```

Or if build isn't available, concatenate manually:

```bash
cat src/0*.js src/1*.js > dist/game_logic.js
```

Then run:

```bash
node tests/test_damage.js
node tests/test_damage_edge.js
node tests/test_targeting.js
```

### Test File Template for `test_damage_edge.js`

```javascript
// Test: Damage Pipeline — Edge Cases
// Run: node tests/test_damage_edge.js

eval(require('fs').readFileSync('dist/game_logic.js', 'utf8'));

function assert(cond, msg) {
  if (!cond) { console.error('FAIL:', msg); process.exit(1); }
}

function makeState(targetOverrides) {
  const attacker = createUnit({
    id: 'atk', name: 'Attacker', party: 'ALLY',
    attackTemplate: AttackTemplate.MELEE_SINGLE,
    attackPower: 5, nAttacks: 1, toughness: 1, speed: 3, drPhys: 0,
    positionCellId: 0, protection: 10,
  });
  const target = createUnit({
    id: 'tgt', name: 'Target', party: 'ENEMY',
    attackTemplate: AttackTemplate.MELEE_SINGLE,
    attackPower: 3, nAttacks: 1, toughness: 1, speed: 3, drPhys: 0,
    positionCellId: 7, protection: 10,
    ...targetOverrides,
  });
  return {
    roundNumber: 1, coinStack: Array(100).fill(true),
    allyParty: createParty('ALLY', [attacker]),
    enemyParty: createParty('ENEMY', [target]),
    remainingSpells: [], phase: Phase.COMBAT, finished: false, stats: createStats(),
  };
}

// Case 1: Prot breaks exactly — no wound
(function testProtBreaksExactly() {
  const state = makeState({ drPhys: 0, protection: 5, toughness: 1 });
  // 5 damage, 0 DR = 5 effective. Prot 5 -> 0, excess = 0. No wound.
  const result = runDamagePipeline(state, 'atk', 'tgt', 5);
  const t = result.gameState.enemyParty.units[0];
  assert(t.protection === 0, 'Protection should be 0');
  assert(t.wounds === 0, 'No wounds when prot breaks exactly');
  assert(!t.dead, 'Not dead');
  console.log('PASS: testProtBreaksExactly');
})();

// Case 2: Prot already 0 — full damage to wounds
(function testProtAlreadyZero() {
  const state = makeState({ drPhys: 0, protection: 0, toughness: 5 });
  // 5 damage, 0 DR, prot already 0 -> wounds = 5. Threshold = 2+5 = 7. 5 < 7 -> WOUNDED
  const result = runDamagePipeline(state, 'atk', 'tgt', 5);
  const t = result.gameState.enemyParty.units[0];
  assert(t.protection === 0, 'Protection stays 0');
  assert(t.wounds === 5, 'All damage becomes wounds');
  assert(!t.dead, 'Not dead (threshold 7)');
  console.log('PASS: testProtAlreadyZero');
})();

// Case 3: Prot breaks with excess — wound capped at 1
(function testProtBreaksWoundCapped() {
  const state = makeState({ drPhys: 0, protection: 2, toughness: 5 });
  // 5 damage, 0 DR, prot 2 -> 0, excess = 3 -> wounds = min(3,1) = 1
  const result = runDamagePipeline(state, 'atk', 'tgt', 5);
  const t = result.gameState.enemyParty.units[0];
  assert(t.protection === 0, 'Protection depleted');
  assert(t.wounds === 1, 'Wound capped at 1 when breaking prot');
  assert(!t.dead, 'Not dead');
  console.log('PASS: testProtBreaksWoundCapped');
})();

// Case 4: WoundFX bypasses DR and protection
(function testWoundFX() {
  const state = makeState({ drPhys: 10, protection: 100, toughness: 1 });
  // WoundFX ignores DR and prot. 1 wound direct. Threshold = 3.
  const result = runWoundFX(state, 'tgt', 1);
  const t = result.gameState.enemyParty.units[0];
  assert(t.protection === 100, 'Protection untouched');
  assert(t.wounds === 1, 'Should have 1 wound');
  assert(!t.dead, 'Not dead');
  console.log('PASS: testWoundFX');
})();

// Case 5: WoundFX kills
(function testWoundFXKills() {
  const state = makeState({ drPhys: 10, protection: 100, toughness: 0, wounds: 1 });
  // Threshold = 2+0 = 2. Wounds 1 + 1 = 2 >= 2 -> DEATH
  const result = runWoundFX(state, 'tgt', 1);
  const t = result.gameState.enemyParty.units[0];
  assert(t.dead === true, 'Should be dead');
  console.log('PASS: testWoundFXKills');
})();

console.log('\nAll damage edge case tests passed.');
```

---

## Verification Gate

All three commands must output only PASS lines:

```bash
./build.sh && node tests/test_damage.js && node tests/test_damage_edge.js && node tests/test_targeting.js
```
