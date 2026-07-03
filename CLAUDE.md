# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A clone of the classic arcade game **Asteroids**, implemented in pure HTML5 Canvas with vanilla JavaScript (ES6+). No frameworks, no bundler, no dependencies, no build step. The entire game logic lives in a single file: `game.js`.

## Running the game

There is no build/lint/test tooling in this repo. To run the game:

```bash
npx serve .
```

Then visit `http://localhost:3000`. Alternatively, open `index.html` directly in a browser (double-click) since there are no dependencies to install or compile.

Since there's no test suite, verify changes by loading the game in a browser and playing it (see the `run` skill / manual browser testing for UI changes).

## Architecture

`index.html` sets up an `800x600` canvas and loads `game.js` as a plain script (no modules, no `defer`). `game.js` runs top-to-bottom and immediately starts the game loop at the bottom of the file (`initGame(); requestAnimationFrame(loop);`).

Everything in `game.js` operates in a single shared coordinate space of `W=800, H=600` with **toroidal wrap-around** (objects exiting one edge reappear on the opposite edge) via the `wrap(v, max)` utility.

### Structure of game.js (top to bottom)

1. **Input** — `keys`/`justPressed` maps populated by `keydown`/`keyup` listeners; `pressed(code)` consumes one-shot presses (used for shooting and restart) while `keys[code]` gives continuous held-state (used for rotation/thrust).
2. **Utils** — `wrap`, `dist`, `rand`, `randInt`.
3. **Entity classes** — `Bullet`, `Asteroid`, `Ship`, `Particle`. Each has its own `update(dt)` and `draw()`. There's no shared base class or entity/component system — each class independently manages its own physics and rendering; keep this pattern when adding new entity types.
4. **Global mutable game state** — `ship`, `bullets`, `asteroids`, `particles` arrays/objects, plus `score`, `lives`, `level`, and a `state` string that acts as a simple state machine: `'playing' | 'dead' | 'gameover'`.
5. **Update phase** (`update(dt)`) — branches on `state` first. Within `'playing'`, it: reads input to spawn bullets, calls `update(dt)` on all entities, filters out dead entities (`.dead` flag convention), does bullet-vs-asteroid collision (splits asteroids via `Asteroid.split()`, awards points from the `POINTS` table, spawns explosion particles), then ship-vs-asteroid collision (skipped while `ship.invincible > 0`), and finally checks for level completion (`asteroids.length === 0` → `nextLevel()`).
6. **Draw phase** (`draw()`) — clears the canvas, draws particles → asteroids → bullets → ship (back-to-front layering), then HUD (`drawHUD`), then a `drawOverlay` for the game-over screen.
7. **Main loop** — a `requestAnimationFrame` loop computing `dt` in seconds, clamped to `0.05` max to avoid physics blow-ups after tab throttling/pauses.

### Key conventions

- Asteroids have 3 sizes (`3` = large → `1` = small) driving `RADII`, `SPEEDS`, and `POINTS` lookup tables (indexed by size). Destroying a non-smallest asteroid spawns 2 smaller ones via `split()` at the same position.
- Entities signal removal via a `dead` boolean flag rather than being spliced immediately; arrays are cleaned up with `.filter(e => !e.dead)` once per frame.
- The ship has temporary invincibility (`ship.invincible`, seconds) after spawning/respawning, visualized as blinking in `Ship.draw()`.
- UI text and in-code comments are in **Spanish** (e.g. `NIVEL`, `PUNTAJE`); match this convention when touching HUD/overlay text or comments.
- All rendering uses raw Canvas 2D API calls (`ctx.save/translate/rotate/restore` per entity) — no external rendering abstraction.
