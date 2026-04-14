# iPad Trail Finder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained HTML file that records a walking path via dead reckoning (accelerometer + compass) and guides a second person to the destination via a glowing map and an AirTag-style finder view.

**Architecture:** Single `trail-finder/index.html` — inline HTML, CSS, and vanilla JS. Three app modes (`welcome` → `recording` → `navigating`) share one sensor loop. All state is in JS variables; no backend, no persistence, no dependencies.

**Tech Stack:** Vanilla HTML5/CSS/JS, Canvas 2D API, DeviceMotionEvent (accelerometer step detection), DeviceOrientationEvent (webkitCompassHeading).

---

## File Structure

```
trail-finder/
  index.html    ← entire app (~500 lines, all inline)
```

No other files. No build step. Served via `python3 -m http.server 8080`.

---

## Task 1: HTML Shell + CSS

**Files:**
- Create: `trail-finder/index.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>Trail Finder</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      background: #0d1117;
      color: #e0e0e0;
      font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', sans-serif;
      height: 100dvh;
      overflow: hidden;
    }

    .screen { display: none; flex-direction: column; height: 100dvh; }
    .screen.active { display: flex; }

    /* ── Welcome ── */
    #screen-welcome {
      align-items: center;
      justify-content: center;
      gap: 24px;
      padding: 40px;
      text-align: center;
    }
    #screen-welcome h1 { font-size: 2.5rem; font-weight: 700; letter-spacing: -1px; }
    #screen-welcome p.subtitle { color: #888; font-size: 1rem; max-width: 280px; line-height: 1.5; }
    #btn-start {
      background: #00e5ff; color: #000;
      border: none; border-radius: 32px;
      padding: 16px 40px; font-size: 1.1rem; font-weight: 700;
      cursor: pointer; margin-top: 16px;
    }
    #sensor-error { color: #ff5252; font-size: 0.85rem; display: none; }

    /* ── Recording ── */
    #screen-recording { position: relative; }
    #hud-recording {
      position: absolute; top: 0; left: 0; right: 0; z-index: 10;
      display: flex; justify-content: space-between; align-items: center;
      padding: 16px 20px;
    }
    #rec-indicator { color: #ff5252; font-size: 0.85rem; font-weight: 600; }
    #step-count { color: #00e5ff; font-size: 0.85rem; }
    #map-canvas { width: 100%; flex: 1; display: block; }
    #btn-stop {
      position: absolute; bottom: 40px; left: 50%; transform: translateX(-50%); z-index: 10;
      background: #ff5252; color: #fff;
      border: none; border-radius: 32px;
      padding: 16px 40px; font-size: 1rem; font-weight: 700; cursor: pointer;
    }
    #btn-stop:disabled { background: #444; color: #666; cursor: default; }

    /* ── Navigating ── */
    #screen-navigating { position: relative; }
    .nav-view { display: none; flex-direction: column; height: 100dvh; }
    .nav-view.active { display: flex; }

    #hud-nav {
      position: absolute; top: 0; left: 0; right: 0; z-index: 10;
      display: flex; justify-content: space-between; align-items: center;
      padding: 16px 20px;
    }
    #distance-display { color: #e0e0e0; font-size: 0.9rem; }
    #btn-to-finder {
      background: #00e5ff22; color: #00e5ff;
      border: 1px solid #00e5ff44; border-radius: 12px;
      padding: 6px 16px; font-size: 0.85rem; font-weight: 600; cursor: pointer;
    }
    #nav-canvas { width: 100%; flex: 1; display: block; }

    /* ── Finder view ── */
    #view-finder { align-items: center; justify-content: center; gap: 16px; padding: 40px; }
    #btn-to-map {
      position: absolute; top: 20px; right: 20px;
      background: #00e5ff22; color: #00e5ff;
      border: 1px solid #00e5ff44; border-radius: 12px;
      padding: 6px 16px; font-size: 0.85rem; font-weight: 600; cursor: pointer;
    }
    #finder-rings { position: relative; width: 260px; height: 260px; }
    #finder-distance { font-size: 3rem; font-weight: 700; color: #e0e0e0; }
    #finder-label { color: #888; font-size: 1rem; }
    #arrival-msg { color: #69f0ae; font-size: 1.1rem; font-weight: 600; display: none; }

    @keyframes pulse-ring {
      0%   { opacity: 0.5; }
      50%  { opacity: 0.1; }
      100% { opacity: 0.5; }
    }
    #finder-rings.near circle { animation: pulse-ring 1s ease-in-out infinite; }
  </style>
</head>
<body>

  <!-- Welcome -->
  <div id="screen-welcome" class="screen active">
    <h1>Trail Finder</h1>
    <p class="subtitle">Walk a path. Hand off the iPad. Let someone follow you.</p>
    <button id="btn-start">Start Recording</button>
    <p id="sensor-error">Please open on an iPad and allow motion access.</p>
  </div>

  <!-- Recording -->
  <div id="screen-recording" class="screen">
    <div id="hud-recording">
      <span id="rec-indicator">● REC</span>
      <span id="step-count">0 steps</span>
    </div>
    <canvas id="map-canvas"></canvas>
    <button id="btn-stop" disabled>Stop Recording</button>
  </div>

  <!-- Navigating -->
  <div id="screen-navigating" class="screen">
    <div id="view-map" class="nav-view active">
      <div id="hud-nav">
        <span id="distance-display">-- m to destination</span>
        <button id="btn-to-finder">FINDER</button>
      </div>
      <canvas id="nav-canvas"></canvas>
    </div>
    <div id="view-finder" class="nav-view">
      <button id="btn-to-map">MAP</button>
      <div id="finder-rings">
        <svg width="260" height="260" viewBox="0 0 260 260" id="finder-svg">
          <circle cx="130" cy="130" r="120" fill="none" stroke="#00e5ff" stroke-width="1"   opacity="0.15"/>
          <circle cx="130" cy="130" r="95"  fill="none" stroke="#00e5ff" stroke-width="1.5" opacity="0.3"/>
          <circle cx="130" cy="130" r="68"  fill="none" stroke="#00e5ff" stroke-width="2"   opacity="0.5"/>
          <circle cx="130" cy="130" r="40"  fill="#00e5ff" opacity="0.08"/>
          <g id="finder-arrow" transform="rotate(0, 130, 130)">
            <polygon points="130,60 140,95 130,85 120,95" fill="#00e5ff"/>
            <line x1="130" y1="90" x2="130" y2="160" stroke="#00e5ff" stroke-width="4" stroke-linecap="round"/>
          </g>
          <circle cx="130" cy="130" r="5" fill="#fff" opacity="0.4"/>
        </svg>
      </div>
      <div id="finder-distance">-- m</div>
      <div id="finder-label">to destination</div>
      <div id="arrival-msg">You're close!</div>
    </div>
  </div>

  <script>
    // JS added in Tasks 2–7
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify layout in a desktop browser**

```bash
open trail-finder/index.html
```

Expected: dark screen, "Trail Finder" heading, cyan "Start Recording" button centered. No console errors.

- [ ] **Step 3: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: trail finder HTML shell and CSS"
```

---

## Task 2: JS State, Screen Switching, Button Wiring

**Files:**
- Modify: `trail-finder/index.html`

- [ ] **Step 1: Replace `// JS added in Tasks 2–7` inside `<script>` with the full state block**

```js
// ── Constants ──────────────────────────────────────────────────────
const STEP_LENGTH        = 0.75;  // meters per step
const STEP_HIGH_G        = 1.8;   // acceleration peak to register step (g)
const STEP_LOW_G         = 1.2;   // must fall below this before next step
const STEP_COOLDOWN_MS   = 300;   // minimum ms between steps
const ARRIVAL_DISTANCE_M = 3.0;   // meters — triggers "You're close!"

// ── State ──────────────────────────────────────────────────────────
let appMode  = 'welcome';    // 'welcome' | 'recording' | 'navigating'
let navView  = 'map';        // 'map' | 'finder'
let recordedPath = [];       // [{x, y}] meters from origin
let currentPos   = {x:0, y:0};
let compassHeading = 0;      // degrees, 0=North, clockwise
let stepCount = 0;
let stepAboveThreshold = false;
let lastStepTime = 0;

// ── DOM refs ───────────────────────────────────────────────────────
const mapCanvas = document.getElementById('map-canvas');
const navCanvas = document.getElementById('nav-canvas');
const mapCtx    = mapCanvas.getContext('2d');
const navCtx    = navCanvas.getContext('2d');

// ── Screen / view switching ────────────────────────────────────────
function showScreen(id) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  appMode = id.replace('screen-', '');
  if (id === 'screen-recording') {
    setTimeout(() => resizeCanvas(mapCanvas), 50);
    requestAnimationFrame(drawRecording);
  }
  if (id === 'screen-navigating') {
    setTimeout(() => resizeCanvas(navCanvas), 50);
    requestAnimationFrame(drawNavMap);
  }
}

function showNavView(id) {
  document.querySelectorAll('.nav-view').forEach(v => v.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  navView = id.replace('view-', '');
  if (navView === 'finder') requestAnimationFrame(drawNavFinder);
  if (navView === 'map')    requestAnimationFrame(drawNavMap);
}

// ── Button wiring ──────────────────────────────────────────────────
document.getElementById('btn-start').addEventListener('click', onStartRecording);
document.getElementById('btn-stop').addEventListener('click', onStopRecording);
document.getElementById('btn-to-finder').addEventListener('click', () => showNavView('view-finder'));
document.getElementById('btn-to-map').addEventListener('click',    () => showNavView('view-map'));

// Stubs — implemented in subsequent tasks
function onStartRecording() {}
function onStopRecording()  {}
function resizeCanvas()     {}
function drawRecording()    {}
function drawNavMap()       {}
function drawNavFinder()    {}
```

- [ ] **Step 2: Smoke-test screen switching in DevTools console**

Open `trail-finder/index.html` in desktop browser. In console run:

```js
showScreen('screen-recording');
```

Expected: welcome disappears, recording screen appears (black canvas, REC indicator, disabled Stop button).

```js
showScreen('screen-navigating'); showNavView('view-finder');
```

Expected: finder screen appears with SVG rings + arrow visible.

- [ ] **Step 3: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: state variables, screen switching, button wiring"
```

---

## Task 3: Sensor Permission + Compass Heading

**Files:**
- Modify: `trail-finder/index.html`

- [ ] **Step 1: Replace the stub functions with real implementations**

Replace these four stubs (leave `drawRecording`, `drawNavMap`, `drawNavFinder` stubs for now):

```js
async function onStartRecording() {
  if (typeof DeviceMotionEvent === 'undefined') {
    document.getElementById('sensor-error').style.display = 'block';
    return;
  }
  try {
    if (typeof DeviceMotionEvent.requestPermission === 'function') {
      const mp = await DeviceMotionEvent.requestPermission();
      if (mp !== 'granted') { alert('Motion access required — reload and allow.'); return; }
    }
    if (typeof DeviceOrientationEvent.requestPermission === 'function') {
      const op = await DeviceOrientationEvent.requestPermission();
      if (op !== 'granted') { alert('Orientation access required — reload and allow.'); return; }
    }
  } catch (e) {
    document.getElementById('sensor-error').style.display = 'block';
    return;
  }
  window.addEventListener('deviceorientation', onOrientation, true);
  window.addEventListener('devicemotion',      onMotion,      true);
  resetRecording();
  showScreen('screen-recording');
}

function onStopRecording() {
  currentPos = {x:0, y:0};   // Person B always starts from origin
  showScreen('screen-navigating');
}

function resetRecording() {
  recordedPath = [{x:0, y:0}];
  currentPos   = {x:0, y:0};
  stepCount    = 0;
  stepAboveThreshold = false;
  document.getElementById('step-count').textContent = '0 steps';
  document.getElementById('btn-stop').disabled = true;
}

function onOrientation(e) {
  // webkitCompassHeading: 0=North, increases clockwise (iOS only)
  // alpha fallback: degrees from North, counter-clockwise — invert it
  if (e.webkitCompassHeading != null) {
    compassHeading = e.webkitCompassHeading;
  } else if (e.alpha != null) {
    compassHeading = (360 - e.alpha) % 360;
  }
}

function onMotion(e) { /* Task 4 */ }
```

- [ ] **Step 2: Verify permission on iPad**

Serve from laptop:
```bash
cd trail-finder && python3 -m http.server 8080
ipconfig getifaddr en0    # note the IP address
```

On iPad Safari: `http://<YOUR_IP>:8080`

Tap "Start Recording". Expected: iOS shows permission prompts for motion + orientation. Tap Allow both. Recording screen appears.

- [ ] **Step 3: Verify fallback on desktop**

Open `http://localhost:8080` in Chrome on laptop.  
Expected: "Please open on an iPad and allow motion access." message appears (DeviceMotionEvent exists on Chrome but `requestPermission` is undefined — the `typeof DeviceMotionEvent === 'undefined'` check won't fire, but the sensors will silently not fire, which is acceptable for the demo).

- [ ] **Step 4: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: sensor permission and compass heading"
```

---

## Task 4: Step Detection + Position Math

**Files:**
- Modify: `trail-finder/index.html`

- [ ] **Step 1: Verify position math with a console test before writing production code**

Open DevTools console and paste:

```js
(function testPositionMath() {
  const SL = 0.75;
  function step(pos, headingDeg) {
    const hr = headingDeg * Math.PI / 180;
    return { x: pos.x + SL * Math.sin(hr), y: pos.y + SL * Math.cos(hr) };
  }
  // 10 steps North (heading=0) → x≈0, y≈7.5
  let pos = {x:0, y:0};
  for (let i=0; i<10; i++) pos = step(pos, 0);
  console.assert(Math.abs(pos.x) < 0.001,       'North x should be 0, got '   + pos.x);
  console.assert(Math.abs(pos.y - 7.5) < 0.001, 'North y should be 7.5, got ' + pos.y);
  // 4 steps East (heading=90) → x≈3.0, y≈0
  pos = {x:0, y:0};
  for (let i=0; i<4; i++) pos = step(pos, 90);
  console.assert(Math.abs(pos.x - 3.0) < 0.001, 'East x should be 3.0, got ' + pos.x);
  console.assert(Math.abs(pos.y) < 0.001,        'East y should be 0, got '   + pos.y);
  console.log('✓ Position math tests passed');
})();
```

Expected: `✓ Position math tests passed`

- [ ] **Step 2: Replace `function onMotion(e) { /* Task 4 */ }` with step detection**

```js
function onMotion(e) {
  const a = e.accelerationIncludingGravity;
  if (!a) return;
  const mag = Math.sqrt(a.x * a.x + a.y * a.y + a.z * a.z);

  if (!stepAboveThreshold && mag > STEP_HIGH_G) {
    stepAboveThreshold = true;
  } else if (stepAboveThreshold && mag < STEP_LOW_G) {
    stepAboveThreshold = false;
    const now = Date.now();
    if (now - lastStepTime >= STEP_COOLDOWN_MS) {
      lastStepTime = now;
      recordStep();
    }
  }
}

function recordStep() {
  const hr = compassHeading * Math.PI / 180;
  currentPos = {
    x: currentPos.x + STEP_LENGTH * Math.sin(hr),
    y: currentPos.y + STEP_LENGTH * Math.cos(hr),
  };
  if (appMode === 'recording') {
    recordedPath.push({...currentPos});
    stepCount++;
    document.getElementById('step-count').textContent = stepCount + ' steps';
    if (stepCount >= 5) document.getElementById('btn-stop').disabled = false;
  }
}
```

- [ ] **Step 3: Test step detection on iPad**

Start recording. Walk 20 normal steps in a straight line.

Expected:
- Step counter increments roughly once per step (±3 is fine)
- Counter does NOT increment while standing still
- Stop button becomes enabled after 5 steps

- [ ] **Step 4: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: step detection and dead reckoning position math"
```

---

## Task 5: Canvas Setup + Map Renderer

**Files:**
- Modify: `trail-finder/index.html`

- [ ] **Step 1: Replace `function resizeCanvas() {}` stub and add all canvas helpers**

```js
function resizeCanvas(canvas) {
  canvas.width  = canvas.offsetWidth  * window.devicePixelRatio;
  canvas.height = canvas.offsetHeight * window.devicePixelRatio;
}
window.addEventListener('resize', () => {
  resizeCanvas(mapCanvas);
  resizeCanvas(navCanvas);
});

// Auto-scale path to fill canvas with padding
function buildTransform(canvas, path, padding) {
  padding = padding || 80;
  if (path.length < 2) {
    const scale = Math.min(canvas.width, canvas.height) / 20;
    return { cx: canvas.width / 2, cy: canvas.height / 2, scale };
  }
  const xs = path.map(p => p.x), ys = path.map(p => p.y);
  const minX = Math.min.apply(null, xs), maxX = Math.max.apply(null, xs);
  const minY = Math.min.apply(null, ys), maxY = Math.max.apply(null, ys);
  const rangeX = maxX - minX || 1, rangeY = maxY - minY || 1;
  const scale = Math.min(
    (canvas.width  - padding * 2) / rangeX,
    (canvas.height - padding * 2) / rangeY
  );
  return {
    cx: canvas.width  / 2 - ((minX + maxX) / 2) * scale,
    cy: canvas.height / 2 + ((minY + maxY) / 2) * scale,
    scale,
  };
}

// World coords (meters) → canvas pixels. Y-axis: canvas down = world south.
function worldToCanvas(t, p) {
  return { cx: t.cx + p.x * t.scale, cy: t.cy - p.y * t.scale };
}

function drawGrid(ctx, canvas, t) {
  const step = Math.max(t.scale * 5, 30);
  ctx.strokeStyle = '#1a1a2e';
  ctx.lineWidth = 1;
  for (let x = ((t.cx % step) + step) % step; x < canvas.width;  x += step) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, canvas.height); ctx.stroke();
  }
  for (let y = ((t.cy % step) + step) % step; y < canvas.height; y += step) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
  }
}

function drawGlowingPath(ctx, t, path, alpha) {
  alpha = alpha == null ? 1.0 : alpha;
  if (path.length < 2) return;
  const pts = path.map(function(p) { return worldToCanvas(t, p); });
  // Glow pass
  ctx.save();
  ctx.globalAlpha = 0.25 * alpha;
  ctx.strokeStyle = '#00e5ff'; ctx.lineWidth = 10;
  ctx.lineCap = 'round'; ctx.lineJoin = 'round';
  ctx.beginPath(); ctx.moveTo(pts[0].cx, pts[0].cy);
  pts.slice(1).forEach(function(p) { ctx.lineTo(p.cx, p.cy); });
  ctx.stroke();
  // Core pass
  ctx.globalAlpha = alpha;
  ctx.lineWidth = 2.5;
  ctx.beginPath(); ctx.moveTo(pts[0].cx, pts[0].cy);
  pts.slice(1).forEach(function(p) { ctx.lineTo(p.cx, p.cy); });
  ctx.stroke();
  ctx.restore();
}

function drawDot(ctx, t, pos, color, r) {
  r = r || 8;
  const pt = worldToCanvas(t, pos);
  ctx.beginPath(); ctx.arc(pt.cx, pt.cy, r, 0, Math.PI * 2);
  ctx.fillStyle = color; ctx.fill();
}

function drawCompassRose(ctx) {
  var x = 44, y = 44, r = 22;
  ctx.save(); ctx.globalAlpha = 0.65;
  ctx.strokeStyle = '#00e5ff'; ctx.lineWidth = 1;
  ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI * 2); ctx.stroke();
  ctx.fillStyle = '#00e5ff';
  ctx.font = 'bold ' + Math.round(r * 0.6) + 'px -apple-system';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText('N', x, y - r * 0.45);
  ctx.strokeStyle = '#ff5252'; ctx.lineWidth = 2.5;
  ctx.beginPath(); ctx.moveTo(x, y - r + 5); ctx.lineTo(x, y); ctx.stroke();
  ctx.restore();
}
```

- [ ] **Step 2: Replace `function drawRecording() {}` stub with the real loop**

```js
function drawRecording() {
  if (appMode !== 'recording') return;
  mapCtx.clearRect(0, 0, mapCanvas.width, mapCanvas.height);
  var path = recordedPath.concat([currentPos]);
  var t = buildTransform(mapCanvas, path);
  drawGrid(mapCtx, mapCanvas, t);
  drawGlowingPath(mapCtx, t, path);
  drawDot(mapCtx, t, {x:0, y:0}, '#69f0ae', 8);   // start — green
  drawDot(mapCtx, t, currentPos,  '#ffffff', 6);   // current — white
  drawCompassRose(mapCtx);
  requestAnimationFrame(drawRecording);
}
```

- [ ] **Step 3: Test recording draw on iPad**

Start recording. Walk an L-shape: ~15 steps forward, turn 90°, ~10 steps.

Expected:
- Canvas shows a glowing cyan path tracing movement
- Green dot stays at center (start)
- White dot moves along the leading edge of the path
- Compass rose (N) visible in top-left

- [ ] **Step 4: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: canvas renderer, glowing path, dots, compass rose"
```

---

## Task 6: Navigation Map View + Finder Arrow

**Files:**
- Modify: `trail-finder/index.html`

- [ ] **Step 1: Verify bearing math with a console test**

```js
(function testBearingMath() {
  function getBearingDeg(cur, dest, headingDeg) {
    var dx = dest.x - cur.x, dy = dest.y - cur.y;
    var deg = Math.atan2(dx, dy) * 180 / Math.PI;
    return ((deg - headingDeg) + 360) % 360;
  }
  // Dest due North, facing North → arrow points 0° (straight up)
  var r1 = getBearingDeg({x:0,y:0}, {x:0,y:10}, 0);
  console.assert(Math.abs(r1) < 1, 'Should be 0°, got ' + r1);
  // Dest due East, facing North → arrow points 90° (right)
  var r2 = getBearingDeg({x:0,y:0}, {x:10,y:0}, 0);
  console.assert(Math.abs(r2 - 90) < 1, 'Should be 90°, got ' + r2);
  // Dest due North, facing East → arrow points 270° (left, toward North from East)
  var r3 = getBearingDeg({x:0,y:0}, {x:0,y:10}, 90);
  console.assert(Math.abs(r3 - 270) < 1, 'Should be 270°, got ' + r3);
  console.log('✓ Bearing math tests passed');
})();
```

Expected: `✓ Bearing math tests passed`

- [ ] **Step 2: Add nav UI helpers after `drawCompassRose`**

```js
function getDist(a, b) {
  var dx = b.x - a.x, dy = b.y - a.y;
  return Math.sqrt(dx * dx + dy * dy);
}

function getBearingDeg(cur, dest, headingDeg) {
  var dx = dest.x - cur.x, dy = dest.y - cur.y;
  var deg = Math.atan2(dx, dy) * 180 / Math.PI;
  return ((deg - headingDeg) + 360) % 360;
}

function getDestination() {
  return recordedPath[recordedPath.length - 1] || {x:0, y:0};
}

function updateNavUI() {
  var dest = getDestination();
  var dist = getDist(currentPos, dest);
  var distStr = Math.round(dist) + ' m';

  document.getElementById('distance-display').textContent = distStr + ' to destination';
  document.getElementById('finder-distance').textContent  = Math.round(dist) + ' m';

  var bearing = getBearingDeg(currentPos, dest, compassHeading);
  document.getElementById('finder-arrow').setAttribute(
    'transform', 'rotate(' + bearing + ', 130, 130)'
  );

  var near = dist <= ARRIVAL_DISTANCE_M;
  document.getElementById('arrival-msg').style.display = near ? 'block' : 'none';
  document.getElementById('finder-rings').classList.toggle('near', near);
}
```

- [ ] **Step 3: Replace `function drawNavMap() {}` and `function drawNavFinder() {}` stubs**

```js
function drawNavMap() {
  if (appMode !== 'navigating' || navView !== 'map') return;
  navCtx.clearRect(0, 0, navCanvas.width, navCanvas.height);
  var dest    = getDestination();
  var allPts  = recordedPath.concat([currentPos]);
  var t = buildTransform(navCanvas, allPts);
  drawGrid(navCtx, navCanvas, t);
  drawGlowingPath(navCtx, t, recordedPath, 0.5);           // recorded path, faded
  drawDot(navCtx, t, {x:0, y:0}, '#69f0ae', 8);            // start — green
  drawDot(navCtx, t, dest,        '#ff5252', 8);            // destination — red
  drawDot(navCtx, t, currentPos,  '#ffffff', 6);            // person B — white
  drawCompassRose(navCtx);
  updateNavUI();
  requestAnimationFrame(drawNavMap);
}

function drawNavFinder() {
  if (appMode !== 'navigating' || navView !== 'finder') return;
  updateNavUI();
  requestAnimationFrame(drawNavFinder);
}
```

- [ ] **Step 4: Test full demo flow on iPad**

1. Start recording → walk 20+ steps in an L-shape → Stop  
2. Map view: verify green start dot, red destination dot, white dot at start, distance shown  
3. Walk away from start — white dot should move, distance should change  
4. Tap FINDER — arrow SVG visible, distance shown in large text  
5. Turn in a circle — arrow rotates to keep pointing at destination  
6. Tap MAP — returns to map view  

- [ ] **Step 5: Commit**

```bash
git add trail-finder/index.html
git commit -m "feat: nav map view, finder arrow, distance display"
```

---

## Task 7: End-to-End Verification + Arrival Polish

**Files:**
- Modify: `trail-finder/index.html` (arrival UX only — no structural changes)

- [ ] **Step 1: Confirm arrival pulse animation works**

In DevTools console while on the Finder screen, simulate arrival:

```js
// Force arrival state
ARRIVAL_DISTANCE_M_TEST = 999;
document.getElementById('arrival-msg').style.display = 'block';
document.getElementById('finder-rings').classList.add('near');
```

Expected: rings pulse (opacity animation), "You're close!" text appears in green.

Remove the test:

```js
document.getElementById('arrival-msg').style.display = 'none';
document.getElementById('finder-rings').classList.remove('near');
```

- [ ] **Step 2: Full end-to-end walkthrough on iPad**

Run this scenario exactly as the demo would be presented:

1. Person A: tap "Start Recording", allow permissions  
2. Person A: walk from one end of the room to another (~20 steps), turn, walk to a third point  
3. Person A: tap "Stop Recording"  
4. Hand iPad to Person B at the starting location  
5. Person B: observe map shows the recorded path in cyan, their position (white dot) at the green start  
6. Person B: tap FINDER — large arrow pointing toward destination, distance in meters  
7. Person B: walk toward destination — arrow stays oriented toward dest, distance decreases  
8. Person B: arrive within ~3m of destination — rings pulse, "You're close!" appears  

- [ ] **Step 3: Verify desktop fallback**

Visit `http://localhost:8080` in Chrome on laptop.  
Expected: welcome screen, no crash, no console errors. If "Start Recording" is tapped, sensor-error message appears (or the sensors silently produce no data — either is acceptable).

- [ ] **Step 4: Final commit**

```bash
git add trail-finder/index.html
git commit -m "feat: trail finder demo complete"
```

---

## Deployment Reference

```bash
# On laptop — from the trail-finder/ directory
python3 -m http.server 8080

# Find your laptop's WiFi IP
ipconfig getifaddr en0

# On iPad — open Safari and visit:
http://<YOUR_LAPTOP_IP>:8080
```

Both devices must be on the same WiFi network. The permission prompt only appears on the first tap of "Start Recording" — once granted, it persists for the session.

---

## Known Limitations (acceptable for demo)

- Dead reckoning drift accumulates over long distances (fine for house-scale)
- Step detection may double-count on fast walking or stairs — adjust `STEP_HIGH_G` threshold if needed
- `webkitCompassHeading` is iOS Safari only; Chrome on Android will use the `alpha` fallback (less accurate)
- No persistence — path is lost on page reload
