# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Hackman is a password-cracking game with an outrun / Tron / Matrix identity. The entire project is one self-contained file, `hackman.html` — markup, CSS, engine, game, and word list. There are no dependencies, no build step, no test suite, no linter, and no package manager. The directory is not a git repository.

Everything must stay in that one file. All effects are procedural (gradients, grid math, particles, synthesized audio), so there are currently no embedded assets; if you ever need one, it has to be a base64 data URI.

## Running and verifying

Open `hackman.html` directly in a browser (`file://` works; nothing is fetched over the network). To serve it instead:

```bash
python3 -m http.server 8000   # then open http://localhost:8000/hackman.html
```

There is nothing to build or watch — reload after an edit.

**There is no test suite, so verify visually with headless Firefox.** Node is not installed on this machine; Firefox is. A plain screenshot of the page is useless because it fires before the first animation frame, so drive the game into a known state first. The pattern that works:

1. Copy the file, strip `onload="startup()"` from the `<body>` tag, and append a `<script>` at the end of `<body>` that calls `startup()`, mutates game state directly, then calls `game.canvas.refresh()` once to force a synchronous frame.
2. Push timers into the past to skip animations — `engine.Timer.prototype.start` takes a millisecond modifier, so `game.menu_timer.start(-2000)` reports the entrance as long finished.
3. Screenshot it:

```bash
firefox --headless --profile "$PWD/ffprofile" --window-size=1600,900 \
  --screenshot "$PWD/out.png" "file://$PWD/harness.html"
```

The profile directory must already exist and be an absolute path, or Firefox hangs until it is killed. Register a `window.addEventListener("error", ...)` handler in the harness that appends the message to the DOM — a script error otherwise shows up as nothing but a blank canvas.

The same harness shape works for assertions: run checks synchronously, append a `<pre>` with the results, and screenshot that. Worth checking after changes to word selection: inject a few thousand synthetic entries into `wordlist`, call `game.buildWordIndex()`, and confirm the length bound and password shape still hold.

## Architecture

Three global objects in the single `<script>` block, in this order: `game`, `engine`, and `wordlist`. `engine` is a self-contained 2D canvas engine that knows nothing about Hackman; `game` is the entire application on top of it. Keep that separation — engine code stays reusable, game rules stay in `game`.

### Design intent

Each of the three visual references owns exactly one layer. This is the load-bearing idea behind the look, and it is not recoverable from reading the code:

- **Outrun owns the world** — gradient sky, the sliced sun, the scrolling perspective grid. Background only.
- **Tron owns the chrome** — every HUD element is an angular hairline bezel with corner ticks and hard glow. Zero corner radius, structure never decoration.
- **Matrix owns data in flux** — the scramble-on-reveal, the rain, the corruption. Text that moves is Matrix; text that sits still is Tron.

The palette is semantic, not decorative: `CYAN` is system and UI, `PHOSPHOR` is the player's progress, `MAGENTA` is the trace closing in, `AMBER` is time pressure. Do not use a colour outside its job.

The signature element is the decode cell. A correct guess makes each matching cell cycle glyphs for ~280ms, then snap to its letter with a bracket flash, a scale pop (`engine.easeOutBack`), and a particle burst. Everything else stays quiet so that moment lands.

### Engine

`engine.Canvas` owns the render loop. Consumers assign the callback properties (`eventGameLoop`, `eventKeyDown`, and the mouse events) and call `start()`; `refresh()` then runs on `requestAnimationFrame`, clears to black, and invokes `eventGameLoop(dt)`. Rendering is **immediate mode** — nothing persists between frames, so anything visible must be redrawn every frame.

Things that constrain callers:

- **Fixed virtual resolution.** The game space is 1920×1080 (`game.GAME_W` / `game.GAME_H`) letterboxed into the window. `resize()` computes a single uniform scale from the smaller axis so nothing stretches, plus a centring offset. Author all layout in game units and ignore the window size.
- **One transform for offset and shake.** `refresh()` folds the letterbox offset and the screen shake into a single `setTransform`, so no draw call needs to know about either. Anything that needs raw device pixels (`scanlines`, `vignette`, `applyGlitch`) resets the transform itself and restores after.
- **Text is rasterised at device scale.** `drawText` and `drawGlyph` cache to offscreen canvases with the scale baked into the font size, which keeps text crisp at any window size but means both caches are cleared on every resize. `drawText` caps its cache and drops it wholesale past ~420 entries, because per-frame dynamic strings like the score would otherwise grow it without bound.
- **Vertical alignment is inconsistent between the text calls.** `drawText` positions from the top of the text; `drawGlyph` and `drawTracked` centre vertically on the given point. This trips up layout maths — check which one you are using before nudging a `y`.
- `engine.ParticleSystem` is a fixed pool (700) reused rather than allocated per burst. `engine.Audio` synthesises every sound with WebAudio, lazily constructing its context on the first keypress to satisfy autoplay policy.

### Game

A three-phase state machine — `PHASE_MENU`, `PHASE_PLAY`, `PHASE_RESULT` — switched on in both `game.loop` and `game.keypress`. A new phase needs a case in each. `game.phase_transitioning` is set on every phase change and cleared at the top of the next frame; `keypress` returns early while it is set, which is what stops one keystroke being consumed by two phases. Preserve it.

`game.init()` resets per-round state. Fields declared in the `game` object literal — the palette, `tiers`, the scoring constants, `team_score` — sit outside it deliberately so they survive across rounds. A new per-round field must be added to `init()`.

Row positions (`cell_row_y`, `trace_row_y`, `key_row_y`) are set in `init()` and shared between drawing and particle spawning, so `game.burstAt` lands on the same cells that `game.drawCells` drew.

`game.finish()` **must freeze the clock before setting `game.outcome`** — `timeLeft()` starts returning `frozen_time` the moment an outcome exists, so the reverse order silently zeroes every time bonus. It then runs a ~1.5s outro beat in `PHASE_PLAY` before handing over to the result screen.

**Corruption drives the whole world.** A single `game.corruption` value (wrong guesses ÷ the tier's allowance) feeds the sky's red shift, rain density, grid colour and jitter, scanline strength, and vignette pulse. The fail state is *detection*, so pressure should look like the system noticing you. Add new pressure effects by reading `corruption` rather than by counting wrong guesses again.

**Panels must stay near-opaque.** They sit over a bright sun and a busy grid; below roughly 0.9 fill alpha the sun bleeds through and turns the plate muddy brown. The password row additionally gets a full-width console strip behind it in `drawCells` so it reads regardless of what is behind.

Scoring is `100 × occurrences × combo × tier multiplier` per hit, with end-of-round bonuses for time left and unused attempts. Keep totals in the thousands — an earlier version multiplied by `50 × (difficulty+1)^(difficulty+1)` and produced unreadable nine-figure scores.

### Dictionary

`wordlist` is a plain object of `"word": "definition"`, and **its shape is a contract** — a much larger list gets pasted in wholesale, so keep the same variable name, the same object literal, one entry per line, and lowercase keys.

Nothing reads it directly. `game.buildWordIndex()` derives a filtered array once at startup (lowercase, letters only, 4–9 characters) and word selection goes through that. The length bound is what keeps a full dictionary from producing 20-character words; do not bypass it.

`game.cleanHint` keeps the whole definition — it only collapses whitespace and drops a dangling leading `1.` when the entry has a single sense, since with two or more the numbering is what separates them.

Definitions are then laid out by `game.buildHintLayout()`, called once per round from `startRound` because wrapping is the only costly part of drawing them. It steps down through `game.HINT_SIZES` until the wrapped lines fit the band between the HUD and the password strip, so the panel grows to its content instead of the text being cut. Smaller type wins twice — more lines fit the band *and* more characters fit each line — so one step down roughly doubles capacity. Against the shipped 4,539-word list, 98.9% of rounds render at the full 20px and nothing truncates; the last-resort ellipsis only exists because the dictionary is editable. Adjusting `HINT_BOTTOM` past about 405 will collide with the password strip, whose top edge sits at 415.

## Conventions

Pre-ES6 style throughout: `"use strict"`, `var` only, function expressions on prototypes, JSDoc on every engine method, and bracket notation for canvas context properties (`context["fillStyle"]`). Match it rather than modernizing piecemeal. Layout is pixel maths in the 1920×1080 game space, positioned by mutating a reused `engine.Point` between draws.
