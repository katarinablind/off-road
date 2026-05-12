# Off-Road Topology

A browser-based 3D off-road driving game built as a single HTML file. Drive from a meadow trailhead to a mountain lake through four biomes, while trying to minimize terrain damage.

---

## Current State (as of March 2026)

The game is mid-redesign. It started as a **2D topographic map** with a 3D car model on top. It is now a **full 3D behind-car perspective** with a terrain mesh, contour lines rendered as 3D geometry, and an arcing sun. The redesign is partway through a 6-phase plan.

**What's working right now:**
- 3D terrain mesh, flat-shaded, vertex-colored by elevation and biome
- Third-person camera follows car from behind (FOV 72°, close enough that car fills ~half screen)
- Topology / contour lines rendered as Three.js `LineSegments` via marching squares on the height field — **these are a hard requirement and must survive any future refactor**
- Contour lines distort when off-road (warp > 0), ripple near rivers (sine-wave animation)
- Sun sphere arcs east → zenith → west as drive progresses, color shifting from dawn-orange to noon-white to sunset-red
- Four biomes: Meadow → Coastal Cliffs → Rainforest → Alpine
- Four time-of-day themes: Dawn / Day / Golden Hour / Sunset — smooth lerp transitions
- Car is 2× scaled, pitch/roll follows terrain slope (cosmetic only — no tipping)
- Three rivers with bridges; off-bridge river crossing damages the car heavily
- Web Audio: engine (3 RPM layers), wind, birds, water proximity, guitar loop, foliage
- WebRTC multiplayer (PeerJS), split-screen via scissor test
- Finish: car parks at Mirror Lake, stats screen

**What's not done yet (Phases 2–6 of redesign):**
- Phase 2: Surface detail — visible bumps, potholes, tire tracks on terrain
- Phase 3: Scenery — instanced trees, cliff faces, rocks
- Phase 4: Act 2 kayaking section (same noise field = river)
- Phase 5: Driving → Kayak cutscene
- Phase 6: Combined score screen, polish

---

## Tech Stack

Single `index.html` (~1900 lines). No build step. Open in browser.

| Layer | What |
|---|---|
| **3D rendering** | Three.js (CDN ES module) — terrain mesh, car model, contour lines, sun |
| **Physics** | Vanilla JS, 2D normalized coords (`nx` 0–1, `scrollOff` 0–3) |
| **Noise** | Custom Perlin + fBM (5-octave for physics, 3-octave for mesh) |
| **Audio** | Web Audio API, real CC0 recordings |
| **Multiplayer** | PeerJS WebRTC, seeded PRNG for identical terrain |
| **UI** | HTML/CSS overlay on WebGL canvas |

### Key constants

```js
WORLD_SCALE = 80      // terrain width in world units
HEIGHT_SCALE = 12     // max terrain elevation
TERRAIN_SEGS = 65     // mesh grid (65×65 faces)
FINISH_DIST = 3       // drive distance in scroll-space (halved from original 6)
```

### Coordinate system

- Physics live in normalized space: `nx` (0–1 lateral), `scrollOff` (0→FINISH_DIST forward)
- 3D world: `wx = (nx - 0.5) * 80`, `wz = (sY - carSY) * 80`, `wy = height * 12`
- "Ahead" = negative Z (camera looks toward −Z); "behind" = positive Z
- `sY = carNY − scrollOff * 5` (decreases as car drives forward)

### The topology line system

This is the most important subsystem. `updateContourLines(scrollOff, warp, t)`:
1. Reads `hCache` (Float32Array of heights at each terrain grid vertex, populated by `updateTerrainMesh`)
2. Runs marching squares across 8 elevation levels
3. Converts edge crossings to 3D world positions (lifted 0.35 units above surface)
4. Writes to a pre-allocated `LineSegments` buffer, updates draw range
5. Off-road distortion: `nx += noise2d(…) * warp * 0.04`
6. Water ripple: `wy += sin(nx*10 + t*2.5) * 0.25` near river sY positions

Colors use a warm sienna/brown — classic topographic map style — to contrast against green and grey terrain.

---

## Redesign Process (Where We Are)

### Phase 1 — 3D terrain + camera ✅ done
Replaced the 2D canvas entirely with a Three.js `PlaneGeometry` terrain mesh. Switched from a top-down `OrthographicCamera` to a `PerspectiveCamera` behind the car. Preserved all physics, audio, and multiplayer unchanged.

Key problems solved in Phase 1:
- **Terrain direction bug**: `wz` sign was inverted, putting the terrain *behind* the camera. Fixed to `wz = (sY - carSY) * WORLD_SCALE`.
- **Performance**: Reduced from 120→65 mesh segments, added `fbm3` (3-octave fast version), throttled terrain updates to scroll delta > 0.008.
- **Topology lines in 3D**: Added marching squares contour geometry after user made this a hard requirement mid-Phase 1.

### Phases 2–6 — not started
See the 6-phase redesign document (originally provided as a PDF, pasted as text in the session that started Phase 1).

---

## How This Project Is Being Built

This project is a collaboration between **Katarina** (design, vision, feedback) and **Claude Code** (implementation). There is no traditional development workflow — no tickets, no PRs, no staging. It runs as a single file opened directly in a browser.

### Katarina's approach to prompting

**Vision-first, details-later.** Large design direction is given upfront ("transform this into a 3D driving game with these 6 phases"), followed by tight iterative loops on specific problems ("the car is too bouncy", "topology lines must stay", "the terrain goes the wrong way").

**Screenshot-driven feedback.** The output of each session is judged by looking at a screenshot or playing the game. Feedback is visual and felt, not metric ("it feels like drifting", "it's too commercial", "it's like airbrush not terrain").

**Non-negotiables are stated explicitly.** When something is a hard requirement — like topology lines — it's flagged directly: *"topology lines are requirements"*. This is the signal to preserve that system across any future refactor, even if it means reverting to 2D.

**Willingness to revert.** Multiple times the direction included an escape hatch: *"if 3D is too tricky for that, let's move back to 2D"*. This keeps the project pragmatic — a working 2D game is better than a broken 3D one.

**Parallel concerns in one message.** A single prompt might cover: visual bugs, audio issues, a narrative addition (cabin), a UI change (move progress bar), and a color note — all at once. Claude works through these as a bundle rather than asking to serialize them.

**The map is the mechanic.** The recurring design insight across all sessions: the topographic map isn't decoration. It reacts — distorting when the car goes off-road, rippling near water. The player is driving *on* the map, not just *through* a landscape.

### What Claude does in each session

1. Reads the current state of `index.html` and `README.md`
2. Asks clarifying questions only when the escape hatch ("or go back to 2D") indicates a real architectural fork
3. Makes surgical edits — avoiding scope creep, not refactoring things that weren't asked about
4. Takes screenshots after each significant change to verify before reporting back
5. Updates the README and memory files to capture decisions that would be lost otherwise

### Session history (abbreviated)

| Session | Key changes |
|---|---|
| Early | 2D topo map, marching squares, 3D car overlay on transparent canvas |
| Mid | Front-wheel steering, off-road warp system, particle effects, rivers with bridges |
| Before redesign | Country guitar music, cabin request, bluer rivers, progress bar relocation |
| Redesign session 1 | Full rewrite: 2D canvas → Three.js 3D terrain, PerspectiveCamera behind car |
| Redesign session 2 | Topology lines as 3D LineSegments, terrain direction fix, car sizing, sun arc, smoother driving |

---

## Design Principles

1. **Topology lines are non-negotiable.** If 3D can't support them, fall back to 2D.
2. **The map reacts.** Contour lines distort off-road, ripple on water.
3. **Steady car, no tipping.** Pitch/roll is cosmetic. No rigid-body physics.
4. **Single file.** Zero deployment friction.
5. **Real audio only.** No synthesized sounds. All CC0 recordings.
6. **Physics stay 2D.** The 3D is a rendering layer over deterministic 2D physics. Multiplayer works because both clients run the same 2D simulation from the same seed.

---

## Running It

```
open index.html
```

Or serve locally (required for audio file loading in some browsers):

```
python3 -m http.server 3456
# then open http://localhost:3456
```

Controls: **W/S** accelerate/brake · **A/D** steer · **Mute** button for guitar

---

## Credits

All sound files CC0 (public domain) via OpenGameArt:
- Birds: isaiah658
- Wind / foliage: wind whoosh loop
- Water: Vistula river recording
- Engine (3 layers): racing car engine
- Guitar: "Countryside" by TAD
