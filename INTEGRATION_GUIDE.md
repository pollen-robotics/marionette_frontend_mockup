# Integration Guide — Marionette Frontend Redesign

This document explains how to integrate the redesigned frontend mockup
(`marionette_frontend_mockup/index.html`) into the real Marionette app
(`marionette/marionette/static/`). It is written for a coding agent (or
developer) who has never seen this codebase before.

---

## Table of Contents

1. [Overview](#overview)
2. [File Mapping](#file-mapping)
3. [Step-by-Step Integration](#step-by-step-integration)
4. [CSS Architecture](#css-architecture)
5. [JavaScript Architecture](#javascript-architecture)
6. [Mock Mode Removal](#mock-mode-removal)
7. [Font Hosting](#font-hosting)
8. [API Contract](#api-contract)
9. [Phase Overlay (Countdown / Recording)](#phase-overlay)
10. [Design Decisions & Rationale](#design-decisions)
11. [What to Keep vs What to Change](#what-to-keep-vs-change)
12. [Testing the Integration](#testing)
13. [Known Limitations](#known-limitations)

---

## 1. Overview <a id="overview"></a>

The mockup is a single `index.html` file (2318 lines) containing inline CSS
and JavaScript. It is a complete, working frontend that can run in two modes:

- **Mock mode** (`?mock=true`): All API calls are intercepted by `mockFetch()`
  which simulates the backend with real timers. No server needed.
- **Production mode** (default): All API calls go to the same origin
  (`/api/...`), which is the real Marionette backend on port 8042.

The redesign addresses these UX problems from the original frontend:
- Cluttered layout (all features visible at once)
- Countdown/recording phases looked identical (users confused them)
- Jargon-heavy labels
- Record action buried among equal-looking panels
- Audio source shown too prominently (most users use mic)
- Description field nobody used (removed entirely)

## 2. File Mapping <a id="file-mapping"></a>

### Mockup files → Production files

| Mockup file | Production target | Notes |
|---|---|---|
| `index.html` (lines 1–1037, `<style>` block) | `marionette/marionette/static/style.css` | Replace entire file |
| `index.html` (lines 1039–1222, `<body>` HTML) | `marionette/marionette/static/index.html` | Replace entire file |
| `index.html` (lines 1224–2318, `<script>` block) | `marionette/marionette/static/main.js` | Replace entire file |
| `fonts/*.woff2` | `marionette/marionette/static/fonts/*.woff2` | New directory |

### Font files needed (minimum set)

The fonts directory has some duplicate files (sora-300/400/600/700/800 are
all identical because Sora is a **variable font** — one file covers weights
300–800). Only these files are actually needed:

| File | Purpose | Size |
|---|---|---|
| `sora-400.woff2` | Sora variable font (weights 300–800) | 33 KB |
| `sora-latin-ext.woff2` | Sora extended Latin characters | 15 KB |
| `dm-mono-400.woff2` | DM Mono regular weight | 15 KB |
| `dm-mono-500.woff2` | DM Mono medium weight | 15 KB |
| `dm-mono-400-latin-ext.woff2` | DM Mono 400 extended Latin | 10 KB |
| `dm-mono-500-latin-ext.woff2` | DM Mono 500 extended Latin | 10 KB |

You can delete the other copies (sora-300, sora-600, sora-700, sora-800,
sora-latin, dm-mono-400-latin, dm-mono-500-latin) — they're redundant.

### Files NOT in mockup (keep from original)

| File | Notes |
|---|---|
| `marionette/marionette/static/test_coverage.json` | Test coverage data — keep as-is |

## 3. Step-by-Step Integration <a id="step-by-step-integration"></a>

### Step 1: Create the fonts directory

```bash
mkdir -p marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/sora-400.woff2 marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/sora-latin-ext.woff2 marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/dm-mono-400.woff2 marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/dm-mono-500.woff2 marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/dm-mono-400-latin-ext.woff2 marionette/marionette/static/fonts/
cp marionette_frontend_mockup/fonts/dm-mono-500-latin-ext.woff2 marionette/marionette/static/fonts/
```

### Step 2: Extract CSS from mockup into style.css

Copy lines 25–1037 from `index.html` (everything between `<style>` and
`</style>`) into `marionette/marionette/static/style.css`.

Then update font paths from `./fonts/` to `/static/fonts/`:

```css
/* Change this: */
src: url('./fonts/sora-400.woff2') format('woff2');
/* To this: */
src: url('/static/fonts/sora-400.woff2') format('woff2');
```

Do the same for all three `@font-face` rules (sora-400, dm-mono-400,
dm-mono-500).

Also **remove** the mock banner CSS (lines 134–144, `.mock-banner` rule).

### Step 3: Extract HTML from mockup into index.html

The production `index.html` needs:
1. The `<head>` section with `<link rel="stylesheet" href="/static/style.css">`
2. The `<body>` content from the mockup (lines 1039–1222)
3. A `<script src="/static/main.js"></script>` tag at the end

**Remove** the mock banner div (`<div class="mock-banner" ...>`).

Here's the target structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <title>Marionette</title>
  <link rel="stylesheet" href="/static/style.css"/>
</head>
<body>
  <!-- Everything from <div class="app"> through the settings drawer -->
  <!-- EXCEPT the mock-banner div -->

  <script src="/static/main.js"></script>
</body>
</html>
```

### Step 4: Extract JavaScript from mockup into main.js

Copy lines 1224–2318 (everything between `<script>` and `</script>`) into
`marionette/marionette/static/main.js`.

Then **remove mock mode** (see [Section 6](#mock-mode-removal)).

### Step 5: Update the backend's static file serving

The backend (`marionette/main.py`) already serves `/static/` from
`marionette/static/`. No changes needed unless the path structure changes.

Verify that the `fonts/` subdirectory is served. FastAPI's `StaticFiles`
mount serves subdirectories by default, so `GET /static/fonts/sora-400.woff2`
should work automatically.

### Step 6: Test

1. Start the backend: `cd marionette && python -m marionette.main`
2. Open `http://localhost:8042` in Chrome and Firefox
3. Verify fonts load (check Network tab — no 404s on font files)
4. Verify state polling works (mode badge updates)
5. Test a recording cycle (countdown → recording → idle)
6. Run existing E2E tests: `pytest tests/e2e --browser chromium`

## 4. CSS Architecture <a id="css-architecture"></a>

### Custom Properties (CSS Variables)

All colors, radii, and font families are defined as CSS variables on `:root`
(lines 64–91). To change the color scheme, only modify these values:

```css
:root {
  --bg: #09090b;            /* Page background */
  --surface: #18181b;       /* Card/panel background */
  --surface-2: #27272a;     /* Elevated surface (hover states) */
  --border: #3f3f46;        /* Border color */
  --text: #fafafa;          /* Primary text */
  --text-muted: #a1a1aa;    /* Secondary text */
  --accent: #e11d48;        /* Red — recording, primary actions */
  --blue: #38bdf8;          /* Blue — countdown phase */
  --green: #34d399;         /* Green — playback, success */
  --orange: #fb923c;        /* Orange — HF upload actions */
  --radius: 16px;           /* Card border radius */
  --radius-sm: 10px;        /* Input/button border radius */
  --font-body: 'Sora', system-ui, sans-serif;
  --font-mono: 'DM Mono', 'Menlo', monospace;
}
```

### CSS Sections (in order)

| Section | Lines | Description |
|---|---|---|
| Font faces | 34–54 | `@font-face` declarations for self-hosted fonts |
| Reset & variables | 56–112 | Box-sizing reset, `:root` variables, body styles, noise texture |
| Layout | 114–126 | `.app` container — centered column, max 720px |
| Mock banner | 128–144 | **REMOVE** — mockup-only |
| Header | 146–231 | Logo, wordmark, mode badge, gear icon |
| Record hero | 233–406 | Giant record button, name/duration fields, hint text |
| Phase overlay | 408–520 | Full-screen countdown/recording display |
| Section tabs | 522–570 | "Moves" and "Community" tab bar |
| Moves list | 572–775 | Move cards, checkboxes, upload bar, dataset selector |
| Community | 777–844 | Community dataset cards |
| Settings drawer | 846–1008 | Slide-in drawer with all settings sections |
| Responsive | 1010–1024 | Mobile breakpoint at 600px |
| Animations | 1026–1037 | `fadeIn` keyframes for page load |

### Key CSS Patterns

- **`data-mode` attribute**: The mode badge uses `$modeBadge.dataset.mode`
  for color switching via `[data-mode="recording"]` selectors.
- **`:has()` selector**: Used for radio button styling
  (`.settings-radio:has(input:checked)`). Works in all modern browsers.
- **`clamp()`**: Phase overlay number uses `clamp(8rem, 20vw, 14rem)` for
  responsive sizing without media queries.
- **Noise texture**: `body::before` has an SVG-based noise pattern using a
  data URI. Pure CSS, no image file needed.

## 5. JavaScript Architecture <a id="javascript-architecture"></a>

The JS is organized into 12 clearly labeled sections:

| Section | Lines | Description |
|---|---|---|
| 1. Mock mode | 1268–1538 | **REMOVE** for production |
| 2. App state & refs | 1540–1603 | Global variables and DOM `getElementById` refs |
| 3. State polling | 1606–1628 | `fetchState()` + `startPolling()` at 1500ms interval |
| 4. UI update | 1631–1709 | `updateUI(state)` — the main sync function |
| 5. Phase overlay | 1712–1772 | Timing math + `requestAnimationFrame` loop |
| 6. Dataset UI | 1775–1789 | `updateDatasetUI()` — populates dataset dropdown |
| 7. Moves rendering | 1792–1896 | `renderMoves()` + `reconcileMoves()` + upload bar |
| 8. User actions | 1899–2051 | API call functions (record, play, delete, upload, etc.) |
| 9. Community | 2054–2132 | Fetch, render, download community datasets |
| 10. Experimental | 2135–2159 | Motion model toggle and selector |
| 11. Event listeners | 2162–2304 | All DOM event bindings |
| 12. Init | 2307–2316 | Mock banner display + start polling |

### Data Flow

```
Backend /api/state ──poll──> fetchState() ──> updateUI(state) ──> DOM
                                                ├── updatePhase(state)
                                                ├── renderMoves(moves)
                                                ├── updateDatasetUI(datasets)
                                                └── updateExperimentalUI(config)

User click ──> action function (e.g. queueRecording)
              ├── POST to /api/record
              └── fetchState() → updateUI() → DOM
```

### State Variables

```javascript
let lastState = null;        // Last received state from backend
let busy = false;            // True when mode is not idle/queued
let clockOffset = 0;         // server_time - local_time (seconds)
let phaseMode = 'idle';      // Current phase: idle/countdown/recording
let phaseStartAt = null;     // Server timestamp: phase start
let phaseEndAt = null;       // Server timestamp: phase end
let selectedMoves = new Set(); // Move IDs selected for HF upload
let communitySelection = new Set(); // Community repo IDs for download
let uploadedAudioId = null;  // ID from /api/upload-audio
let pollingHandle = null;    // setInterval handle
let stateSeq = 0;            // Monotonic counter for out-of-order protection
let latestSeq = 0;           // Highest processed seq
let hfUsername = null;        // Auto-detected from backend
```

### Out-of-Order Protection

`stateSeq` increments on every poll. The response handler checks
`if (id < latestSeq) return;` to discard stale responses. This prevents
a slow response from overwriting a newer one.

## 6. Mock Mode Removal <a id="mock-mode-removal"></a>

The mock mode code is cleanly isolated. To remove it:

### Option A: Delete and replace (recommended)

1. **Delete** everything between `// BEGIN MOCK` and `// END MOCK` markers
   (lines 1276–1538 in the mockup).
2. **Delete** the `MOCK` constant and `apiFetch` wrapper.
3. **Replace** all `apiFetch(` calls with `fetch(` throughout the file
   (approximately 15 occurrences).
4. **Delete** the `$mockBanner` reference and the init line that shows it.
5. **Delete** the `.mock-banner` div from HTML and CSS.

### Option B: Minimal change

Change the `apiFetch` function to a plain passthrough:

```javascript
// This replaces the entire mock section:
function apiFetch(url, options) {
  return fetch(url, options);
}
```

This is simpler but leaves a useless wrapper function.

### Occurrences of apiFetch to replace

Search for `apiFetch(` — it appears in these functions:
- `fetchState()`
- `queueRecording()`
- `queuePlayback()`
- `stopPlayback()`
- `stopRecording()`
- `deleteMove()`
- `selectDataset()`
- `createDataset()`
- `syncToHF()`
- `updateDatasetRoot()`
- `uploadAudioFile()`
- `fetchCommunity()`
- `downloadCommunity()`
- `$featureMotionModels` change handler
- `$motionModelSelect` change handler
- `$recDuration` change handler

## 7. Font Hosting <a id="font-hosting"></a>

### Why self-hosted?

The Reachy Mini robot (a Raspberry Pi CM4) may not have internet access.
Google Fonts requires external HTTP requests, which would fail silently and
fall back to system fonts. Self-hosting ensures the design always looks
correct.

### Font details

**Sora** — The main UI font. A geometric sans-serif with a modern, clean look.
- Type: Variable font (one file covers all weights)
- Weights used: 300 (light, not currently used), 400 (body), 500 (labels),
  600 (sublabels), 700 (headings, wordmark), 800 (phase numbers, logo)
- File: `sora-400.woff2` (33 KB) — covers the full weight range
- License: OFL (Open Font License)

**DM Mono** — Monospace font for metadata, badges, and code-like elements.
- Type: Static font (separate files per weight)
- Weights used: 400 (normal), 500 (medium — used in mode badge)
- Files: `dm-mono-400.woff2` (15 KB), `dm-mono-500.woff2` (15 KB)
- License: OFL

### @font-face declarations

```css
@font-face {
  font-family: 'Sora';
  src: url('/static/fonts/sora-400.woff2') format('woff2');
  font-weight: 300 800;  /* Variable font range */
  font-style: normal;
  font-display: swap;    /* Show fallback font immediately, swap when loaded */
}
@font-face {
  font-family: 'DM Mono';
  src: url('/static/fonts/dm-mono-400.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}
@font-face {
  font-family: 'DM Mono';
  src: url('/static/fonts/dm-mono-500.woff2') format('woff2');
  font-weight: 500;
  font-style: normal;
  font-display: swap;
}
```

### Fallback stack

Both font families have system fallbacks:
- Body: `'Sora', system-ui, -apple-system, sans-serif`
- Mono: `'DM Mono', 'Menlo', 'Consolas', monospace`

If fonts fail to load, the design degrades gracefully to system fonts.

## 8. API Contract <a id="api-contract"></a>

The mockup uses exactly the same API endpoints as the current frontend.
No backend changes are needed. Here is the complete contract:

### Polling

| Method | Endpoint | Request | Response |
|---|---|---|---|
| GET | `/api/state` | `cache: 'no-store'` | Full state object (see below) |

Polled every 1500ms. The `cache: 'no-store'` header prevents browser caching.

### Recording

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| POST | `/api/record` | `{ duration, record_audio, label, uploaded_audio_id? }` | Starts 3s countdown then recording |
| POST | `/api/record/stop` | `{}` | Cancels countdown or stops recording |

### Playback

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| POST | `/api/play` | `{ move_id }` | Starts playing a recorded move |
| POST | `/api/play/stop` | `{}` | Stops current playback |

### Move Management

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| DELETE | `/api/moves/{move_id}` | — | Permanently deletes a move |

### Datasets

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| GET | `/api/datasets` | — | List all datasets |
| POST | `/api/datasets` | `{ name, label? }` | Create new dataset |
| POST | `/api/datasets/select` | `{ dataset_id }` | Switch active dataset |
| POST | `/api/datasets/root` | `{ path }` | Change datasets folder |
| POST | `/api/datasets/sync` | `{ hf_username, move_ids }` | Upload moves to HF |
| GET | `/api/datasets/community` | — | List community datasets |
| POST | `/api/datasets/download` | `{ repo_id, label? }` | Download a community dataset |

### Audio Upload

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| POST | `/api/upload-audio` | `FormData { file }` | Upload WAV/MP3 for playback |

### Experimental Features

| Method | Endpoint | Request Body | Notes |
|---|---|---|---|
| POST | `/api/experiments` | `{ motion_models?, duration_seconds? }` | Toggle features |
| POST | `/api/motion-model` | `{ name }` | Select active motion model |

### State Object Shape

```json
{
  "server_time": 1234567890.123,
  "mode": "idle|countdown|recording|playing|queued|starting_up|error",
  "message": "Human-readable status text",
  "active_move": "move-id or null",
  "phase_start_at": null,
  "phase_end_at": null,
  "recording_stats": null,
  "moves": [
    {
      "id": "happy-dance",
      "label": "happy-dance",
      "duration": 5.2,
      "created_at": 1234567890,
      "has_audio": true,
      "is_uploaded": false
    }
  ],
  "config": {
    "preferred_duration": 5.0,
    "default_duration": 5.0,
    "countdown_seconds": 3,
    "audio_available": true,
    "hf_username": "RemiFabre or null",
    "dataset_root_path": "/path/to/datasets",
    "active_dataset_path": "/path/to/active/dataset/data",
    "features": { "motion_models": false },
    "feature_support": { "motion_models": true },
    "motion_models": null
  },
  "datasets": {
    "active_id": "default",
    "root_path": "/path/to/datasets",
    "entries": [
      {
        "id": "default",
        "label": "Local dataset",
        "path": "/path/to/datasets/local_dataset",
        "origin": "local"
      }
    ]
  }
}
```

## 9. Phase Overlay (Countdown / Recording) <a id="phase-overlay"></a>

This is the most important UX improvement in the redesign. The original app
showed countdown and recording identically — users couldn't tell which phase
they were in.

### How it works

1. When the backend returns `mode: "countdown"` with `phase_start_at` and
   `phase_end_at`, the overlay appears with a **blue** background and shows
   large integers counting down: **3**, **2**, **1**.

2. When mode changes to `"recording"`, the overlay switches to a **red**
   background with a decimal countdown (e.g., **4.2s**) and a blinking red
   dot.

3. The overlay disappears when mode returns to `"idle"`.

### Timing math

```javascript
clockOffset = server_time - (Date.now() / 1000);
now = Date.now() / 1000 + clockOffset;  // Corrected local time
remaining = phase_end_at - now;
ratio = remaining / (phase_end_at - phase_start_at);  // 1.0 → 0.0
```

The `clockOffset` is recalculated on every state poll. This handles:
- Clock drift between client and server
- Network latency (partially — it's not RTT-corrected, but good enough
  for a 1500ms polling interval)

### Animation loop

```javascript
function animationLoop() {
  updatePhaseDisplay();
  requestAnimationFrame(animationLoop);
}
animationLoop();
```

This runs at 60fps and updates the overlay's number display and progress bar.
It's intentionally always running (even when the overlay is hidden) to avoid
setup/teardown complexity. The cost is negligible since `updatePhaseDisplay()`
short-circuits when `phaseStartAt == null`.

### Visual comparison

| Property | Countdown | Recording |
|---|---|---|
| Background | Blue radial gradient | Red radial gradient |
| CSS class | `.countdown` | `.recording` |
| Number | Integer: `3`, `2`, `1` | Decimal: `4.2s` |
| Number size | 8–14rem (huge) | 4–7rem (large) |
| Text color | `--blue` (#38bdf8) | `--accent` (#e11d48) |
| Sublabel | "GET READY" | "RECORDING" |
| Indicator | Pulsing number | Blinking red dot |
| Progress bar | Blue fill | Red fill |
| Stop button | "Cancel" | "Stop Recording" |

## 10. Design Decisions & Rationale <a id="design-decisions"></a>

### Hero record button

The record button is 140x180px (button + ring), deliberately oversized.
First-time users should immediately understand: "I press this to start."
The button has:
- A white circle inside (universal "record" icon)
- The word "RECORD" below the circle
- A subtle glow ring that intensifies on hover
- Three distinct visual states: default (red), countdown (blue), recording
  (red with stop icon)

### Settings in a drawer (not inline)

Audio source, HF login, storage folder, and experimental features are in a
slide-out drawer behind the gear icon. This removes clutter from the main
view. Most users never change these settings.

### Tabs for Moves/Community

The moves list and community browser are in separate tabs. Most users spend
99% of their time in "Moves." The community tab only loads data when you
click "Fetch community datasets" — no automatic loading.

### HF username is auto-detected

The HF login field is **read-only**. The backend's `config.hf_username`
(from `huggingface-cli whoami`) determines the login state. There is no
text input. If not logged in, a hint says to run `huggingface-cli login`.

### No description field

The original app had a "Description" field that nobody ever filled in.
It has been removed entirely.

### Downloaded dataset warning

When the active dataset has `origin: "downloaded"`, a warning appears below
the record button: "Switch to a local dataset to record new moves." The
record button is also disabled.

### Duration step="any"

The mockup uses `step="any"` on the duration input (not `step="0.5"` like
the original). This fixes a bug where the HTML5 validation rejected
durations like `7.3` that came from uploaded audio files.

## 11. What to Keep vs What to Change <a id="what-to-keep-vs-change"></a>

### From the mockup — KEEP everything except:

| Remove | Reason |
|---|---|
| Mock banner HTML + CSS + JS | Only for mockup previewing |
| `MOCK` constant | Not needed in production |
| `mockState` object | Not needed in production |
| `mockFetch()` function | Not needed in production |
| `mockCommunityDatasets` array | Not needed in production |
| `mockRecordTimer` / `mockPlayTimer` | Not needed in production |
| `apiFetch()` wrapper | Replace with plain `fetch()` |
| `$mockBanner` DOM reference | Not needed in production |
| Init line `if (MOCK && $mockBanner)...` | Not needed in production |

### From the original frontend — DO NOT carry over:

| Original feature | Status |
|---|---|
| Description field | Removed by design |
| Inline audio source selector | Moved to settings drawer |
| Inline HF username input | Replaced with auto-detected read-only display |
| All-panels-visible layout | Replaced with hero + tabs + drawer |
| Progress bar in main view | Replaced with full-screen overlay |
| Community datasets in main view | Moved to separate tab |

### From the original frontend — CONSIDER carrying over:

| Feature | Notes |
|---|---|
| Status message bar | The mockup shows status in the mode badge; the original had a full message bar. Could add `state.message` display somewhere. |
| Recording stats display | After recording, `state.recording_stats` has pose count and rate. The mockup doesn't display this. Could show as a toast/notification. |
| Error mode handling | The mockup handles `starting_up` and `error` modes minimally. May want to add a more prominent error display. |
| `localStorage` persistence | The original saved `rec-name` to localStorage. The mockup doesn't. Consider adding if users want name persistence. |

## 12. Testing the Integration <a id="testing"></a>

### Manual testing checklist

- [ ] Fonts load correctly (no 404s in Network tab)
- [ ] State polling works (mode badge shows "IDLE")
- [ ] Record button starts countdown (blue overlay, 3-2-1)
- [ ] Recording phase is visually distinct (red overlay, decimal timer)
- [ ] Overlay disappears after recording completes
- [ ] New move appears in moves list
- [ ] Play button works and highlights the playing card
- [ ] Delete button shows confirmation dialog
- [ ] Dataset selector lists all datasets
- [ ] New dataset creation works
- [ ] Downloaded dataset shows warning + disables record
- [ ] Settings drawer opens/closes
- [ ] Audio source switching works (mic/upload/silent)
- [ ] Audio file upload works (drag-and-drop + click)
- [ ] HF username displays correctly (or "Not logged in")
- [ ] Upload bar appears when moves are checkbox-selected
- [ ] Community tab fetches and displays datasets
- [ ] Community dataset download works
- [ ] Responsive layout works on narrow screens

### Automated tests

The existing E2E tests (`tests/e2e/test_ui.py`) will need updates because
the DOM structure has changed significantly. Key changes:

| Original selector | New selector | Notes |
|---|---|---|
| `#start-btn` | `#record-btn` | Record button ID changed |
| `#move-label` | `#rec-name` | Name input ID changed |
| `#duration` | `#rec-duration` | Duration input ID changed |
| `.panel` | `.record-hero` | Recording section class changed |
| `#moves-section` | `#tab-moves` | Moves container ID changed |
| `.move-item` | `.move-card` | Move card class changed |
| `#settings-section` | `#settings-drawer` | Settings container changed |
| `.progress-bar` | `.phase-overlay` | Progress display changed |

## 13. Known Limitations <a id="known-limitations"></a>

1. **No toast/notification system**: The mockup doesn't show success/error
   messages after actions. The mode badge updates, but there's no explicit
   feedback like "Recorded happy-dance (5.2s, 520 poses)." Consider adding
   a simple toast component.

2. **No keyboard shortcuts**: The original had none either, but the record
   button could benefit from a spacebar shortcut.

3. **No loading states**: When the backend is slow, there's no visual
   feedback on buttons (except the upload button which shows "Uploading...").
   Consider adding a loading spinner or disabled state to all action buttons.

4. **Community downloads are sequential**: If you select 3 community datasets,
   they download one at a time in a `for` loop. Could be parallelized.

5. **No error recovery for phase timing**: If the phase overlay gets stuck
   (e.g., backend crashes during recording), it will show stale data until
   the next poll returns a different mode. The original app had the same issue.

6. **Logo placeholder**: The "M" in a colored square is a placeholder. Replace
   with a real logo SVG or image when available.

---

## Quick Reference: Mockup Preview

To preview the mockup without any backend:

```bash
cd marionette_frontend_mockup
python -m http.server 8080
# Then open http://localhost:8080/index.html?mock=true
```

To preview against the real backend:

```bash
# Terminal 1: Start backend
cd marionette && python -m marionette.main

# Terminal 2: Serve mockup
cd marionette_frontend_mockup && python -m http.server 8080
# Open http://localhost:8080/index.html
# Note: CORS will block API calls unless the backend allows the origin.
# For testing, use the mock mode instead.
```

The recommended way to test integration is to copy the files into
`marionette/marionette/static/` and access via `http://localhost:8042`.
