
# FS-1 Logic Spec — Analysis & Gap Report

---

## Part A: Elements Needing Clarification

Ranked from **most underspecified** to least. Items marked ⚠️ block implementation; items marked ❓ are ambiguous but workable with assumptions.

---

### ⚠️ 1. Coin Stack — Purpose & Consumption

The spec says the Coin Stack is "100 Booleans" re-initialized every round, but never explains **what consumes them or how**. This is central to the combat system and completely undefined.

- Is it a shared randomness pool (pop the next bool to decide hit/miss)?
- Is it indexed by some unit or attack parameter?
- Do both parties draw from the same stack, or each from their own?

> **This is the single biggest gap — no pipeline references it directly.**

---

### ⚠️ 2. Unit Data Model — No Parameter List

The spec says a Unit is "a flat data map of numeric and string parameters" but never enumerates them. Scattered references imply at least:

| Implied Parameter     | Referenced In                          |
|-----------------------|----------------------------------------|
| N of Attacks          | Unit Attack Loop (L3)                  |
| Attack Template       | Unit Attack Loop / Find Target         |
| Toughness             | Damage Pipeline (step 3)               |
| Protection            | Damage Pipeline (step 2)               |
| Damage Reduction      | Damage Pipeline (step 1)               |
| Speed (?)             | Turn order ("fastest unit")            |
| Damage / Power (?)    | Damage Pipeline input — never sourced  |
| Alive / Dead status   | Party Turn Loop, Damage Pipeline       |
| Locked status         | Party Turn Loop filter                 |
| Temp effects (Empower, Weaken, Lock) | New Round Pipeline       |

> **Need the full canonical list with value types and ranges.**

---

### ⚠️ 3. Attack Templates — Undefined

The Find Target pipeline and Unit Attack Loop both depend on an "Attack Template" to determine *who* can be targeted, but templates are never defined.

- "Melee Wide" is the only example (finds 1–3 targets).
- What other templates exist? (Melee Single? Ranged? AoE?)
- Does the template encode valid hex-cells, or a targeting rule?
- Is the template a property of the Unit or of the Attack action?

---

### ⚠️ 4. Damage Source — Where Does the Number Come From?

The Damage Pipeline receives "incoming damage" and processes reduction/protection/wounding, but nowhere does the spec say **how the raw damage value is determined**.

- Is it a flat Unit parameter (e.g., `Attack Power`)?
- Does the Coin Stack modify it?
- Is it computed from attacker stats vs. some formula?

---

### ⚠️ 5. Spell Symbols & Their Effects

Spells are a "super-hex of 7 hexes with Symbols mapped to coordinates" and the Apply Spell pipeline says "apply the corresponding Symbol FX." But:

- What Symbols exist?
- What is each Symbol's FX? (Damage? Heal? Apply Lock/Empower/Weaken?)
- Are Symbols defined in Content FS-1 (referenced but not included)?
- Can a single cell have multiple symbols applied?

---

### ⚠️ 6. Hex Geometry — Party (5) vs Spell (7) Mapping

The Party is a 5-hex model. The Spell is a 7-hex super-hex. The Apply Spell pipeline says each symbol's coordinate matches "a cell in the target Zone."

- How does a 7-hex grid overlay onto a 5-hex grid?
- Do 2 cells always miss / overflow? Or does the 7-hex contain the 5-hex plus 2 empty border cells?
- Is there a specific coordinate system shared between both?

---

### ❓ 7. FSM State Lifecycle — AFTERMATH Is Undefined

The FSM lists five states but only four have clear roles:

| State      | Role                                        |
|------------|---------------------------------------------|
| SETUP      | Run Setup Pipeline, transition to COMBAT     |
| COMBAT     | Round Loop executes                          |
| AFTERMATH  | ??? — mentioned as where "next game state" is yielded |
| SPELLCAST  | Entered via REVERT, apply spell              |
| ENDGAME    | Show score                                   |

**Options:**
- **(A)** AFTERMATH is the "waiting" phase between rounds (player sees result, chooses CONFIRM or REVERT). The round loop lives across COMBAT↔AFTERMATH.
- **(B)** AFTERMATH is a brief processing state that resolves animations/logging before returning to COMBAT.
- **(C)** COMBAT and AFTERMATH are effectively one state and the distinction is only about UI blocking (input locked during COMBAT, unlocked during AFTERMATH).

---

### ❓ 8. Turn Order — "Fastest Unit" Semantics

"Party that has the fastest unit goes first." This implies a Speed-like parameter but raises questions:

**Options:**
- **(A)** Each unit has a `Speed` param. Compare the single highest Speed across both parties. That party acts first. (Simple, one comparison.)
- **(B)** Compare average speed of alive units in each party.
- **(C)** Speed determines unit order *within* a party too (not just party-first). Units across both parties interleave by speed.

> Option C would fundamentally change the loop structure (L2 would merge into L1). The spec's current nesting implies A or B.

---

### ❓ 9. Backup State — When Is It Created?

CONFIRM "overwrites the backup state variable with current state." REVERT restores it. But the spec doesn't say **when the backup is first created**.

**Options:**
- **(A)** Backup is created at the start of each Round (step 1 of Round Loop), before any combat.
- **(B)** Backup is created when CONFIRM is issued (i.e., the confirmed state becomes the backup for the *next* round's revert).
- **(C)** Both — initial backup at round start, then CONFIRM refreshes it.

> The CONFIRM description ("overwrites backup with current state") leans toward **(B)**: confirming the round saves it as the new revert point.

---

### ❓ 10. Empower & Weaken — Mechanical Effect

Listed as temp effects cleared each round, but their actual gameplay effect is never described.

**Options:**
- **(A)** Empower increases outgoing damage by a flat or multiplied amount; Weaken decreases it.
- **(B)** Empower/Weaken modify the Coin Stack odds or consumption.
- **(C)** They modify Protection or Damage Reduction values temporarily.

> These are likely defined in Content FS-1 (which is referenced but not provided).

---

### ❓ 11. Stats Data Object — Undefined

Referenced by the FINISH command and Endgame Pipeline ("compute and display Score") but the spec never says what Stats tracks.

- Total damage dealt?
- Rounds survived?
- Spells used?
- Units killed?

---

### ❓ 12. "Finished" Flag — When Is It Set?

CONFIRM checks "if GameState finished flag is true" but the flag's trigger isn't stated explicitly.

**Likely answer:** It's set when the exit condition is detected (one party has no alive units), but this should be stated clearly in the spec — specifically *where* in the loop it's checked and set.

---

### ❓ 13. LOCKED Status — How Is It Applied?

DEAD is clearly applied by the Damage Pipeline. LOCKED is removed each round as a temp effect, but **nothing in the spec applies it**. Presumably a Spell Symbol FX, but this needs confirmation.

---

### ❓ 14. Unit-to-Hex Mapping — Multi-Cell Units

"Each unit is mapped to a set of one or more unique hex cells." This implies some units span 2+ hexes.

- Does occupying multiple cells affect targeting (can a multi-cell unit be targeted once or per cell)?
- Does it affect spell application (symbol hits one cell of a multi-cell unit)?

---
