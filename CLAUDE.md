# ACCEL DRIV — Claude Code Project Guide

## What This Is
A Three.js track editor ("Track Suite") with an integrated racing game ("Horizon Drive"), deployed via GitHub Pages.
Two-file architecture: `index.html` is the editor (~1116 lines), `game.html` is the standalone game (~1794 lines).

**Live URL:** https://phdev.github.io/accel-driv/

## Push & Merge Workflow

There are **no PRs**. Push directly to `main` using two sequential pushes after every commit:

```bash
# 1. Push to feature branch (standard)
git push -u origin claude/accelerometer-game-controls-rG6UR

# 2. Push to main via PAT (this IS the "merge")
git push https://x-access-token:<PAT>@github.com/phdev/accel-driv.git claude/accelerometer-game-controls-rG6UR:main
```

The `<branch>:main` syntax pushes the local branch onto remote `main`. The PAT provides write access. GitHub Pages deploys from `main` at root `/`.

**PAT is stored in the Claude Code memory file** (`MEMORY.md`) — not committed to the repo. Claude Code sessions will have it automatically.

**After both pushes succeed, say: "pushed and merged"**

## Tech Stack
- **Three.js r128** (CDN) — editor uses basic renderer, game uses EffectComposer + UnrealBloomPass
- **Spark.js** (CDN) — Gaussian splat rendering in the editor's Splat tab
- **World Labs Marble API** — AI-powered 3D environment generation
- **No build step, no bundler**
- **GitHub Pages** — deployed from `main`, root `/`

## Files
- `index.html` — Track Suite editor (track editing, terrain, gameplay config, Play mode, splat viewer)
- `game.html` — Standalone Horizon Drive racing game (rendering, physics, AI, controls, HUD)
- `track.json` — 3000 spline points defining the default track shape
- `track_backup.json` — Backup copy of the default track
- `base_basic_shaded.glb` — Car model (~2400 tris), used in the game
- `base_basic_pbr.glb` — PBR version of the car model

## Architecture Overview

### Editor (`index.html`)

The landing page is a track editor with 5 tabs:

**Track tab** — Load/export tracks, edit track points, control points, spline manipulation, smoothing, elevation offset, loop creation, image-to-track conversion, per-control-point width editing

**Terrain tab** — Procedural terrain generation using seeded simplex noise (configurable amplitude, frequency, octaves, lacunarity, persistence, seed). Track-aware blending (clearance, blend radius, depression). Vertex-colored height map with 4-color gradient.

**Game tab** — Place speed pads (count, boost multiplier, length) and AI cars (count, spacing, grid columns, start offset). Export full scene as JSON.

**Play tab** — Launches the racing game inline via an iframe. The editor dynamically generates a standalone HTML document (via `buildGameHTML()`) containing the full game engine, injects the current track data, and loads it as a blob URL. Press ESC or exit button to return to the editor.

**Splat tab** — World Labs Marble API integration for AI 3D environment generation (text or screenshot+text prompts). Spark.js-based Gaussian splat viewer supporting .spz, .ply, .splat, .sog formats. Splat opacity and scale controls.

**Editor 3D viewport** — Three.js r128, no post-processing. Orbit/pan/zoom controls. Gizmo cube overlay. Grid helper. Mobile-responsive sidebar drawer.

### Game Engine (embedded in Play tab / `game.html`)

**Rendering** — Three.js r128 with UnrealBloomPass post-processing (threshold 0, strength 1, radius 0.1, exposure 1). ACES filmic tone mapping. FogExp2 atmosphere. Neon-lit track with emissive barriers.

**Track & Movement** — `CatmullRomCurve3` spline (Y/Z swapped for Three.js coords). Rail mode (default): car follows spline via `curve.getPointAt(t)` with lateral offset. Free-roam mode available via checkbox.

**Car & Physics** — GLB model auto-centered and scaled to 8 units. Auto-gas with manual braking. Speed capped at 270 km/h. Drift state machine: GRIP ↔ DRIFT with assist stabilization, speed bleed, exit snap boost. 15° normal yaw, up to 45° during drift.

**AI** — 5 opponents with staggered speeds (165-248 km/h), road weaving, boost pad usage. Box geometry placeholders with unique hue-shifted colors.

**Boost Pads** — 40 pads placed along track, 1.5s boost to 390 km/h.

**XP System** — Continuous XP during drift, discrete awards for boosts (+200) and passing AI (+200). Multiplier increments on events, resets after 3s idle.

**Controls** (priority order):
1. External API (`window.AccelDriv`) — for web wrapper integration
2. Keyboard (arrow keys / WASD)
3. Touch
4. Accelerometer (DeviceOrientation `gamma`)
   - iOS: requires user gesture for permission, activates after 3 samples
   - Android: activates after 20 samples averaging >5° (filters false positives)
   - Deadzone: 3° normal, 8° free-roam

**HUD** — Speed, time, progress %, position (1st-6th), boost indicator, drift indicator, XP display with multiplier. Tilt indicator bar. Finish screen with time/placement/XP.

**External API** (`window.AccelDriv` in `game.html`):
- `setSteer(v)`, `setBrake(v)`, `setDrift(v)`, `releaseAll()`
- `start()`, `restart()`
- `getState()`, `setFreeRoam(v)`, `getCFG()`

## Key Implementation Details

### Play Mode Game Generation
The editor's Play tab generates a complete standalone HTML game at runtime via `buildGameHTML()`. Track data is serialized as JSON and injected into the generated HTML. The game runs in an iframe using a blob URL. GLB assets are loaded from the GitHub raw URL.

### Image-to-Track Pipeline
1. Upload image → grayscale conversion → Otsu thresholding
2. Distance transform (BFS) → ridge detection (local maxima)
3. Nearest-neighbor chain ordering → uniform resampling → smoothing
4. Auto-loop closure if endpoints are close enough

### Terrain Generation
Seeded simplex noise with configurable FBM (fractional Brownian motion). Track-aware height blending: clearance zone forces terrain to track height + depression, smooth hermite blend in transition zone.

### Drift Visuals
- Normal steering: car yaws up to 15°
- Drifting: car yaws up to full `driftAngle` (max 45°)
- Drift direction follows gyro/steer input in real-time
