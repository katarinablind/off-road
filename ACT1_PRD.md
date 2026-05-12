# Act 1 (Off-Road Driving) — PRD

Target file: `index.html` (single-file project). All work happens here.

This is the **canonical reference implementation** for the project's interactive-topology language. Act 2 (kayak) must mirror it. Anything labeled "the way Act 1 does it" in `ACT2_*` docs traces back to this document.

## 1. Vision

Act 1 is a 3D off-road driving experience where the topographic map is the world. The player drives a car through procedurally generated terrain with contour lines layered onto the road surface, time-of-day lighting that progresses with the route, and collisions/hazards that affect car stability and distort the topology. The car never moves on screen — the world scrolls past it.

## 2. Core Principle: Avatar-Static / World-Active

This is the rule that defines the project. Act 1 already implements it correctly:

- The car sits at a fixed point in world space (the camera tracks it). The car's **screen position is constant**.
- WASD changes `c.speed` and `c.heading`, which feed into a shared `scrollOffset` that scrolls the **world past the car**, not the car across the world.
- The terrain mesh is **resampled and repositioned per frame** so the car always sits at world Z=0; what changes is which slice of the noise field gets uploaded into the mesh's vertex positions.
- The car's *presence and movement* distorts the contour lines through the `c.warp` halo when off-road.

This is the model. Everything else in the project — the minimap, Act 2's kayak — is a translation of this same model into a different render target.

## 3. Visual Language

Act 1 establishes the visual language. These are the elements that must hold:

### 3.1 Topology lines
- 3D `LineSegments` mesh built per frame from **marching squares** on the same height cache (`hCache`) the terrain mesh writes during its update — single source of truth, no double-sampling.
- 27 contour levels from `.18` to `.85` step `.025` (currently hardcoded). Lines are confined to a road corridor (`roadDist < ROAD_HALF_W * 3`) — the gameplay surface — by design. Forest never carries topology lines.
- Color is dark warm gray, fading slightly at the corridor edges so lines don't end abruptly.

### 3.2 Terrain bands
- The 3D terrain mesh uses `MeshStandardMaterial` with `flatShading: true` and per-vertex colors driven by elevation through the active `THEMES.pal` palette. No smooth airbrush gradients — the faceted shading is intentional.
- Biome tinting (cliff, forest, alpine) modulates the palette per region. Road cells are sharply differentiated with a dirt color and tire ruts.

### 3.3 Forest detail markers
- Trees are 3D `InstancedMesh` with three canopy types and per-instance colors. Density and placement are deterministic (`TREE_SLOTS`).
- The forest is the bathymetric equivalent of "land" in Act 2 — it's the surface the player is *not* supposed to be on, and it's rendered as scattered 3D detail rather than a flat fill.

### 3.4 Time-of-day theme transitions
- Five named themes (`dawn`, `day`, `golden`, `sunset`) interpolate by drive progress in `getTimeOfDayTheme()`. Fog color, light intensity/color, sky color, and the entire palette transition in lockstep.
- This is what gives the drive its emotional arc. It must continue to apply.

### 3.5 Warp halo (off-road distortion)
- A scalar `c.warp ∈ [0, 1]` rises when the car is off-road, in a river, or otherwise unstable.
- Inside `updateContourLines()`, when `warp > 0.02`, every contour vertex in the visible road corridor is jittered by `noise2d(...) * warp * 0.04` — the project's canonical `WARP_AMPLITUDE`.
- Because contour lines are confined to the road corridor, the distortion is automatically localized to the player's neighborhood — there's no separate halo radius needed in 3D.

### 3.6 Minimap (2D top-down)
- A small 2D top-down topology view in the bottom-left corner sampled at `MM_COLS × MM_ROWS = 80 × 100`, with **adaptive elevation bands** computed from the visible window's `[hMin, hMax]`, bilinearly upscaled via `OffscreenCanvas`, marching squares contours over the same field, and a car marker.
- The minimap is the reference implementation of the bilinear-upscale + adaptive-bands pattern that Act 2 must adopt.

### 3.7 What is forbidden
- Smooth airbrush radial gradients
- Per-pixel HSL noise washes that erase topology
- Any visual that doesn't read clearly as a topographic map at a glance
- Removing topology lines from the road corridor for any reason

## 4. Interaction Model

### 4.1 Movement
- WASD or arrow keys, hands-off camera. Steering is **stackable** — small held-key increments accumulate into `c.heading`, decay slowly. Precise small inputs preferred over jerky responses (per `feedback_steering` memory).
- Speed is clamped (`-0.012, 0.025`) and decays by friction. The world moves; the car doesn't.
- Camera is third-person behind the car at a fixed offset, with subtle warp-driven shake when off-road.

### 4.2 Stability
- `c.stability ∈ [10, 100]` degrades on rough terrain, off-road, hazards (potholes, bumps, puddles, mud), tree collisions, and rivers. Recovers slowly on smooth road.
- Tree hits and pothole hits trigger particle bursts and impact sounds.
- The level/bubble indicator at the bottom of screen reflects combined roll/lateral/warp state.

### 4.3 Catapult
- When `c.warp > CATAPULT_TH` (0.95), the car is "catapulted" back toward the road centerline — a soft recovery to keep the player from getting permanently stuck deep in the woods.

## 5. Cohesion Targets

Act 1 is the **source** of cohesion, not the recipient. Its job is to be internally consistent so Act 2 has something concrete to mirror. The dimensions Act 2 must inherit:

| Dimension | Act 1 specifies it as |
|---|---|
| Avatar position | Static, screen-pinned |
| World motion | Scrolls past avatar via shared scroll offset |
| Topology source | Marching squares over a single height cache |
| Topology corridor | Constrained to the gameplay surface (road / channel) |
| Distortion | `noise2d * warp * 0.04` (`WARP_AMPLITUDE = 0.04`) |
| Palette | Theme-driven, warm-anchored, faceted (no gradients) |
| Detail markers | Off-surface areas filled with scattered detail (trees / shore dots) |
| Recovery | Stability scalar with soft catapult / push-back |

## 6. Out of Scope

- Act 2 (kayak). Covered by `ACT2_PRD.md`.
- Real geographic accuracy — the procedural mountain is fine.
- Any change to the 3D terrain pipeline that would replace `flatShading: true` with smooth shading.

## 7. Definition of Done

1. Topology lines render every frame on the road corridor with no flicker or popping as the car drives.
2. Contour line distortion is visible when the car goes off-road, and recedes when the car returns to the road.
3. Terrain bands track the active theme and transition smoothly with drive progress.
4. The minimap is legible at all times and uses adaptive bands.
5. Performance: 60 fps on the same hardware Act 2 targets.
6. The visual tokens (`WARP_AMPLITUDE`, `MM_NUM_BANDS`, etc.) live in one place each, named consistently, so Act 2 can import the same constants by name.
