# 🧩 Live Puzzle — Gesture-Controlled AI Webcam Puzzle Game

**Author:** Dileep Kumar

> A zero-install, browser-native puzzle game controlled entirely by your bare hands. No mouse. No keyboard. Just you and the camera.

---

## ✨ Overview

**Live Puzzle** is a real-time, hand-gesture-controlled puzzle game that runs entirely in the browser — no backend, no build tools, no installation required. Powered by Google's MediaPipe Hands ML model running in WebAssembly, the game lets you:

1. **CAPTURE** — Point your index finger to aim a frame at any scene through your webcam, hold it steady, and snap a photo using a natural pinch gesture.
2. **SOLVE** — The captured image is scrambled into a 3×3 tile grid. Pinch and drag tiles back into the correct order using only your fingers in the air.

The entire experience is contained in a **single `.html` file**.

---

## 🎬 Demo Flow
[ WEBCAM LIVE ] → [ FRAME SCENE WITH FINGER ] → [ HOLD TO LOCK ] → [ PINCH TO CAPTURE ]
↓
[ 3×3 SCRAMBLED PUZZLE ] → [ PINCH + DRAG TO SWAP TILES ] → [ CONFIRM → WIN ]
↓
[ TIMER STOPS ] → [ LEADERBOARD ] → [ REBOOT → PLAY AGAIN ]

---

## 🗂️ Project Structure
live-puzzle.html
├── <style>                    — Full CSS (dark theme, scanlines, animations)
├── <video id="webcam">        — Hidden live webcam stream source
├── <canvas id="game-canvas">  — Main rendering surface (tiles, frame, video)
├── <canvas id="hands-canvas"> — Skeleton overlay (pointer-events: none)
├── <div id="ui-overlay">      — Phase label + solve timer (top-right panel)
├── <div id="reference-panel"> — Shows captured image during solve phase
├── <div id="instructions">    — Bottom bar with contextual hints
├── <button id="confirm-btn">  — Manual solution submission button
├── <div id="leaderboard">     — Modal with top 5 times from localStorage
└── <script>
├── CONFIG object              — All tunable constants
├── state object               — Single source of truth for game state
├── Tile class                 — Per-tile data (id, slot, position, drag state)
├── createPuzzle()             — Slices captured image into 9 tiles
├── shuffleTiles()             — Fisher-Yates randomization of slots
├── checkWin()                 — Validates tile IDs match current slots
├── GestureEngine              — isPinching(), getFrameRect(), getDistance()
├── onResults()                — MediaPipe callback; drives skeleton + game loop
├── updateGame()               — Per-frame state machine updater
├── handleSolveInteraction()   — Pinch-drag-drop tile management
├── dropTile()                 — Releases tile, swaps slots
├── captureScene()             — Crops video frame → creates puzzle
├── renderGame()               — Canvas drawing (video, frame rect, tiles, glow)
├── drawTile()                 — Draws one puzzle tile from source image
├── transitionTo()             — Phase state machine (LOADING→CAPTURE→SOLVE→WIN)
├── updateTimer()              — Interval-based MM:SS.d display
├── saveScore() / showLeaderboard() — localStorage-based score system
└── init()                     — Camera + MediaPipe setup, canvas sizing, boot

---

## 🧠 Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Hand Tracking | [MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker) | Runs in-browser via WebAssembly. Detects 21 landmarks per hand at ~30fps. |
| Camera Access | `navigator.mediaDevices.getUserMedia()` | Native Web API. Streams into hidden `<video>` element. |
| Rendering | HTML5 `<canvas>` (2 layers) | `game-canvas` for puzzle + video; `hands-canvas` for skeleton overlay. |
| Gesture Engine | Custom vanilla JS | Reads MediaPipe landmarks; computes pinch distance, frame rect, drag target. |
| Puzzle Logic | Vanilla JS `Tile` class + slot swapping | Source coords + slot assignment; no canvas transforms required. |
| UI / Styling | Pure CSS with CSS Variables | Dark terminal theme, scanline texture, neon glow, JetBrains Mono font. |
| Persistence | `localStorage` | Top 5 scores saved locally; no backend. |
| Font | [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono) via Google Fonts | Monospace; reinforces the hacker terminal aesthetic. |
| Dependencies | CDN only (`jsdelivr.net`) | No npm, no bundler, no server. |

### CDN Dependencies

```html
<!-- MediaPipe Hands -->
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>

<!-- Font -->
<link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
```

---

## 🤌 Gesture System

MediaPipe returns 21 normalized (0–1) 3D keypoints per hand each frame. The gesture engine maps specific landmark pairs to game actions:

### Landmark Index Reference (MediaPipe)
8 ← Index fingertip
4 ← Thumb tip
12 ← Middle fingertip
0 ← Wrist

### Gesture Table

| Gesture | Detection Method | Threshold | Used For |
|---|---|---|---|
| **Pinch** | Euclidean distance between pt 4 (thumb tip) and pt 8 (index tip) | `< 0.05` (normalized) | Snap photo; grab tile; drop tile |
| **Frame Aim** | Index fingertip (pt 8) position on screen | Fixed 60% viewport box centered on fingertip | Define capture crop zone |
| **Hold Steady** | Frame rect delta between frames | `< 0.03` on both axes | Trigger countdown before capture |
| **Drag** | Pinch held + hand movement | — | Move grabbed tile across grid |
| **Release / Drop** | Pinch distance opens back above threshold | — | Place tile in nearest slot |
| **Skeleton Draw** | All 21 landmarks + `HAND_CONNECTIONS` constant | — | Visual feedback overlay |

### Pinch Countdown Logic
Frame detected → stable for 1500ms → captureScene() fires
↑                  ↑
delta < 0.03     captureStartTime set

---

## 🎮 Game Phases

### Phase 1: CAPTURE

1. Webcam stream mirrors horizontally and renders to `game-canvas`.
2. When an index finger is detected, a **neon green rectangle** (60% of canvas) is drawn centered on the fingertip.
3. Frame stability is tracked frame-by-frame. A 1.5-second countdown begins on lock.
4. On capture: `captureScene()` uses a temporary off-screen canvas to crop and mirror-correct the video frame, storing it as `state.captureFrameImage`.
5. The cropped image is saved to the reference panel (`<img id="ref-image">`).

### Phase 2: SOLVE

1. `createPuzzle()` slices `captureFrameImage` into a 3×3 grid of `Tile` objects (9 total).
2. `shuffleTiles()` performs Fisher-Yates shuffle on slot assignments.
3. Each frame, non-dragged tiles lerp toward their `targetX/Y` at 20% interpolation speed.
4. Pinching over a tile selects it (`isBeingDragged = true`). The tile follows the pinch pointer.
5. The nearest grid slot is highlighted with a dashed neon rectangle as the drop target.
6. Releasing the pinch swaps the dragged tile's slot with whatever tile occupies the target slot.
7. Clicking **CONFIRM** checks if all `tile.id === tile.currentSlot`.

### Phase 3: WIN

1. Timer stops. Neon flash overlay appears.
2. Score (formatted time string) is saved to `localStorage` alongside the date.
3. After 2 seconds, the **Leaderboard modal** slides in showing top 5 times.
4. **REBOOT SYSTEM** resets all state and returns to CAPTURE phase.

---

## ⚙️ Configuration (`CONFIG` object)

```js
const CONFIG = {
    gridSize: 3,              // Grid dimensions (3×3 = 9 tiles)
    pinchThreshold: 0.05,     // Normalized distance for pinch detection
    captureHoldTime: 1000,    // ms to hold frame before auto-capture (UI label only)
    frameColor: '#00ff88',    // Neon green accent color
    tileGlow: 'rgba(0,255,136,0.5)', // Shadow color for lifted tiles
    snapWidth: 450,           // Captured image width (px)
    snapHeight: 450           // Captured image height (px)
};
```

> To change difficulty, increase `gridSize` to `4` (16 tiles) or `5` (25 tiles) — the rest of the system scales automatically.

---

## 🎨 Visual Design

| Element | Detail |
|---|---|
| **Theme** | Cyberpunk terminal / neural-link hacker |
| **Background** | `#0a0a0a` with CSS scanline texture overlay |
| **Accent** | `#00ff88` neon green — borders, glows, labels, skeleton |
| **Font** | `JetBrains Mono` 400 / 700 — all text monospace |
| **Scanlines** | `body::before` — `repeating-linear-gradient` RGB color split |
| **Canvas border** | `2px solid rgba(0,255,136,0.2)` + `box-shadow` neon glow |
| **Tile borders** | `rgba(255,255,255,0.1)` subtle white separator |
| **Dragged tile** | `shadowBlur: 20`, `shadowColor: #00ff88` |
| **Drop target** | Dashed neon rectangle: `setLineDash([5,5])` |
| **Win state** | CSS `@keyframes flash` overlay + `@keyframes cascade` tile exit |
| **Instructions** | `@keyframes pulse` — opacity 0.8↔1 with text-shadow |
| **Loader** | Progress bar fills over 2 seconds while MediaPipe loads |

---

## 🔒 Privacy & Security

- **All ML inference runs locally** — MediaPipe Hands uses WebAssembly; no video data leaves the device.
- **No server, no API calls, no telemetry** — purely client-side.
- **Webcam access** requires explicit browser permission; the stream is never uploaded.
- **Leaderboard** uses `localStorage` — data never leaves the browser.

---

## 🚀 Getting Started

### Requirements

- A modern browser: Chrome 90+, Edge 90+, Firefox 95+, Safari 15.4+
- A webcam
- An internet connection (to load MediaPipe and font from CDN on first load)

### Running Locally

```bash
# Option 1 — Just open it
open live-puzzle.html

# Option 2 — Serve locally (avoids camera permission issues on file://)
npx serve .
# or
python3 -m http.server 8080
# then visit http://localhost:8080/live-puzzle.html
```

> **Note:** Chrome blocks `getUserMedia()` on `file://` URLs. Use a local server if the camera doesn't initialize.

---

## 🐛 Known Issues & Limitations

| Issue | Cause | Workaround |
|---|---|---|
| Camera not starting on `file://` | Chrome security policy | Use `localhost` server |
| Occasional false pinch triggers | Lighting / hand angle | Keep hand well-lit; face palm toward camera |
| MediaPipe WASM load delay | First-time CDN fetch (~3–5MB) | Progress bar covers this; subsequent loads are cached |
| Tile drag jitter at edges | Landmark coordinate clamping | Keep hand centered in frame |
| Leaderboard times sort lexicographically | `String.localeCompare` on `MM:SS.d` format | Works correctly for times under 10 minutes |

---

## 🔧 Potential Improvements

- [ ] Solvability check on shuffle (parity validation for odd-grid puzzles)
- [ ] Two-hand framing (use both index fingertips to define a custom crop rectangle)
- [ ] Difficulty selector (3×3 / 4×4 / 5×5 grid)
- [ ] Sound effects (`AudioContext` synth on pinch, tile snap, win)
- [ ] Mobile support (touch fallback when no gesture available)
- [ ] Confetti win animation via canvas particles
- [ ] Gesture onboarding tutorial overlay on first visit
- [ ] PWA manifest + service worker for offline play

---

## 📄 License

MIT — free to use, modify, and distribute. Attribution appreciated but not required.

---

> *"The interface disappears. Only the puzzle remains."*
