# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla-JS Tetris (HTML5 Canvas, no dependencies). Three static files — `index.html`, `style.css`, `game.js` — with no `package.json`, bundler, linter, or tests. The README and user-facing UI strings are in Spanish; match that when adding visible text or docs.

## Running

There is no build step. Open `index.html` directly, or serve the directory:

```bash
python3 -m http.server 8000   # then visit http://localhost:8000
```

Verification is manual in the browser (there is no test runner, so no single-test command).

## Architecture

All game logic lives in `game.js`, a single non-module script that `index.html` loads at the end of `<body>`. It grabs DOM elements by id at load time, so any id renamed in `index.html` must be updated in the `getElementById` block at the top of `game.js`.

State is a set of module-level globals (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropAccum`, `dropInterval`, `animId`, …) reset by `init()`. Two things mutate this state concurrently: the `requestAnimationFrame` `loop()` (gravity + `draw()`) and the `keydown` listener (move/rotate/drop/pause). Both route piece-locking through `lockPiece()` → `merge()` → `clearLines()` → `spawn()`, and `spawn()` calls `endGame()` when the new piece collides immediately.

Conventions worth knowing before editing:

- Board cells hold `0` (empty) or a piece-type index `1–7`. That same index selects the entry in `COLORS` and `PIECES` (both have `null` at index 0), so adding a piece means extending both arrays in lockstep and updating `randomPiece()`'s hard-coded `7`.
- Rotation is clockwise only (`rotateCW` on the shape matrix). `tryRotate()` does simple horizontal kicks `[0, -1, 1, -2, 2]`, not SRS.
- `collide(shape, ox, oy)` reads the global `board` and treats `ny < 0` as free, so pieces can sit above the visible top.
- Rendering is a full redraw each frame (`draw()`: grid → board → ghost at alpha 0.2 → current piece). `drawNext()` only runs from `spawn()`.

## Gotchas

- Canvas size is duplicated: `index.html` hard-codes `width="300" height="600"` for `#board` (and `120×120` for `#next-canvas`), while `game.js` derives drawing from `COLS`/`ROWS`/`BLOCK` (and a local `NB = 30` in `drawNext`). Changing the grid constants requires editing the HTML attributes too.
- The speed curve is duplicated and unnamed: the `1000` initial interval is set in `init()` and the formula `Math.max(100, 1000 - (level - 1) * 90)` is inlined in `clearLines()`. The README lists `dropInterval` as a tunable constant, but it is not one — change both places.
- `endGame()` calls `cancelAnimationFrame(animId)`, but when it is triggered from inside `loop()` (gravity lock) the loop still reaches its trailing `requestAnimationFrame(loop)`, so the loop keeps running behind the Game Over overlay. Hard/soft drop from `keydown` stops correctly. Keep this in mind if you touch game-over or pause flow.
- Pause resumes by calling `loop()` directly after resetting `lastTime`; restart is `init()`, which also cancels any pending frame.
