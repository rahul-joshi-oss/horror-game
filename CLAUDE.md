# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

No build step required. Open `index.html` directly in a modern browser (Chrome, Firefox, Edge). The only external dependency is Three.js r128, loaded from CDN at runtime.

There is no package.json, no npm, no TypeScript, no linting, and no test framework. All game code lives in a single file: `index.html`.

## Architecture

The entire game — HTML, CSS, audio, AI, rendering, and game logic — is contained in `index.html` (~2,664 lines). It is a first-person 3D horror game ("DESCENT") built on Three.js with procedurally generated floors, a stealth/avoidance entity, and a terminal hacking minigame for progression.

### Code Sections (by line range)

| Lines | Section |
|-------|---------|
| 1–158 | HTML structure, CSS, UI overlays (HUD, minimap, modals) |
| 159–197 | **`C` object** — all tunable game constants (speeds, ranges, grid size, etc.) |
| 200–233 | **`DIFFICULTY`** presets — Easy/Normal/Hard override values applied over `C` |
| 243–290 | **`gs` object** — global mutable game state (floor, keycards, entity state, player pos/rot) |
| 294–464 | **`Audio` class** — all sounds via Web Audio API (procedural synthesis, no audio files) |
| 469–597 | Utility functions: `shuffle`, `gridToWorld`, `worldToGrid`, `dist2D`, BFS pathfinder `findPath` |
| 600–671 | Line-of-sight (`hasLineOfSight`) and tunnel grid generation (MST-based vent network) |
| 673–753 | `generateFloorData()` — procedural maze + object placement (terminals, stairs, vents, tables) |
| 758–792 | `setupScene()` — Three.js renderer, camera, lights, fog |
| 797–1125 | `buildFloor()` — creates all 3D geometry for a given floor |
| 1130–1343 | Entity: `buildEntity()` + `updateEntity()` — AI states (patrol/chase), BFS pathfinding, detection |
| 1359–1476 | `updatePlayer()` — WASD movement, collision, sprint/crouch/stamina, flashlight |
| 1481–1613 | Minimap: `initExplored()`, `updateExplored()`, `renderMinimap()` |
| 1624–1917 | Chase sequence: `buildChaseLevel()`, `updateChase()`, `triggerEscape()` / `winGame()` |
| 1924–1984 | Vent tunnel: `enterVentTunnel()`, `exitVentTunnel()` |
| 1989–2277 | Interaction + terminal minigame: `interact()`, `accessTerminal()`, `renderTerminalGame()`, `handleTerminalInput()` |
| 2279–2383 | Stair transitions (`goUpStairs`, `goDownStairs`) and HUD updates |
| 2388–2498 | Main loop (`animate()`) and entry point (`init()`) |

### Key Data Structures

- **`C`** — singleton config object. All magic numbers live here. Modify this before touching game logic.
- **`gs`** (game state) — single global object tracking floor number, keycard flags, player/entity positions, entity behavior state (`gs.eState`: `'patrol'` | `'chase'`), vent state, chase sequence progress.
- **`floorData[n]`** — per-floor procedural data: maze grid (21×21), terminal/stair/vent/table positions.
- **`floorMeshes[n]`** — Three.js objects per floor; only the current floor is visible.

### Core Game Loop

```
animate() [requestAnimationFrame]
  ├─ updatePlayer(dt)      ← input, collision, stamina
  ├─ updateEntity(dt)      ← BFS pathfinding, detection, floor changes
  ├─ updateChase(dt)       ← active only during final chase sequence
  ├─ updateInteractPrompt()
  ├─ updateExplored() + renderMinimap()
  └─ renderer.render(scene, camera)
```

### Entity AI

The entity has two states in `gs.eState`:
- **`patrol`**: random BFS walk; listens for player noise (range 0–8 units, scaled by movement speed) and checks line-of-sight (range ~10 units, scaled by flashlight state).
- **`chase`**: BFS toward player; interrupted by player entering a vent or hiding under a table while crouching.

Entity aggression increases as more keycards are collected (`gs.keycards`). Detection thresholds and memory duration are set by the active difficulty preset.

### Progression System

1. Player explores each of 5 floors, accesses a **terminal** (WASD/QEFR hacking minigame), collects a keycard.
2. Keycard unlocks the stairwell to the next floor.
3. Final terminal (Floor 5) grants the exit keycard and triggers the **chase sequence**.
4. Chase sequence uses a separate wide-corridor level layout (`buildChaseLevel`); entity follows a fixed accelerating path.
5. Player must descend all 5 floors and reach the exit door to win.

### Audio

All audio is synthesized procedurally in `Audio` class — oscillators, noise buffers, filters, gain envelopes. No audio files are loaded. Chase music uses 3 oscillators with distortion and a pulsing LFO. Ambient drone runs continuously in the background.

### Modifying the Game

- **Tune gameplay feel**: edit values in the `C` object (lines 164–197) or difficulty presets (lines 200–233).
- **Add new floor objects**: extend `generateFloorData()` for placement logic, then render them in `buildFloor()`.
- **New entity behaviors**: add states to `updateEntity()` and set `gs.eState` to trigger them.
- **New interactions**: extend the `switch` statement in `interact()` and add object types in `generateFloorData()`.
- **New sounds**: add methods to the `Audio` class following the existing oscillator/noise pattern.
