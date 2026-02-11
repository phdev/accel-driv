# ACCEL DRIV — Claude Code Project Guide

## What This Is
A standalone Three.js racing game built from scratch, deployed via GitHub Pages.
Single-file architecture: everything lives in `index.html` (~1970 lines).

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
- **Three.js r128** (CDN) with EffectComposer, UnrealBloomPass, ShaderPass, GLTFLoader
- **Single index.html** — no build step, no bundler
- **GitHub Pages** — deployed from `main`, root `/`

## Files
- `index.html` — The entire game (rendering, physics, UI, shaders, controls)
- `track.json` — 3000 spline points defining the track shape
- `base_basic_shaded.glb` — Car model (~2400 tris), used by player and all AI cars

## Architecture Overview

### Rendering Pipeline
- UnrealBloomPass post-processing (threshold 0, strength 1, radius 0.1, exposure 1)
- **Frame generation system** (platform-adaptive):
  - **Android**: Interpolation — renders 15-30fps real frames, synthesizes to 30fps via camera reprojection shader
  - **iOS**: Extrapolation — renders at GPU speed, synthesizes toward 120fps using camera motion prediction
  - **Desktop**: Off — renders normally
  - Uses dual WebGLRenderTargets + GLSL reprojection shader with camera matrix delta

### Track & Movement
- `CatmullRomCurve3` spline from track.json data (Y/Z swapped for Three.js coords)
- **Rail mode** (default): car follows spline via `curve.getPointAt(t)`, lateral offset for steering
- **Free-roam mode**: car moves independently, two-pass nearest-t search for ground height, tangent-based Y interpolation, track-edge collision clamping

### Car & Physics
- GLB model auto-centered and scaled to 8 units
- Auto-gas with manual braking
- Drift state machine: GRIP ↔ DRIFT with assist stabilization, speed bleed, exit snap boost
- Continuous quad-strip skid marks per wheel during drift
- 5 AI opponents with staggered speeds, road weaving, boost pad usage

### Controls (priority order)
1. External API (`window.AccelDriv`) — for web wrapper integration
2. Keyboard (arrow keys / WASD)
3. Touch
4. Accelerometer (DeviceOrientation `gamma`)
   - iOS: requires user gesture for permission, activates after 3 samples
   - Android: activates after 20 samples averaging >5° (filters false positives)
   - Deadzone: 3° normal, 8° free-roam

### UI
- HUD: speed, time, progress %, position, boost indicator, drift indicator
- Debug panel (top-right): bloom sliders, DPR slider, FPS counters (real/display/synth%), frame gen mode
- Tilt indicator bar at bottom
- Free-roam checkbox, drift button
- No start screen — game auto-starts with 3-2-1-GO countdown

### External API (`window.AccelDriv`)
- `setSteer(v)`, `setBrake(v)`, `setDrift(v)`, `releaseAll()`
- `start()`, `restart()`
- `getState()` — includes frameGen stats
- `setFreeRoam(v)`, `getCFG()`

## Key Implementation Details

### Frame Generation Render Order
On Android, FG target capture happens BEFORE `composer.render()` to avoid framebuffer rebind issues on some WebGL implementations.

### Free-Roam Ground Tracking
Two-pass nearest-t search (coarse 0.002 step, fine 0.0002 step) with warm-start from `freeNearestT`. Ground height uses tangent slope interpolation + 25% lerp smoothing (`freeSmoothY`).

### Drift Visuals
- Normal steering: car yaws up to 15°
- Drifting: car yaws up to full `driftAngle` (max 45°)
- Drift direction follows gyro/steer input in real-time

## Progress / Completed Features
- Three.js scene with neon-styled track, barriers, ground plane, fog
- GLB car model for player and AI (with emissive materials)
- CatmullRomCurve3 track spline from 3000 data points
- Bloom post-processing with debug sliders
- Frame generation (Android interpolation, iOS extrapolation)
- DPR (device pixel ratio) runtime slider
- Accelerometer steering (iOS permission-gated, Android auto-detect)
- 5 AI opponents with unique colors, staggered speeds, boost pad usage
- 40 boost pads with visual pulse animation
- Drift system with state machine, skid marks, visual yaw
- Free-roam mode with ground tracking, track-edge collision
- Camera system with lerped position/target, downhill tracking
- Player position tracking (1st-6th)
- Finish screen with time and placement
- External API for web wrapper control
- Auto-start (no title screen)
