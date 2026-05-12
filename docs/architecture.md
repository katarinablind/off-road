# Off the Road — Architecture Reference

A technical audit of the Act-1 foundation (3D world + topology + streaming world + minimap) and a map of what's reusable for Act 2 (kayak). Written so Act 2 can be built on top of the existing primitives instead of parallel-implementing them.

Everything lives in `/index.html` (single file, currently ~3200 lines). Line ranges below are anchors, not contracts.

---

## 1. Coordinate systems

This is the keystone abstraction — understand it first.

### Three spaces

| Space | Range | What it is |
|---|---|---|
| **Normalized** `(nx, sY)` | `nx ∈ [0,1]`, `sY` unbounded | Seed-space. All noise/height sampling happens here. Deterministic across clients. |
| **World** `(wx, wy, wz)` | in Three.js units | What the camera sees. Lights, meshes, fog live here. |
| **Scroll** `sharedScrollOffset` | 0 → `FINISH_DIST` (3) | Scalar route progress. Increases as the player advances. |

### The key identity

```
carSY = 0.7 - sharedScrollOffset * 5
wz    = (anythingSY - carSY) * WORLD_SCALE   // ahead = negative Z
wx    = (nx - 0.5) * WORLD_SCALE
wy    = sampleTerrainHeight(nx, sY) * HEIGHT_SCALE
```

The **local car is pinned at Z=0** in world space. The world streams past it. Every world-space object (trees, lake, mountain group, end scene) re-anchors per frame via the same `(sY - carSY) * WORLD_SCALE` formula.

This is what makes the terrain feel infinite despite being a 90×90 plane mesh with only ~90×90 vertices: the mesh's vertex positions are rewritten every frame from the sliding sample window `[carSY - SY_VISIBLE*TERRAIN_AHEAD, carSY + SY_VISIBLE*(1-TERRAIN_AHEAD)]`.

**Act-2 implication**: the same "pin the agent, stream the world" trick works for a 2D top-down view. Or you can choose a fixed-world / moving-camera model instead. Both are feasible. The decision drives almost everything else.

---

## 2. Height field / scalar field

### Functions (lines 412, 479)

```
sampleTerrainHeight(nx, sY)      // full-fidelity: 5-octave FBM, all features
sampleTerrainHeightFast(nx, sY)  // 3-octave FBM, same features — used in tight loops
```

Both return a positive scalar (elevation). Structure:

```
base        = fbm(nx*4+10, sY*3+10) * biome_weight
+ road      = smoothed road centerline with undulations, inclines, steep sections
+ hazards   = deterministic potholes/bumps/rocks (seeded via ROAD_HAZARDS)
+ rivers    = gentle dip near river lines
```

**The pattern is reusable**: for kayaking the scalar field encodes *depth* (negative elevation) and the same layered approach (base noise + channel shaping + deterministic rocks/shoals) transfers directly.

### `getBiome(sY)` (line 401)

Returns one of `meadow | cliff | forest | alpine` based on route progress, tints terrain colors.

### `getRoadX(sY)` (line 243)

```js
ROAD_BASE + sin(s*.6)*.1 + sin(s*1.4+1.5)*.06 + sin(s*2.8+3)*.03
```

A sum-of-sines centerline. This is the template for any smooth 1D path — including a river centerline if you stay with the 1D-scroll model for Act 2.

---

## 3. Marching squares — the contour mechanic

### The table (line 836)

```js
MS_TABLE = [..., ...]   // 16 entries, each a list of [edge,edge] segment pairs
// Edge indices: 0=top, 1=right, 2=bottom, 3=left
// Bits packed: TL<<3 | TR<<2 | BR<<1 | BL<<0
```

**This table is used in three places — keep it as a single source of truth.**

### The three consumers

| Consumer | Location | Output | Warp source |
|---|---|---|---|
| `updateContourLines` | line 745 | 3D `THREE.LineSegments` | `warpAmt` (off-road factor) + river ripple |
| `renderAct2River` | line 2142 | 2D canvas strokes | Static (flow animated via noise coordinate shift) |
| `updateMinimap` | line 2571 | 2D canvas strokes | Radial bubble around car position |

All three share:
1. Sample the scalar field on a grid → `field[(cols+1)*(rows+1)]`
2. For each cell, compute 4-bit corner classification against iso-level
3. Look up `MS_TABLE[bits]` → segment list
4. Linearly interpolate edge crossing point
5. Optionally warp the resulting (x,y) before drawing

**Reusable extraction target**: a single `marchingSquares(field, cols, rows, iso, onSegment)` helper where `onSegment(x0, y0, x1, y1)` is a callback. The Act-1 3D path, the Act-2 water path, and the minimap would all call this. Currently each reimplements the same loop.

### The warp idiom (Puget-Sound radial bubble)

Only the minimap currently implements the full bubble:

```js
const dx = pt.x - carPx, dy = pt.y - carPy;
const d2 = dx*dx + dy*dy;
if (d2 < WARP_R2) {
  const d    = Math.sqrt(d2);
  const fall = 1 - d/WARP_R;
  const ease = fall*fall*(3-2*fall);     // smoothstep
  pt.x += (dx/d) * ease * deflectAmp;    // push outward
  pt.y += (dy/d) * ease * deflectAmp;
}
```

This is applied **both to contour vertices and to the band-color sample positions**, so the visual is consistent. Constants: `WARP_R=22` pixels, `deflectAmp = WARP_R` (push inner vertices out to the edge, making the disc read as an island).

**This is the hero visual of the project.** Any kayak Act 2 should carry it. The existing Act-2 code (`renderAct2River`) does *not* implement a bubble — only `waterGradient`-based drift. That's a reuse gap to close.

### Iso-levels (the breathing bug)

Minimap uses fixed range `hMinS=0.0, hMaxS=0.9` (line 2614). A previous version EMA-smoothed the observed `hMin/hMax`, which caused the whole topology to inhale/exhale as the window shifted. **Keep iso-levels absolute.** The Act-1 3D contour system uses a fixed `CONTOUR_LEVELS` array — same principle.

---

## 4. Rendering pipeline (Act 1)

### Terrain mesh (line 623)

- `PlaneGeometry(WORLD_SCALE, WORLD_SCALE*SY_VISIBLE, 90, 90)` — 91×91 vertices.
- `updateTerrainMesh(scrollOff)` rewrites every vertex's Y (and RGB color attribute) each frame.
- Vertex colors blend theme palette by elevation, biome tint, and road-dist blend.
- Hazards (potholes/bumps) are carved directly into the mesh via the same `sampleTerrainHeight` call.

### Trees — instanced, streaming

- `TREE_COUNT = 2000`, split across 3 `InstancedMesh` types.
- `TREE_SLOTS` (line 1088): a deterministic array of `{routePos, side, latW, scale, rotY, colors, tType, i}` seeded at load.
- `updateTrees(scrollOff)` (line 1109): per frame, for each slot, converts `treeSY = 0.7 - routePos*5` → `wz = (treeSY - carSY) * WORLD_SCALE`, samples terrain height, writes the instance matrix.
- Same streaming pattern generalizes to anything scattered in the world.

### Contour lines in 3D (line 745)

- Pre-fills `hCache` (91×91 Float32Array) from `sampleTerrainHeightFast` for the current window.
- Walks all cells at each `CONTOUR_LEVELS` iso, emits segments into a preallocated `cPos`/`cCol` Float32Array (max 24000 segments).
- `THREE.LineSegments` with `vertexColors:true, linewidth:2` — color fades with distance from road.
- Throttled: rebuilt every 100ms (line 2860). Terrain mesh itself rebuilds every frame.

### End scene (line 885)

The trailhead/parking/signs/barrier. All added to `mountainGroup`, positioned per-frame via the same scroll→Z conversion. Just fixed (see prior session — `endSceneSY = LSY + 0.02` anchor).

---

## 5. Minimap (line 2551)

The most complete 2D topology renderer in the project — **Act-2's starting template**, not Act-1's leftover.

### Pipeline

```
1. Sample height field on 90×90 grid (MM_COLS × MM_ROWS)
2. Compute MM_NUM_BANDS=14 iso-levels from fixed range [0, 0.9]
3. Band fill:
   a. For each pixel, compute canvas-space position
   b. If inside WARP_R bubble around car: deflect sample position radially
   c. Sample height at deflected position → find band index → color from theme palette
   d. Blend toward road color if near road centerline
   e. Write into OffscreenCanvas(90, 90) ImageData
   f. Bilinear upscale (imageSmoothing) to MM_W=240 × MM_H=240
4. Contour lines via marching squares:
   a. For each iso-level, walk cells, emit deflected edge segments
   b. Every 3rd level at higher opacity (major contour style)
5. Road edges (thin translucent polylines)
6. Red halo overlay: contour segments inside bubble, while tree-hit flash >0
7. Car marker: rotated chevron with drop shadow + outline
```

### DPR + square aspect

```js
const _mmDpr = clamp(devicePixelRatio, 1, 3);
mmCanvas.width  = MM_W * _mmDpr;   // retina backing store
mmCanvas.height = MM_H * _mmDpr;
mmCanvas.style.width  = MM_W + 'px';
mmCanvas.style.height = MM_H + 'px';
mmCtx.scale(_mmDpr, _mmDpr);       // logical coords stay MM_W × MM_H
```

### Throttle

`if (mmFrame++ % 3 !== 0) return;` — 20Hz. Cheap at current resolution.

### What's reusable verbatim for Act 2

Almost the entire pipeline. Rename `sampleTerrainHeightFast` → `sampleWaterField`, swap the road overlay for a channel outline, flip the band palette to water colors, and you're 80% of the way to the Sound map.

---

## 6. Input and agent state

### `keys` dictionary + `keyDown(pIdx, action)` (line 1554)

```js
const acts = [['w','arrowup'], ['s','arrowdown'], ['a','arrowleft'], ['d','arrowright']];
return keys[acts[action][0]] || keys[acts[action][1]];
```

Action indices: `0=forward, 1=back, 2=left, 3=right`. Reused identically by `updateCar` and `updateKayak`. Do not change this interface.

### Car state (`makeCar`, line 1547)

```
{ idx, nx, ny, speed, lateralSpeed, heading, warp, isOffRoad,
  catVx, catVy, catting, catTimer, catTargetX,     // catapult (hazard launch)
  scrollOffset, screenNX, screenNY,
  stability, damage, finished, parkPhase, parkTimer, parkTargetX }
```

### Kayak state (existing, `makeKayak`, line 1969)

```
{ idx, px, scrollY, vx, vs, heading,
  paddleAngle, paddleSide,
  stability, warp,
  isOnLand, finished, wake: [] }
```

**Reuse assessment**: the shape is close enough. The kayak drops `ny`, `speed`, the catapult subsystem, and `parkPhase`/`parkTimer`. It adds `wake` (a trail array) and `paddleAngle`. If you want multiplayer parity with Act 1, keep `idx` and mirror the sync pattern.

---

## 7. Multiplayer (PeerJS)

### Wire format

- `setupHost` creates a Peer, URL `?join=<id>&seed=<_seed.toString(36)>`.
- Data messages: `{t:'car', c}`, `{t:'scroll', v}`, `{t:'start'}`, `{t:'time', v}`.
- Host is authoritative on `sharedScrollOffset`; guest applies remote scroll if received.
- Seed is sent in the URL — the entire world is deterministic from seed.

### For Act 2

- Kayak state has the same `{idx, px, scrollY, vx, heading, stability}` shape to sync.
- If Puget Sound geography is hard-coded polygons (not seed-generated), multiplayer still works — the polygons are static constants, not RNG output.
- If any Act-2 content (rocks, current noise) uses `rng`, route it through the same seed so host and guest see the same world.

---

## 8. Audio (line 1394)

- `audioCtx`, gain nodes per source, filters tuned for engine/gravel/foliage.
- `mkDist(a)` generates a waveshaper curve for distortion.
- `makeLoopSource(buffer, gainNode, startGain)` abstraction — reusable for paddle loop / water ambient.
- `updateAudio(spd, warp, damage)` (line 1495) is the per-frame mixer. Act 2 will need its own equivalent `updateKayakAudio(speed, turbulence, impact)`.
- Currently `audioMuted = true` by default — user opt-in.

CC0 paddle splash source: still TBD.

---

## 9. Theme / time-of-day

`THEMES` (line 166): day / golden / dusk / night / dawn.
`getTimeOfDayTheme(progress)` (line 218): interpolates between two THEMES based on `progress ∈ [0,1]`.
Each theme: `{ pal: [[r,g,b], ...], bg, aL, dL, dC, fogNear, fogFar, roadCol, ... }`.

Terrain vertex colors, minimap bands, and fog all read the same palette — so Act 2 gets time-of-day for free if it uses `theme.pal`.

---

## 10. Current Act-2 attempt — what's there (lines 1903-2468)

The existing Act-2 code is a working-but-unshipped prototype. What it does:

- **River**: synthetic sum-of-sines centerline (`getRiverX`) with phase-based section types (`calm | narrows | rapids | pool`).
- **Water field**: `sampleWaterField(nx, s, flowOffset)` — noise + parabolic depth + rock noise in rapids.
- **Physics**: gradient-based drift (`waterGradient`), turbulence at high gradient, rock bouncing, edge push-back.
- **Rendering**: 2D canvas (`#topo-canvas`) with water-color band fill, marching-squares contour lines, kayak sprite with wake.
- **HUD integration**: reuses `h-speed-0`, `h-stab-0`, `progress-fill`, `timer`.

### What to keep

- `makeKayak` state shape (minus `vs` if dropping the 1D-scroll model).
- Paddle animation logic.
- Wake trail array.
- The rock-deflection + stability-damage loop.
- The 2D-canvas rendering skeleton.

### What to replace if doing Puget Sound properly

- `getRiverX` / `getRiverHW` / `getRiverSection` (all synthetic) → polygon-based `isWater(px, py)` from encoded Sound shoreline.
- `sampleWaterField` → depth = f(distance-from-shore, channel noise). Different sign convention (positive=depth, not raw noise).
- The 1D-scroll `scrollY` axis → 2D free position `(px, py)`. This is the biggest structural change.
- Sine-wave river clip path → signed-distance or point-in-polygon shore rendering.
- Gradient drift → named channel flow vectors.

### What's missing from the current Act-2 code vs. the minimap

- **No radial warp bubble around the kayak** — the Puget Sound visual idiom doesn't carry over.
- **No DPR-aware backing store** — will look soft on retina.
- **No bilinear upscale** — contours are rendered at full canvas resolution, which is fine but more expensive than the minimap's 90×90 → 240×240 pattern.
- **No theme palette integration** — hardcoded `BAND_COLORS` instead of `theme.pal`.

---

## 11. Proposed reusable primitives (extraction targets)

Extract before Act-2 rewrite so both Acts use the same plumbing. These don't exist as named functions today; they're inlined across 3 call sites.

### `marchingSquares(field, cols, rows, iso, visit)`

```js
function marchingSquares(field, cols, rows, iso, visit) {
  const stride = cols + 1;
  for (let r=0; r<rows; r++) for (let c=0; c<cols; c++) {
    const i = r*stride + c;
    const v00 = field[i], v10 = field[i+1], v01 = field[i+stride], v11 = field[i+stride+1];
    const bits = (v00>=iso?8:0) | (v10>=iso?4:0) | (v11>=iso?2:0) | (v01>=iso?1:0);
    if (bits===0 || bits===15) continue;
    for (const [e0, e1] of MS_TABLE[bits]) {
      const p0 = edgeInterp(c, r, e0, iso, v00, v10, v01, v11);
      const p1 = edgeInterp(c, r, e1, iso, v00, v10, v01, v11);
      visit(p0[0], p0[1], p1[0], p1[1]);
    }
  }
}
```

### `applyRadialWarp(x, y, cx, cy, warpR, amp)`

```js
function applyRadialWarp(x, y, cx, cy, warpR, amp) {
  const dx = x - cx, dy = y - cy, d2 = dx*dx + dy*dy;
  if (d2 >= warpR*warpR || d2 < 1e-6) return [x, y];
  const d = Math.sqrt(d2);
  const fall = 1 - d/warpR;
  const ease = fall*fall*(3-2*fall);
  const push = ease * amp;
  return [x + (dx/d)*push, y + (dy/d)*push];
}
```

Used everywhere: minimap contour vertices, minimap band samples, kayak's future topo canvas.

### `renderTopoCanvas({ctx, field, cols, rows, bounds, isoLevels, palette, warp, overlays})`

The single canvas-render function that both the minimap and Act 2 should call. `bounds` = canvas rect in pixels, `warp` = `{cx, cy, r, amp} | null`, `overlays` = array of fn(ctx) to draw road edges / shorelines / river centerlines / etc. on top.

### `sampleScalarField(nx, sY, layerFn)`

A general driver that composes FBM base + feature layers. Act-1's terrain and Act-2's water both become specific `layerFn` implementations.

---

## 12. Frame loop shape (line 2851)

```
frame():
  if !started:
    preview path — slow scroll, camera orbit, render, return
  if currentAct === 2:
    updateKayak; resize topo canvas; renderAct2River; update HUD; return
  // Act 1:
  updateCar(s) → maxDelta → sharedScrollOffset advance
  updateTerrainMesh; updateTrees; updateContourLines (throttled)
  updateCamera; updateParticles
  render end scene (mountainGroup + lakeMesh) per-frame anchored
  render scene3d
  update HUD, level indicator, minimap
  updateKayakHighlight (post-finish glow)
```

The top-level branch on `currentAct` is clean. Act 2's entire rendering path bypasses Three.js — if you keep 2D, the `renderer.render` call is simply skipped. If you go 3D-top-down, you'd reuse the same WebGL canvas with a different camera and a different scene graph.

---

## 13. File structure

Single file, 3207 lines currently. Rough breakdown:

| Lines | Section |
|---|---|
| 1–160 | CSS |
| 160–200 | THEMES data |
| 200–330 | Config, RNG, multiplayer plumbing |
| 330–400 | PeerJS |
| 400–540 | Noise, biome, height sampling |
| 540–610 | Features, lake, rivers, hazards |
| 610–740 | Three.js scene + terrain mesh |
| 740–860 | Contour lines (3D) + MS_TABLE |
| 860–1050 | Sun, lake, end-scene meshes |
| 1050–1165 | Trees |
| 1165–1400 | Car mesh builder |
| 1400–1550 | Audio |
| 1550–1820 | Car physics |
| 1820–1900 | Finish screens |
| 1900–2470 | Act-2 kayak (current attempt) |
| 2470–2550 | Level indicator, kayak highlight |
| 2550–2830 | Minimap |
| 2830–3207 | Main frame loop, HUD, etc. |

The MEMORY.md constraint says "single HTML file" — so extractions stay as in-file helpers, not modules.
