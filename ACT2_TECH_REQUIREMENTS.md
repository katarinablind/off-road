# Act 2 — Technical Requirements

Companion to `ACT2_PRD.md`. Each requirement is testable and is referenced by ID in `ACT2_AUDIT.md`. Target file: `index.html`.

## TR-1. Avatar-Static / World-Active

**TR-1.1** The kayak's screen position is determined by `viewTop = k.scrollY - K_HEAD_OFFSET` where `K_HEAD_OFFSET` is a small constant (~0.15). The kayak's `(k.px, k.scrollY)` is the world position; the renderer maps it to screen via `toScreen()`.

**TR-1.2** Input (WASD / arrows) mutates `k.vx` and `k.scrollY`, never a screen-space coordinate.

**TR-1.3** No code path in Act 2 reads `mouseX/mouseY` (or any DOM mouse event handler) for field, contour, warp, proximity, or audio updates.

## TR-2. Field Sampling

**TR-2.1** `sampleWaterField(nx, s, flowOffset) -> number` is deterministic from its arguments. It does not read kayak state.

**TR-2.2** A separate `applyKayakWarp(value, nx, s, kayak)` function (or equivalent inlined logic) is the **only** place the kayak's position influences a sampled value. Currently the kayak does not influence the field at all — adding this is fine, but it must be in one identifiable place.

## TR-3. Sample Grid + Bilinear Upscale

**TR-3.1** Per frame, sample a low-resolution grid `field[r][c]` of size `(COLS+1) × (ROWS+1)` with `ACT2_COLS = 80, ACT2_ROWS = 100`. The current `240 × 180 = 43,200` samples is ~5× over budget; reduce to `80 × 100 = 8,000` to match the minimap.

**TR-3.2** Track `hMin` and `hMax` over the sampled grid each frame.

**TR-3.3** Render filled bands by drawing the low-res grid into an `OffscreenCanvas` of size `COLS × ROWS`, then `drawImage`-ing scaled-up to full canvas with smoothing enabled. (Currently uses a regular `<canvas>` cached on the function — works, but should match the minimap's `OffscreenCanvas` pattern for consistency.)

## TR-4. Adaptive Contour Levels

**TR-4.1** Contour iso levels are computed from `[hMin, hMax]` each frame, **not** hardcoded:
```js
const NUM_BANDS = 14;
const levels = [];
for (let i = 1; i <= NUM_BANDS; i++) {
  levels.push(hMin + (i / (NUM_BANDS + 1)) * (hMax - hMin));
}
```
This must replace the current `WATER_LEVELS = [.05, .10, .16, ..., .52]`.

**TR-4.2** Marching squares runs against the same `field` grid that produced the bands.

**TR-4.3** Stroke style per level matches the off-road treatment: stroke alpha and weight ramp slightly with band index `t = level / NUM_BANDS`, mirroring `updateMinimap()`.

## TR-5. Kayak Warp Halo

**TR-5.1** The scalar `k.warp` (already defined in `makeKayak`) is updated each frame in `updateKayak`:
- `+= shoalFactor * dt` when sampled water value crosses the rock threshold
- `+= edgeFactor * dt` when `edgeDist > hw * 0.85`
- `+= turbFactor * dt` when local gradient magnitude is high (rapids)
- `-= recoveryRate * dt` otherwise (calm water)
- Clamped to `[0, 1]`.

**TR-5.2** When drawing contour line segments inside a screen-space radius `R_warp` of the kayak, segment endpoints are jittered by `noise2d(x, y) * k.warp * WARP_AMPLITUDE * cellSize` where `WARP_AMPLITUDE ≈ 0.04` — the same constant Act 1 uses in `updateContourLines()`.

**TR-5.3** The warp halo origin is the **kayak's screen position** as computed by `toScreen(k.px, k.scrollY)`.

**TR-5.4** The current per-segment turbulence offset (`turb` inside the marching squares loop, lines ~2270–2279) is replaced or augmented by the warp halo. The warp halo is the player-driven distortion; the turbulence offset can remain as a global rapids effect.

## TR-6. Input

**TR-6.1** WASD keys are read via `keyDown(idx, n)` where `n ∈ {0:W, 1:S, 2:A, 3:D}`. This is already in place.

**TR-6.2** Mouse is not read by Act 2 at all.

## TR-7. Detail Markers (Shores)

**TR-7.1** A scattered dot pass runs once per frame for cells whose center falls **outside** `isInRiver(nx, s)`. Density and tone modulated by sampled value, deterministic from `(nx, s)` so dots stay in place when scrolling (use a hash of integer-rounded world coords, not `Math.random()`).

**TR-7.2** The current flat shore fill `topoCtx.fillStyle = '#3a3228'` (line ~2157) becomes the *base* color, with detail dots drawn on top.

**TR-7.3** Detail dots are drawn before contour lines so contour lines stay legible above them.

## TR-8. Visual Cohesion with Act 1

**TR-8.1** The Act 2 band palette must be replaced. Currently:
```js
const BAND_COLORS = ['#081428','#0c1e3a','#102a48','#163656','#1c4462','#24526e','#2e627a','#3a7488'];
```
This is candy blue and breaks continuity with the warm Act 1 mountain. The replacement palette must:
- Live in the same module-level constant location
- Read as continuous with Act 1's terrain colors (sample the warm earth tones used in Act 1's terrain mesh material / minimap)
- Allow a cool tint for the deepest 2–3 bands (deep water reads as cooler) but anchor the rest in warm earth/forest tones

**TR-8.2** Contour line color (currently `LINE_COLOR = 'rgba(100,210,255,0.6)'`) must shift to a warmer luminous tone consistent with Act 1.

**TR-8.3** The page background `document.body.style.background = '#0a1628'` set in `launchAct2()` must change to a warm dark tone matching Act 1's atmosphere.

## TR-9. Performance

**TR-9.1** With grid at `80 × 100 = 8,000` samples, `renderAct2River()` should drop from its current load (~5× heavier) to comparable to the minimap's per-frame cost.

**TR-9.2** Marching squares pass and band fill share the same `field` grid (already true).

**TR-9.3** The whitewater foam pass (lines ~2300–2321) iterates the full grid. After grid reduction this becomes 8,000 cells instead of 43,200; no further optimization needed.

## TR-10. Visual Tokens

**TR-10.1** The following constants must exist in one place near the top of the Act 2 section, named consistently:

```js
const ACT2_COLS = 80;
const ACT2_ROWS = 100;
const NUM_BANDS = 14;
const WARP_AMPLITUDE = 0.04;
const WARP_HALO_RADIUS = 180;     // px in topo canvas space
const WARP_RECOVERY = 0.4;        // per second
const K_HEAD_OFFSET = 0.15;
```

**TR-10.2** Palette arrays (`BAND_COLORS`, `LINE_COLOR`, `ROCK_COLOR`, shore base) all live next to each other and are clearly labeled.

## TR-11. Verification Hooks

**TR-11.1** A `?act2debug=1` query param overlays:
- `k.scrollY`, `k.vx`, `k.warp`
- A red dot at the kayak's screen position
- The current `[hMin, hMax]`
- Number of contour segments drawn

**TR-11.2** Setting `WARP_AMPLITUDE = 0` must still render a coherent topology — proves the field is independent of the avatar.

**TR-11.3** Holding `S` (back-paddle) for 2s must visibly scroll the river *upstream* and the kayak marker must not move on screen.
