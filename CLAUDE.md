# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

FablePhysics: a 2D physics lab in a **single self-contained file**, `fisica.html`. It uses no libraries, no build step and no package.json, and it draws on a canvas. The UI text, code comments and commit messages are in **Italian**, so keep new ones in Italian too. It is also published on GitHub Pages at https://gillus.github.io/FablePhysics/fisica.html.

## Running and testing

- Run it by opening `fisica.html` in a browser. There is nothing to build.
- In the browser console, `window.world` (the `World` instance) and `window.loadScene(name)` are exposed for debugging.
- There is no test suite. The engine script exports itself through `module.exports` when `module` exists, so you can load it headlessly in Node by pulling the `<script id="engine">` block out of the HTML:

```bash
node -e '
const src = require("fs").readFileSync("fisica.html","utf8").match(/<script id="engine">([\s\S]*?)<\/script>/)[1];
const m = { exports: {} }; new Function("module", src)(m);
const { World, Body, Wall, Link, Scenes, PPM } = m.exports;
const w = new World(800, 600); Object.assign(w, Scenes.cradle(w));
const e0 = w.energy().tot; for (let i = 0; i < 240; i++) w.step(1/120);
console.log(e0, w.energy().tot);   // energy drift check
'
```

## Architecture

`fisica.html` has three parts: the CSS/HTML layout, `<script id="engine">` and `<script id="ui">`.

**Engine (`<script id="engine">`)** must stay free of the DOM so that it can still be tested in Node.
- Internal units are **pixels and seconds**, and `PPM = 100` px per metre. Physical parameters are in SI: `gravity` in m/s², `stiffness` in N/m and `damping` in N·s/m. They are converted with `PPM` inside the solver, and `energy()` and `momentum()` return SI values (joules). Keep this conversion in mind whenever you add forces or energy terms.
- `Body` is a disk with mass computed from `π·r²·density` in metres, and inertia `½mr²`. `fixed: true` makes it a pin: infinite mass, `invMass = 0`, and it is excluded from collisions and broadphase. `Wall` is a thick segment (`half` is its half-thickness). `Link` is a spring when `rigid: false` and a distance-constraint rod when `rigid: true`. If `k`/`c` are `null`, the link uses the world's global `stiffness`/`damping`.
- `World.step(dt)` splits `dt` into `substeps` (8) substeps. Each substep does the following, in this order:
  1. Spring forces.
  2. Semi-implicit Euler integration, with gravity, air drag and a `maxSpeed` clamp.
  3. `broadphase()`, which is O(n²) below 40 bodies and a spatial hash grid above that.
  4. `iterations` (6) Gauss-Seidel passes over rods (`solveRod`), then ball pairs (`resolve`), then walls and canvas bounds (`resolveStatic`).
- Contacts use impulses with restitution plus Coulomb friction at the contact point, including spin and torque. Restitution is zeroed below a velocity threshold to avoid jitter at rest. Rolling resistance damps `omega` only while a body touches a wall. Positional correction uses a slop of 0.05 and a factor of 0.6.
- `Scenes.<name>(world, opts)` calls `world.clear()`, builds the scene relative to `world.w`/`world.h`, and **returns suggested globals** (`{restitution, friction, gravity}`) instead of setting them itself. The UI applies them through the sliders. The `collision` scene tags its center balls with `tag: 'center'`, and the UI's "center mass" slider acts on that tag. `mulberry32` is used as a seeded RNG so the scenes are deterministic.

**UI (`<script id="ui">`)** is an IIFE that owns the canvas, the input handling, the drawing and the energy chart.
- The loop is a fixed-timestep accumulator: `FIXED = 1/120`, scaled by the slow-motion `state.speed`, with at most 40 steps per frame. A body being dragged (`state.drag.type === 'grab'`) is held in place across each `world.step`.
- Sliders are wired through `bindRange(id, outId, fmt, apply)`, which writes straight into `world.*` or `state.*`. To set a slider from code, set `.value` and then dispatch `input`, as `loadScene` does.
- To add a scene, you need to update three places: the `Scenes` object, a `data-scene` button in the HTML, and the `names` map in `loadScene`.
- Input: left-click or drag creates a ball, or grabs and throws one. Right-drag draws a wall. Tools switch with `1`–`4` (ball, spring, rod, delete). `Space` pauses, `V` toggles velocity vectors, `Delete` clears the scene and `Esc` cancels.
