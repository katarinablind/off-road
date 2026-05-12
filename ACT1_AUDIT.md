# Act 1 — Implementation Audit

Audit of `index.html` Act 1 code against `ACT1_TECH_REQUIREMENTS.md`.

Status legend: ❌ Missing · ⚠️ Wrong · 🟡 Partial · ✅ Met

**Headline:** Act 1 substantially meets its own spec. The structural and visual systems are all in place — single-source-of-truth height field, warp halo, theme transitions, adaptive minimap, throttled performance. The remaining gaps are small: a few inlined magic numbers should become named constants, and there's no debug overlay. This audit exists primarily to **freeze** the working implementation as a contract that Act 2 will mirror.

---

## TR-1. Avatar-Static / World-Active

| ID | Status | Evidence |
|---|---|---|
| TR-1.1 | ✅ | Camera tracks car each frame: `camPos.x = lerp(camPos.x, camTargetX, .06)` etc., where `camTargetX = carWX + shake`. (Lines 2913–2928) The car always renders at the camera's tracked offset. |
| TR-1.2 | ✅ | `updateCar()` mutates `c.speed`, `c.heading`, `c.lateralSpeed`, and returns `sDelta` which is added to `sharedScrollOffset`. (Lines 1667–1803) |
| TR-1.3 | ✅ | `updateTerrainMesh(scrollOff)` recomputes vertex positions per frame, walking `sYMin..sYMax` derived from `carSY = 0.7 - scrollOff*5`. (Lines 627–728) The mesh is repositioned at `(0,0,0)` so the car at world Z=0 is always centered. |
| TR-1.4 | ✅ | No `mouseX/mouseY` reads in Act 1 physics, terrain, or contour code. |

## TR-2. Field Sampling — Single Source of Truth

| ID | Status | Evidence |
|---|---|---|
| TR-2.1 | ✅ | `sampleTerrainHeight` (line 400, 5-octave `fbm`) and `sampleTerrainHeightFast` (line 467, 3-octave `fbm3`) both defined. |
| TR-2.2 | ✅ | `updateTerrainMesh` writes to `hCache[vIdx] = v` (line 650) while building vertex positions. `updateContourLines` reads from `hCache[j*(S+1)+i]` (lines 788–791) — no resampling. |
| TR-2.3 | ✅ | Both samplers take only `(nx, sY)` arguments. They read `RIVERS`, `PASS_WY`, `ROAD_HAZARDS` (deterministic constants), but no time/car state. |

## TR-3. 3D Contour Lines

| ID | Status | Evidence |
|---|---|---|
| TR-3.1 | ✅ | `contourLines3d = new THREE.LineSegments(contourGeo, contourLineMat)` with `MAX_CSEGS = 24000` and pre-allocated `cPos = new Float32Array(MAX_CSEGS*6)`. (Lines 832–840) |
| TR-3.2 | ✅ | `MS_TABLE` defined at line 823, used by `updateContourLines` (line 794), `updateMinimap` (line 2639), and Act 2 `renderAct2River` (line 2260). Single canonical table. |
| TR-3.3 | ✅ | `updateContourLines` skips cells with `roadDist > ROAD_HALF_W * 3` (line 786). Rule held. |
| TR-3.4 | 🟡 | `CONTOUR_LEVELS` exists as a module-level constant (lines 830–831), populated as `.18 → .85 step .025`. Hardcoded by design (per spec). The spec is met; flagging as 🟡 only because the *rationale* for hardcoding isn't documented in the source — a one-line comment would lock this in. |
| TR-3.5 | ✅ | Edge fade `(1 - roadDist / (ROAD_HALF_W * 3))` applied at line 800; color is `(.28*ef, .16*ef, .10*ef)` — dark warm gray. |

## TR-4. Warp Halo (Off-Road Distortion)

| ID | Status | Evidence |
|---|---|---|
| TR-4.1 | ✅ | `c.warp` updated in `updateCar`: targetWarp from off-road distance, river boost, bridge clear. (Lines 1693–1696, 1729) |
| TR-4.2 | 🟡 | Distortion applied at line 764: `noise2d(nx*12+t*.3, sY*8+t*.2)*.5-.25`, then `nx + distort*warpAmt*.04`. The `.04` is **inlined**, not a named `WARP_AMPLITUDE` constant. Functionally correct, structurally not extractable for Act 2 to import. |
| TR-4.3 | ✅ | Corridor restriction (TR-3.3) is the halo. Confirmed working. |
| TR-4.4 | ✅ | Asymmetric recovery: `c.warp += (target - c.warp) * dt * 3.5` rising vs `* dt * 15` falling (lines 1694–1695). |

## TR-5. Input + Steering

| ID | Status | Evidence |
|---|---|---|
| TR-5.1 | ✅ | Lines 1678–1680: `c.heading -= steer*dt*2.4` etc., decay `*.91`. Stackable held-key behavior matches the steering memory. |
| TR-5.2 | ✅ | `c.lateralSpeed = c.heading * 0.32` clamped to `[-0.35, 0.35]` (lines 1681–1682). Road-pull `*0.25*dt` (line 1687) is gentle enough to override. |
| TR-5.3 | ✅ | Speed clamp `[-0.012, 0.025]`, friction `*0.95` (line 1669). |

## TR-6. Stability + Hazards

| ID | Status | Evidence |
|---|---|---|
| TR-6.1 | ✅ | Stability degradation rules at lines 1697–1771 (rough terrain, off-road, tree collision, hazards, river). All clamped to floor of 10. |
| TR-6.2 | ✅ | Recovery at line 1774: `if (!c.isOffRoad && !inRiver && c.warp < .05) c.stability = Math.min(100, c.stability + 1.2*dt)`. |
| TR-6.3 | ✅ | Tree hit clamps `_treeHitFlash = Math.min(1, ...+0.3)` (line 1716) — clamping fix from earlier session is in place. |

## TR-7. Time-of-Day Theme System

| ID | Status | Evidence |
|---|---|---|
| TR-7.1 | ✅ | `THEMES = { dawn, day, golden, sunset }` at lines 160–193, each with full palette + lighting + HUD tints. |
| TR-7.2 | ✅ | `getTimeOfDayTheme(progress)` at lines 212–218 with the documented transition windows. |
| TR-7.3 | ✅ | Per-frame application at lines 2962–2969: clear color, fog, lights, body background. Terrain palette read from `T.pal` inside `updateTerrainMesh` (line 636). Minimap reads `theme.pal` (line 2562). |
| TR-7.4 | ✅ | End-scene fog/light boost at lines 2971–2975. |

## TR-8. Minimap

| ID | Status | Evidence |
|---|---|---|
| TR-8.1 | ✅ | `MM_COLS = MM_ROWS = 90`; throttle `if (mmFrame % 3 !== 0) return`. |
| TR-8.2 | ✅ | `hMin/hMax` tracked; `mmLevels` adaptive. `MM_NUM_BANDS = 14`. |
| TR-8.3 | ✅ | `OffscreenCanvas(MM_COLS, MM_ROWS)`; `imageSmoothingEnabled = true` + `drawImage(mmOff, 0, 0, MM_W, MM_H)`. |
| TR-8.4 | ✅ | Marching squares pass over the same `field` grid. Every third level darker: `li%3===0 ? 'rgba(40,30,20,.3)' : 'rgba(40,30,20,.15)'`. |
| TR-8.5 | ✅ | Road edges from `getRoadX(sy) ± ROAD_HALF_W`. |
| TR-8.6 | ✅ | Car marker with `rotate(-car.heading)`, no center dot, refined arrow geometry. |
| TR-8.7 | ✅ | `getTimeOfDayTheme(progress)`, palette read from `theme.pal`. |
| TR-8.8 | ✅ | `MM_W === MM_H === 180`, `MM_COLS === MM_ROWS === 90`, `viewW === viewH === 0.55`. Square at every level. |
| TR-8.9 | ✅ | DPR-aware init: backing store sized `MM_W*dpr × MM_H*dpr`, CSS box `MM_W × MM_H`, single `mmCtx.scale(dpr, dpr)` at init. |
| TR-8.10 | ✅ | `syTop = carSY - carFrac * viewH` — fixed from the legacy `carSY - (1-carFrac)*viewH`, which placed the arrow ~0.24 sy-units behind its actual terrain. This was the source of the user-reported "minimap is distorted." |
| TR-8.11 | ✅ | `_mmHMinS` / `_mmHMaxS` EMA-smoothed at α = 0.06. |
| TR-8.12 | ✅ | Field is sampled deterministically per frame; the only motion sources are viewport scroll, the tree-hit warp halo within `WARP_R` of the arrow when `_treeHitFlash > 0`, and river-adjacent rows from `RIVERS[i].dist`. |
| TR-8.13 | ✅ | Bands are opaque and `drawImage` covers `[0,MM_W] × [0,MM_H]` each frame. |
| TR-8.14 | ✅ | One `field` array, written by the band-color pass, read by the marching-squares contour pass — no resampling. |

## TR-9. Performance

| ID | Status | Evidence |
|---|---|---|
| TR-9.1 | ✅ | Terrain throttle at line 628: `if (Math.abs(scrollOff - _lastTerrainScroll) < 0.008) return`. |
| TR-9.2 | ✅ | `MAX_CSEGS = 24000`, pre-allocated `cPos`, `cCol` (lines 832–834). |
| TR-9.3 | ✅ | Trees / lake / mountain group toggled via `.visible`, never added/removed mid-game. |
| TR-9.4 | ✅ | Minimap third-frame throttle. |
| TR-9.5 | ✅ | `updateTerrainMesh` calls `sampleTerrainHeightFast` (line 649); `updateContourLines` reads from `hCache` (no resampling); car physics calls `sampleTerrainHeight` (5-octave) for accuracy (lines 1698–1700). |

## TR-10. Visual Tokens

| ID | Status | Evidence |
|---|---|---|
| TR-10.1 | 🟡 | `MM_NUM_BANDS`, `MM_COLS`, `MM_ROWS`, `ROAD_HALF_W`, `CATAPULT_TH`, `FINISH_DIST`, `HEIGHT_SCALE`, `WORLD_SCALE` all exist as named constants. **`WARP_AMPLITUDE` is the only missing one** — currently inlined as `*.04` at line 765. |
| TR-10.2 | ✅ | Themes are the single source of truth. Terrain mesh reads `T.pal`, minimap reads `theme.pal`. No inline color literals in the rendering paths (HUD/UI colors are separate). |

## TR-11. Verification Hooks

| ID | Status | Evidence |
|---|---|---|
| TR-11.1 | ❌ | No debug overlay. |
| TR-11.2 | ❌ | n/a — `WARP_AMPLITUDE` is not a separable knob (TR-10.1). |
| TR-11.3 | ✅ | Implicit: `S` calls `c.speed -= .8*dt` (line 1668), which decreases the scroll delta. The car stays at world Z=0 in screen-camera space throughout. |

---

## Summary scoreboard

| Category | Met | Partial | Wrong | Missing |
|---|---|---|---|---|
| Avatar/world inversion (TR-1) | 4 | 0 | 0 | 0 |
| Field sampling (TR-2) | 3 | 0 | 0 | 0 |
| 3D contour lines (TR-3) | 4 | 1 | 0 | 0 |
| Warp halo (TR-4) | 3 | 1 | 0 | 0 |
| Input + steering (TR-5) | 3 | 0 | 0 | 0 |
| Stability + hazards (TR-6) | 3 | 0 | 0 | 0 |
| Theme system (TR-7) | 4 | 0 | 0 | 0 |
| Minimap (TR-8) | 14 | 0 | 0 | 0 |
| Performance (TR-9) | 5 | 0 | 0 | 0 |
| Tokens (TR-10) | 1 | 1 | 0 | 0 |
| Debug (TR-11) | 1 | 0 | 0 | 2 |
| **Total** | **45** | **3** | **0** | **2** |

Compare with Act 2's audit (9 met / 3 partial / 7 wrong / 11 missing): Act 1 is essentially complete. Three small partials, two missing debug hooks, zero "wrong."

---

## Real gaps (the only things to actually fix in Act 1)

1. **TR-10.1 / TR-4.2** — Extract the inlined `*.04` warp amplitude into a named constant `WARP_AMPLITUDE = 0.04` near the other Act 1 visual tokens. This is the single highest-leverage change because it lets `ACT2_*` import the exact same constant by name and guarantees the two acts distort at the same scale.
2. **TR-3.4** — Add a one-line comment above `CONTOUR_LEVELS` documenting *why* it's hardcoded (temporal stability across frames in the 3D scene), so future-you doesn't try to "fix" it to adaptive and break line stability.
3. **TR-11.1** — Add a `?act1debug=1` overlay surfacing `c.warp`, `c.stability`, `sharedScrollOffset`, segment count, and minimap `[hMin, hMax]`. Useful immediately for verifying changes.

That's it. Three small edits, all in the same vicinity of the file.

---

## Recommended order of work

1. **Extract `WARP_AMPLITUDE` constant + use it in `updateContourLines`** (TR-10.1, TR-4.2). ~2 lines.
2. **Comment `CONTOUR_LEVELS`** documenting the hardcoded-by-design rationale (TR-3.4). ~1 line.
3. **Add `?act1debug=1` overlay** (TR-11.1). ~30 lines, isolated, doesn't touch anything else.

After these, Act 1 is a clean, freezeable contract for Act 2 to mirror.
