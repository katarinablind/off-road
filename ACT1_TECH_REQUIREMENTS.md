# Act 1 — Technical Requirements

Companion to `ACT1_PRD.md`. Each requirement is testable and is referenced by ID in `ACT1_AUDIT.md`. Target file: `index.html`.

## TR-1. Avatar-Static / World-Active

**TR-1.1** The car's screen position is determined by camera tracking — the camera's `(x, y, z)` interpolates toward the car's world position each frame so the car always renders near the same screen offset.

**TR-1.2** Input (WASD / arrows) mutates `c.speed`, `c.heading`, `c.lateralSpeed`, and the shared `scrollOffset`. It never directly mutates the car's screen position.

**TR-1.3** The terrain mesh's vertex positions are recomputed per frame as the scroll offset advances, so the visible window of the noise field "scrolls past" the car.

**TR-1.4** No code path in Act 1 reads `mouseX/mouseY` for terrain, contour, warp, or physics updates.

## TR-2. Field Sampling — Single Source of Truth

**TR-2.1** `sampleTerrainHeight(nx, sY) -> number` is the canonical (5-octave) height function. `sampleTerrainHeightFast(nx, sY)` is the 3-octave performance variant used for the mesh and the minimap.

**TR-2.2** The terrain mesh writes its sampled values into a shared `hCache` Float32Array as it builds vertex positions. The contour line builder reads from `hCache` rather than re-sampling. This is the "no double-sampling" rule.

**TR-2.3** Both height samplers are deterministic in their arguments — no time, no car state, no scroll state read inside.

## TR-3. 3D Contour Lines

**TR-3.1** A single `THREE.LineSegments` mesh holds all 3D contour lines, with capped `MAX_CSEGS` and pre-allocated typed arrays. No allocations in the per-frame builder.

**TR-3.2** Marching squares uses the canonical `MS_TABLE` constant. The same `MS_TABLE` is used by all renderers (3D contours, minimap, Act 2 river).

**TR-3.3** Contour lines are confined to the **road corridor** by skipping cells whose center has `roadDist > ROAD_HALF_W * 3`. This is the "topology lines only on the gameplay surface" rule from project goals.

**TR-3.4** Contour levels exist as a module-level constant `CONTOUR_LEVELS`. They are currently hardcoded `.18 → .85 step .025` (27 levels). The hardcoded approach is acceptable for the 3D scene because:
- The terrain noise field has a known stable range (`fbm` returns ~0–1).
- Lines must be **temporally stable** as the car drives — they should not slide around as the visible window's `[hMin, hMax]` shifts.
- Adaptive levels are required only for the minimap and Act 2, where the visible window changes character dramatically.

(This means TR-3.4 is *not* a requirement to make levels adaptive. It is a requirement to keep them stable and document why.)

**TR-3.5** Contour line color is dark warm gray, with edge fade `(1 - roadDist / (ROAD_HALF_W * 3))` so lines don't end abruptly at the corridor edge.

## TR-4. Warp Halo (Off-Road Distortion)

**TR-4.1** Each car carries a scalar `c.warp ∈ [0, 1]`. It rises when:
- `c.isOffRoad` is true, scaled by `(dfr - ROAD_HALF_W) / 0.18`
- The car is in a river (without bridge)
And it decays toward `0` when the car is on the road or on a bridge.

**TR-4.2** Inside `updateContourLines()`, when `warpAmt > 0.02`, every contour vertex is jittered by `noise2d(nx*12 + t*0.3, sY*8 + t*0.2) * warpAmt * WARP_AMPLITUDE` where `WARP_AMPLITUDE = 0.04`.

**TR-4.3** Because TR-3.3 already constrains contour lines to the road corridor (the player's neighborhood), TR-4.2 does not need a separate screen-space halo radius — the corridor *is* the halo. This is a deliberate Act 1 design choice and must be preserved.

**TR-4.4** Recovery is asymmetric: warp drops faster than it rises (`dt*15` vs `dt*3.5`) so the player isn't punished for brief excursions.

## TR-5. Input + Steering

**TR-5.1** WASD via `keyDown(c.idx, 0..3)`. Steering is **stackable**: held A/D applies `±steer*dt*2.4` to `c.heading`, which decays by `*0.91` per frame. This produces precise small-increment steering rather than jerky toggles. (Per `feedback_steering` memory.)

**TR-5.2** `c.lateralSpeed = c.heading * 0.32` clamped to `[-0.35, 0.35]`. The car can be pushed deep off-road if the player holds A/D long enough — the road-pull `(rTarget - c.nx) * 0.25 * dt` is gentle enough to be overridden.

**TR-5.3** Speed is clamped to `[-0.012, 0.025]` and decays by `*0.95` friction.

## TR-6. Stability + Hazards

**TR-6.1** `c.stability ∈ [10, 100]` is the canonical recovery scalar. It degrades on:
- Rough terrain (slope above threshold)
- Off-road (`c.isOffRoad`)
- Tree collisions (heavy)
- Hazards: pothole (-4), bump (-6), puddle (-3), mud (continuous)
- Rivers without bridge

**TR-6.2** Stability recovers at `+1.2/s` when on smooth road with `warp < 0.05`.

**TR-6.3** Tree collisions trigger `playImpactSound(1.0)`, particle burst, and a clamped `_treeHitFlash` (the clamping is critical — unbounded values flood the level indicator red).

## TR-7. Time-of-Day Theme System

**TR-7.1** Five named themes in `THEMES`: `dawn`, `day`, `golden`, `sunset`. Each contains a 17-color palette `pal`, fog/light parameters, road colors, and HUD tints.

**TR-7.2** `getTimeOfDayTheme(progress)` returns an interpolated theme based on drive progress. Transitions: dawn→day at 0–15%, day at 15–40%, day→golden at 40–65%, golden→sunset at 65–85%, sunset at 85%+.

**TR-7.3** Per frame, the active theme is applied to: clear color, fog color/distances, ambient + directional light intensity/color, document body background, terrain palette (via `T.pal`), and minimap palette.

**TR-7.4** When the end scene (lake / mountain group) is visible, fog is pushed back and lights are brightened so the vista reads correctly.

## TR-8. Minimap

**TR-8.1** `MM_COLS = MM_ROWS = 90`, throttled to every third frame (`mmFrame % 3 !== 0`).

**TR-8.2** `hMin`/`hMax` tracked per frame; `MM_NUM_BANDS = 14` adaptive levels computed as `hMin + i/(MM_NUM_BANDS+1) * hRange`.

**TR-8.3** Bilinear upscale: low-res `OffscreenCanvas (mmOff)` filled with `ImageData`, then `drawImage` scaled up with `imageSmoothingEnabled = true`.

**TR-8.4** Marching squares pass over the same `field` grid; lines drawn on top of upscaled bands. Every third level is darker for readability.

**TR-8.5** Road edges drawn as thin polylines from `getRoadX(sy) ± ROAD_HALF_W`.

**TR-8.6** Car marker drawn as a triangle rotated by `-car.heading`.

**TR-8.7** The minimap uses the **active theme palette** (via `getTimeOfDayTheme(progress)`) so it transitions in lockstep with the 3D scene.

### TR-8.NoDistortion — Anti-distortion contract for the minimap

These requirements collectively define what it means for the Act 1 minimap to be **not distorted**. Each must hold per frame. Failing any one is a regression.

**TR-8.8 (Square aspect)** All three of `MM_W === MM_H`, `MM_COLS === MM_ROWS`, and `viewW === viewH` must hold. The pixel aspect of one cell, the sample aspect of one cell, and the noise-space aspect of one cell are therefore all 1:1. No directional stretch on any axis.

**TR-8.9 (DPR-correct backing store)** The canvas backing store must be `MM_W * dpr × MM_H * dpr`, the CSS box must be `MM_W × MM_H`, and the context must be scaled by `dpr` exactly once at init via `mmCtx.scale(dpr, dpr)`. This guarantees the arrow path and contour strokes render at full physical resolution on retina screens. No re-scaling per frame, no transform reset.

**TR-8.10 (Arrow ↔ terrain alignment)** The car arrow's drawn screen position must coincide with the field cell that represents its actual `carSY`. Concretely:

```js
const carPy = carFrac * MM_H;
// At canvas y = carPy, the row is r = carFrac * MM_ROWS, and that row's
// sample sy must equal carSY:
//   syTop + carFrac * viewH === carSY
// ⇒ syTop === carSY - carFrac * viewH
```

If `syTop` is computed any other way (e.g. `carSY - (1 - carFrac) * viewH`, the legacy formula), the arrow visibly sits on terrain that is offset from the car's actual position — a subtle but real distortion that becomes more obvious as the canvas becomes square.

**TR-8.11 (Stable adaptive bands)** `_mmHMinS` / `_mmHMaxS` must be EMA-smoothed at `α ≈ 0.06`. Without this, the adaptive band thresholds twitch frame-to-frame as the visible window slides past local features, and the contour bands appear to crawl even when no time-based animation exists. Already in the implementation as of this document.

**TR-8.12 (No time-based animation by default)** `sampleTerrainHeightFast` is deterministic in `(nx, sy)`. The minimap's contour positions must change ONLY as a function of:
- viewport scroll (the car driving),
- the tree-hit warp halo within `WARP_R` of the arrow when `_treeHitFlash > 0`,
- river-adjacent flow rows (rows whose sy is within ±8 of any `RIVERS[i].dist`).

Any other animation source (per-frame `t * noise2d` modulation outside these zones, etc.) is a violation.

**TR-8.13 (No ghost trails)** Each frame must fully overwrite the canvas content. The bilinear band `drawImage(mmOff, 0, 0, MM_W, MM_H)` must cover the full `[0, MM_W] × [0, MM_H]` rectangle. No `clearRect` is required because the bands are opaque; if the bands ever become semi-transparent, an explicit `clearRect` must be added.

**TR-8.14 (Single source of truth grid)** Both the band-color pass and the marching-squares contour pass must read from the same `field` array sampled in the current frame. If the contour pass were to re-sample `sampleTerrainHeightFast`, the contour lines and the bands would visibly disagree — the same kind of distortion as TR-2.2 forbids in the 3D pipeline.

## TR-9. Performance

**TR-9.1** Terrain mesh updates are throttled: `if (Math.abs(scrollOff - _lastTerrainScroll) < 0.008) return;` — skip if scroll hasn't moved enough.

**TR-9.2** Contour line builder uses a pre-allocated `cPos` / `cCol` Float32Array of capacity `MAX_CSEGS = 24000` segments. No per-frame allocations.

**TR-9.3** Tree, lake, and end-scene meshes are added to the scene once and toggled via `.visible` rather than added/removed.

**TR-9.4** Minimap is throttled to every third frame.

**TR-9.5** The 3D contour pass uses `sampleTerrainHeightFast` (3-octave) by reading from `hCache`. The 5-octave `sampleTerrainHeight` is reserved for the car's own physics queries where accuracy matters.

## TR-10. Visual Tokens

**TR-10.1** The following constants must exist as named module-level constants, in one place each, so Act 2 can reuse them by name:

```js
const WARP_AMPLITUDE = 0.04;     // currently inlined as `*.04` in updateContourLines
const MM_NUM_BANDS = 14;         // ✅ exists
const MM_COLS = 80;              // ✅ exists
const MM_ROWS = 100;             // ✅ exists
const ROAD_HALF_W = 0.10;        // ✅ exists
const CATAPULT_TH = 0.95;        // ✅ exists
const FINISH_DIST = 3;           // ✅ exists
const HEIGHT_SCALE = 4;          // ✅ exists
const WORLD_SCALE = 80;          // ✅ exists
```

**TR-10.2** Theme palettes (`THEMES`) are the single source of truth for color. Both the 3D terrain mesh and the minimap read from `T.pal` — never from inline literal colors. (Currently true, must remain true.)

## TR-11. Verification Hooks

**TR-11.1** A `?act1debug=1` query param overlays:
- `c.warp`, `c.stability`, `c.speed`, `c.heading`
- `sharedScrollOffset` and current biome
- Number of contour segments drawn this frame
- `[hMin, hMax]` of the minimap window

**TR-11.2** Setting `WARP_AMPLITUDE = 0` (after TR-10.1 makes it a constant) must still render coherent contour lines — proves the warp halo is decoupled from line generation.

**TR-11.3** Holding `S` for 2s must visibly reverse the world scroll while the car stays at the same screen position.
