# Act 2 (Kayak Rapids) — PRD

Target file: `index.html` (single-file project). All work happens here.

`/Users/katarina/experiments/puget-sound/` is a sibling project that was once supposed to share this visual/interaction language but currently doesn't. It is **reference only** and is **not** being modified by this work.

## 1. Vision

Act 2 is the bathymetric counterpart to Act 1. The car has reached the trailhead, the player launches the kayak, and the river takes over. The river is a topographic map you paddle through — the contour lines aren't decoration, they're the medium of interaction, the same way the mountain road's contour lines are in Act 1.

## 2. Core Principle: Avatar-Static / World-Active

This is the rule that already unifies Act 1 and Act 2 and must continue to hold:

- The kayak sits at a fixed point on screen (top of view, since the river flows top→bottom).
- WASD scrolls the **river past the kayak**, never the kayak across the river.
- The kayak's *presence and movement* distorts the contour lines in its immediate neighborhood — a warp halo around the static avatar.
- This mirrors Act 1: the car never moves on screen, the road scrolls under it, and topology lines warp in a halo around the car when off-road.

Act 2's `renderAct2River()` already uses `viewTop = k.scrollY - 0.15` so the kayak stays near the top while the river flows past. The structural inversion is therefore **already done**. What's missing is the *visual reaction* and the *visual language* that make the inversion legible.

## 3. Visual Language (must match Act 1)

Act 1 established a visual language that the off-road minimap (`#minimap-canvas`) renders successfully and that Act 2 must adopt:

### 3.1 Topology lines
- Marching squares at **adaptive iso levels** computed from the visible window's `[hMin, hMax]` each frame — not hardcoded thresholds.
- 14 contour bands as a baseline (matches Act 1 minimap's `MM_NUM_BANDS = 14`).
- Same stroke treatment: alpha and weight ramp slightly with band index.

### 3.2 Filled bands
- Bands between contour levels are filled with palette colors and **bilinearly upscaled** from a low-resolution sample grid (~80×100), exactly the same path the minimap uses with `mmOff` / `mmOffCtx`.
- Each band is one color step in a hand-picked palette. No airbrush gradients, no per-pixel HSL noise washes.
- The palette must read as continuous with Act 1's warm mountain bands. Cool teals are acceptable for deep channels, but the dominant treatment — dark base, layered mid-tones, light highlights — must feel like the same world seen through water.

### 3.3 Detail markers (shores)
- Land/shore cells receive **scattered detail dots** — small dark filled circles, density modulated by sampled value, deterministic from world coordinates so dots don't shimmer when scrolling. This is the bathymetric equivalent of the off-road forest tree dots seen from above.
- Currently shores are flat `#3a3228`. Replace with the dot pass.

### 3.4 Warp halo
- A warp scalar lives on the kayak (`k.warp` is already defined in `makeKayak`, currently unused for rendering).
- When drawing contour line segments inside a screen-space radius around the kayak, segment endpoints are jittered by `noise2d * warp * 0.04` of cell size — the same constant Act 1 uses in `updateContourLines()`.
- Warp increases when the kayak is on rocks, near the river edge, or in high-gradient water; decays in calm pools.

### 3.5 What is forbidden
- Smooth airbrush radial gradients
- Per-pixel HSL noise washes that erase topology
- Visual elements that don't read as a topographic / bathymetric map at a glance
- Any visual that breaks continuity with Act 1's mountain treatment

## 4. Interaction Model (already largely correct)

| Aspect | Status |
|---|---|
| Avatar pinned on screen | ✅ already done (`viewTop = k.scrollY - 0.15`) |
| WASD scrolls field past avatar | ✅ already done in `updateKayak()` |
| Mouse not consulted | ✅ already done — `index.html` Act 2 reads no mouse state |
| Rock/edge collisions push kayak | ✅ already done |
| Stability meter degrades on rocks/turbulence | ✅ already done |
| **Warp halo distorts contours around kayak** | ❌ `k.warp` exists but is never read by the renderer |

## 5. Cohesion Between Acts

| Dimension | Act 1 | Act 2 |
|---|---|---|
| Avatar | Car, screen-center | Kayak, top of view |
| Movement | Road scrolls under car | River scrolls under kayak |
| Topology lines | Marching squares, adaptive levels | Marching squares — currently fixed levels (must become adaptive) |
| Bands | Bilinear-upscaled palette fills | Bilinear-upscaled — already in place, palette needs alignment |
| Detail markers | Scattered tree dots | Scattered shore dots — must be added |
| Warp halo | Active around car when off-road | Defined on kayak, not yet rendered |
| Recovery | Stability meter | Same |

A player crossing from Act 1 to Act 2 should feel a continuous visual world.

## 6. Out of Scope

- The puget-sound sibling project. Reference only.
- Multiplayer kayaks (Act 2 already initializes `kayaks[1]` but the user-facing experience for this PRD is single player).
- Real Puget Sound geography. The current procedural river (`getRiverX`, `getRiverHW`, `getRiverSection`) is fine.
- Audio. Existing engine works.

## 7. Definition of Done

1. Act 2 contour bands use **adaptive levels** computed from the current visible window each frame.
2. Act 2 sample grid is **80×100**, not 240×180.
3. The kayak warp halo visibly distorts contour lines in a radius around the kayak when on rocks or in high-gradient water.
4. Shore areas show scattered detail dots, not flat fill.
5. Palette reads as continuous with Act 1 mountain — same family of warm/experimental tones, no candy blue.
6. Frame time stays under 16ms on the same hardware Act 1 runs at 60fps.
