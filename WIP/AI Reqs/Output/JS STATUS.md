 build.sh already concatenates all 13 modules (00–12) into dist/game.html. It's fully buildable and runnable today — you can open game.html in a browser right now. The game loop works, spell casting works, and the log div  
  shows all events. What you get is a text-only debug view on a canvas with a unit list.

  What the game logic side is missing

  Technically, modules 00–10 (pure logic) are complete. The gaps are in the integration layer:

  12_main.js — line 42:
  // TODO: Let player choose target zone (ALLY/ENEMY)
  // For now, default to ENEMY
  game.castSpell(i, 'ENEMY');
  All 4 spells are currently hardcoded to target ENEMY. Healing Light on ally party is unreachable. This is a functional logic gap — the targetZone parameter is wired up in the engine, just not exposed in the UI.

  What a Phaser frontend would need

  The contract between the engine and any renderer is already defined:

  ┌───────────────────────────────┬───────────────────────────────────────────────────┐
  │         Engine output         │             What a renderer consumes              │
  ├───────────────────────────────┼───────────────────────────────────────────────────┤
  │ graphicEvents: [[...], [...]] │ Parallel animation groups, played sequentially    │
  ├───────────────────────────────┼───────────────────────────────────────────────────┤
  │ onPhaseChange(phase, state)   │ Phase transitions (AFTERMATH, SPELLCAST, ENDGAME) │
  ├───────────────────────────────┼───────────────────────────────────────────────────┤
  │ renderState(state, canvas)    │ Full state snapshot for the scene                 │
  └───────────────────────────────┴───────────────────────────────────────────────────┘

  11_renderer_stub.js is the only file to replace. The interface stays the same.

  Proposed steps to make it user-testable

  Step 1 — Fix target zone selection in 12_main.js
  When a spell can target either zone, show two buttons ("Cast on Allies" / "Cast on Enemies"). This is 10–15 lines and unblocks Healing Light and Displacement on ally party. This is the only game-logic-adjacent gap.        

  Step 2 — Draw the hex grid on canvas (still no Phaser)
  Replace the text dump in renderState() in 11_renderer_stub.js with Canvas 2D hex cells. The 12 cell positions, two parties, and unit icons can be drawn with fillRect / arc. No animation yet — just a static snapshot that   
  updates after each action. This alone makes the game readable as a playable thing.

  Step 3 — Add event replay in playEvents()
  The graphicEvents array is already structured as [[parallel], [parallel]] — designed for sequential animation. The simplest non-Phaser approach: step through event groups on a timer or on click, highlighting the relevant  
  cells. Even just coloring attack/hit/death events differently makes it legible.

  Step 4 — Phaser renderer (replaces steps 2–3)
  If going to Phaser, change build.sh to load Phaser from CDN in the HTML head, then rewrite 11_renderer_stub.js as a Phaser scene. The graphicEvents groups map directly to tween chains. renderState maps to scene refresh.   
  12_main.js boots Phaser.Game instead of just wiring buttons.

  ---
  Shortest path to something user-testable without Phaser:
  Steps 1 + 2 = a playable game where you can see the grid, read unit states, cast spells at either party, and follow the log. Probably 2–4 hours of work. The game logic is fully done and won't need touching.