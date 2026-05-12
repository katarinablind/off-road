# Act 2 — Implementation Audit

Audit of `index.html` Act 2 code (lines 1888–2453) against `ACT2_TECH_REQUIREMENTS.md`.

Status legend: ❌ Missing · ⚠️ Wrong · 🟡 Partial · ✅ Met

**Headline:** the *structural* inversion (avatar-static + world-scrolling + marching squares + bilinear upscale + WASD-only input) is **already done**. The gaps are visual: hardcoded contour levels, no warp halo distortion, candy-blue palette breaks continuity, no shore detail markers, sample grid is 5× over budget.

This is what the user meant by "a visual approximation without any of the actual function and refined visuals behind it" — the function shell is there, the refined visuals aren't.

---

## TR-1. Avatar-Static / World-Active

| ID | Status | Evidence |
|---|---|---|
| TR-1.1 | ✅ | `viewTop = k.scrollY - 0.15` (line 2136); `toScreen()` defined at 2142–2144. Kayak stays near top of view, river scrolls past. |
| TR-1.2 | ✅ | `updateKayak()` mutates `k.scrollY`, `k.vx`, `k.px` from WASD via `keyDown` (lines 2061–2066). |
| TR-1.3 | ✅ | No `mouseX/mouseY` reads anywhere in Act 2 (`launchAct2`, `updateKayak`, `renderAct2River`). |

## TR-2. Field Sampling

| ID | Status | Evidence |
|---|---|---|
| TR-2.1 | ✅ | `sampleWaterField(nx, s, flowOffset)` (lines 1920–1936) is deterministic in its arguments — does not read kayak. |
| TR-2.2 | ❌ | No `applyKayakWarp` exists. Kayak does not currently influence the field at all. (This is allowable — the warp halo can be a render-time line jitter, not a field modification — but TR-5 still requires the visual effect.) |

## TR-3. Sample Grid + Bilinear Upscale

| ID | Status | Evidence |
|---|---|---|
| TR-3.1 | ⚠️ | `ACT2_COLS = 240, ACT2_ROWS = 180` (line 2125) → 43,200 samples per frame. Spec requires 80×100 = 8,000 to match minimap. |
| TR-3.2 | ❌ | No `hMin`/`hMax` tracking. Field values are used directly against hardcoded thresholds. |
| TR-3.3 | 🟡 | An offscreen canvas is cached on `renderAct2River._offC` (lines 2192–2196) and drawn back with `imageSmoothingEnabled = true` (lines 2233–2235). Works, but uses a regular `<canvas>` rather than `OffscreenCanvas`. The *bilinear upscale path* is correct; only the wrapper differs from the minimap's pattern. |

## TR-4. Adaptive Contour Levels

| ID | Status | Evidence |
|---|---|---|
| TR-4.1 | ❌ | `WATER_LEVELS = [.05, .10, .16, .22, .28, .34, .42, .52]` (line 1948) — 8 hardcoded levels, no adaptation to actual field range. Spec requires 14 adaptive levels per frame from `[hMin, hMax]`. |
| TR-4.2 | ✅ | Marching squares (lines 2237–2298) runs against the same `field` grid that produced the bands. |
| TR-4.3 | ⚠️ | Stroke style is mostly constant: `LINE_COLOR` for everything except the rock-line which gets a special foam treatment (lines 2243–2250). No alpha/weight ramp by band index. |

## TR-5. Kayak Warp Halo

| ID | Status | Evidence |
|---|---|---|
| TR-5.1 | ⚠️ | `k.warp` is declared in `makeKayak` (line 1957) but **never assigned anywhere** in `updateKayak`. It is also never read by the renderer. |
| TR-5.2 | ❌ | No segment jitter in a halo around the kayak. The closest thing is the global `turb` offset inside the marching squares loop (lines 2270–2279) which uses field gradient — applied everywhere, not anchored to the kayak. |
| TR-5.3 | ❌ | n/a — no halo exists. |
| TR-5.4 | n/a | The existing `turb` offset is the global rapids effect that TR-5.4 explicitly allows to remain. |

## TR-6. Input

| ID | Status | Evidence |
|---|---|---|
| TR-6.1 | ✅ | `keyDown(k.idx, 0..3)` for W/S/A/D in `updateKayak` (lines 2061–2066). |
| TR-6.2 | ✅ | No mouse input in Act 2. |

## TR-7. Detail Markers (Shores)

| ID | Status | Evidence |
|---|---|---|
| TR-7.1 | ❌ | No scattered dot pass. Shore cells are filled with a single flat color. |
| TR-7.2 | ❌ | `topoCtx.fillStyle = '#3a3228'; topoCtx.fillRect(0,0,W2,H2)` (lines 2157–2158) draws a uniform background. Inside the band loop (lines 2208–2230), shore cells just inherit `shoreRGB = [0x3a, 0x32, 0x28]` — also flat. |
| TR-7.3 | ❌ | n/a — no dots to draw before contours. |

## TR-8. Visual Cohesion with Act 1

| ID | Status | Evidence |
|---|---|---|
| TR-8.1 | ⚠️ | `BAND_COLORS = ['#081428', ..., '#3a7488']` (line 1949) is candy blue. Breaks continuity with Act 1's warm mountain. |
| TR-8.2 | ⚠️ | `LINE_COLOR = 'rgba(100,210,255,0.6)'` (line 1951) is sky blue. Should be a warm luminous tone. |
| TR-8.3 | ⚠️ | `document.body.style.background = '#0a1628'` (line 2438) is cold midnight blue. Should be warm dark. |

## TR-9. Performance

| ID | Status | Evidence |
|---|---|---|
| TR-9.1 | ⚠️ | 240×180 = 43,200 samples per frame — about 5× the minimap budget. Reducing to 80×100 fixes this. |
| TR-9.2 | ✅ | Both passes share the `field` grid (already true). |
| TR-9.3 | 🟡 | Whitewater foam loop iterates the full grid (lines 2301–2321). At 43,200 cells this is heavy; after the TR-3.1 reduction it becomes 8,000 and is fine. |

## TR-10. Visual Tokens

| ID | Status | Evidence |
|---|---|---|
| TR-10.1 | ❌ | Constants are scattered: `RIVER_LENGTH`, `RIVER_HW` at line 1894–1895; `ROCK_THRESH`, `WATER_SCALE` at 1918–1919; `WATER_LEVELS`, `BAND_COLORS`, `ROCK_COLOR`, `LINE_COLOR` at 1948–1951; `ACT2_COLS`, `ACT2_ROWS` at 2125. The named warp/halo constants don't exist. |
| TR-10.2 | 🟡 | Palette arrays are clustered (1948–1951) but mixed with thresholds. |

## TR-11. Verification Hooks

| ID | Status | Evidence |
|---|---|---|
| TR-11.1 | ❌ | No debug overlay. |
| TR-11.2 | ❌ | n/a — `WARP_AMPLITUDE` doesn't exist as a separable knob. |
| TR-11.3 | ✅ | Implicitly works: `S` calls `k.scrollY -= 0.003 * dt * 60` (line 2062) which decrements scrollY, scrolling the river upstream while the kayak stays at top. (The visible effect depends on the back-paddle being strong enough to overcome the auto-scroll on line 2058 — currently 0.003 vs 0.003 + flowOffset, so at idle this nets near zero. Worth tuning, but the architecture is correct.) |

---

## Summary scoreboard

| Category | Met | Partial | Wrong | Missing |
|---|---|---|---|---|
| Avatar/world inversion (TR-1) | 3 | 0 | 0 | 0 |
| Field sampling (TR-2) | 1 | 0 | 0 | 1 |
| Grid + upscale (TR-3) | 0 | 1 | 1 | 1 |
| Adaptive levels (TR-4) | 1 | 0 | 1 | 1 |
| Warp halo (TR-5) | 0 | 0 | 1 | 2 |
| Input (TR-6) | 2 | 0 | 0 | 0 |
| Shore detail (TR-7) | 0 | 0 | 0 | 3 |
| Visual cohesion (TR-8) | 0 | 0 | 3 | 0 |
| Performance (TR-9) | 1 | 1 | 1 | 0 |
| Tokens (TR-10) | 0 | 1 | 0 | 1 |
| Debug (TR-11) | 1 | 0 | 0 | 2 |
| **Total** | **9** | **3** | **7** | **11** |

Compare with the original Puget Sound audit (2 met / 6 partial / 6 wrong / 20 missing): Act 2 here is in dramatically better shape because the avatar-static / world-active inversion is already in place. The remaining work is **rendering, not architecture**.

---

## Root cause of the user's complaint

> "rn it's a visual approximation without any of the actual function and refined visuals behind it"

The user's read is half-right: the **function** (avatar-static, scrolling, marching squares, WASD) is genuinely present. What's missing is the **refined visuals** that would make the function visible:

1. **Adaptive levels** — without them, the contour bands don't track the actual height range of what you're paddling through, so the visual stops *meaning* anything as the river changes character.
2. **Warp halo** — without it, the kayak feels like a sprite floating over a procedural texture, not an agent affecting the world.
3. **Palette continuity** — the cold blue makes Act 2 feel like a different game from Act 1, not a continuation of the same world.
4. **Shore detail markers** — flat shore fill removes the "this is the same world as Act 1's forest, just from above" reading.

Fix those four and the function becomes visible.

---

## Recommended order of work

1. **Reduce grid + adaptive levels (TR-3.1, TR-3.2, TR-4.1, TR-4.3).** Replaces 5–10 lines, immediate visual payoff and frees performance budget.
2. **Palette swap (TR-8.1, TR-8.2, TR-8.3).** ~10 lines of constants. Re-establishes Act 1 ↔ Act 2 continuity.
3. **Warp halo (TR-5.1, TR-5.2).** Add `k.warp` updates in `updateKayak`; in the marching squares loop, jitter segments inside `WARP_HALO_RADIUS` of the kayak's screen position.
4. **Shore detail dots (TR-7).** Add a deterministic-hashed dot pass before contour lines.
5. **Token consolidation (TR-10).** Move all Act 2 constants into one labeled block.
6. **Debug overlay (TR-11.1).** Last, for verifying the above.

Steps 1 and 2 alone change the *feeling* of Act 2 the most. Steps 3 and 4 close the gap to Act 1.
