# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A from-scratch WebGPU ocean renderer (no engine, no rendering library, no runtime dependencies). Plain ES modules in the browser; Node is only used for the dev server and the bundler. There is no test suite, linter or type checker. Results are checked visually, using the screenshot harness described below.

## Commands

```
npm install        # only dependency is esbuild (used by the minified build)
npm start          # node server.mjs [port], serves http://localhost:8080/ (index.html + js/ modules)
npm run build      # regenerates ocean.html and ocean.min.html
```

`ocean.html` and `ocean.min.html` are committed build outputs. After any change under `js/` or to `index.html`, run `npm run build` and commit the regenerated bundles together with the source. Never edit the bundles by hand. If esbuild is missing, the build still succeeds but prints `esbuild not found` and only strips comments and indentation from the JS, so run `npm install` first. The fallback search in `findEsbuild()` looks only in a Windows npx cache path.

`build.mjs` builds the bundles by concatenating the modules in a fixed order (`common, sim, sky, ocean, post, main`). It does this with regexes: it removes `import ... from` lines and the `export` keyword. This means:
- Imports must be single-statement `import { ... } from '...';` lines, and exports must be `export const|function|async function|let|class`. Other forms will break the bundle.
- All top-level names across all modules share one scope in the bundle, so they must be unique.
- A new module must be added to `wgslModules`/`order` in `build.mjs`.
- The minifier strips comments and whitespace inside **every** backtick template literal in the shader modules. Those files should contain only WGSL template literals (with `${...}` interpolation, which is preserved).

## Architecture

- `js/main.js` contains all host-side code: device and pipeline setup, every GPU resource, the tunable parameters `P` (the settings panel and URL params edit these), the camera, CDLOD quadtree node selection on the CPU (instanced draws), the per-frame encode, and the screenshot readback.
- `js/wgsl/*.js` export WGSL source as JS string constants. Shaders are put together by string concatenation of shared snippets from `common.js` (`FRAME_WGSL`, `ATMO_CORE_WGSL`, `NOISE_WGSL`, `PERIODIC_WGSL`) and `sky.js` (`SKY_RENDER_FUNCS`, `SKY_CLOUD_FUNCS`, which the ocean shader reuses for reflections and the sky background). Sizes such as `N`, `CASCADES`, `ENV_SIZE` are JS constants interpolated into the WGSL.
- Per-frame flow:
  1. Compute: spectrum evolve, then row/column radix-2 FFTs for the 6 cascades, slope-moment mips (LEAN), foam map update, then the atmosphere LUTs (transmittance, multiple scattering, sky-view), the environment cube and its roughness mips, sun visibility, and the water-height probe under the camera.
  2. Render: sky background and ocean surface into HDR with 4x MSAA.
  3. Post: bloom down/up chain, then a composite pass (AgX tone mapping, dither).
- **Uniform layouts are hand-indexed.** `struct Frame` in `common.js` must match the raw `frameF[...]` offsets written in `writeFrame()` in `main.js` (96 floats). The sim uniform block (`simF`, 56 floats) works the same way. If you add or reorder a field, update both sides and keep vec4 alignment.
- The camera height clamp uses an async `mapAsync` readback of `HEIGHT_PROBE`, and GPU timing uses `timestamp-query` when it is available. Both are non-blocking and skip frames while a map is pending.
- The per-frame code avoids allocations by reusing preallocated typed arrays and descriptors. Keep new per-frame code allocation-free in the same way.
- Some header comments are stale. For example, `main.js` says "5 cascades" and `common.js` mentions "FR_* constants" that don't exist. Trust the code over these comments.

## Visual checks / debugging

URL parameters (parsed in `main.js`):
- `w`, `h`: force the render resolution. `cam=x,y,z,yawDeg,pitchDeg`, `fov=deg`, `ui=0` (hide the HUD), `t=seconds` (sim start time).
- Any key of `P` overrides a setting, e.g. `windSpeed=12&swellHs=2&sunElev=8`.
- `dbg=N`: ocean shader debug views (see `ocean.js` near `F.post.w`). 1 = foam/Jacobian, 3 = reflection, 4 = Fresnel, 5 = water body, 6 = foam lighting, 7 = sky-up. 2 or ≥ 8 = world foam map levels plus fresh foam.
- `shot=name.png`: render `frames` frames (default 240) at a fixed dt of 1/60, read back the swap chain, and POST the PNG to the dev server, which writes it to `shots/` (gitignored). `seq=N` saves N consecutive shots. When finished, the page title becomes `SHOT DONE`. `keep=1` keeps rendering afterwards.
- `log()` in `main.js` also POSTs to `/__log`, so client messages and init failures appear in the `npm start` console.

For automated checks, run the dev server and open a `?shot=...` URL in headless Chrome with WebGPU enabled. Wait for the title to change to `SHOT DONE`, then inspect the PNG in `shots/`.
