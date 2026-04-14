# iPad Trail Finder — Design Spec
**Date:** 2026-04-14  
**Status:** Approved

## Overview

A single-file web app (HTML/CSS/vanilla JS) that runs in Safari on an iPad. Person A walks through a space while the app records their path using dead reckoning (accelerometer step detection + compass heading). They hand the iPad to Person B at the starting location. Person B uses the app to navigate to where Person A went, guided by a map view and an AirTag-style finder view.

Accuracy is intentionally approximate — the goal is directional guidance, not precision positioning. This is a concept demo; the code is not intended for production use.

---

## Use Case

1. **Person A** opens the app at Location A, presses Start Recording, and walks to Location B.
2. Person A presses Stop and hands the iPad back to someone at Location A.
3. **Person B** (at Location A) presses Navigate and follows the recorded trail to find Location B.

---

## Screens

### 1. Welcome Screen
- App name ("Trail Finder"), one-line description
- "Start Recording" button
- On tap: calls `DeviceMotionEvent.requestPermission()` and `DeviceOrientationEvent.requestPermission()` (iOS 13+ requirement)
- If permission denied or sensors unavailable: show "Please open on an iPad and allow motion access"

### 2. Recording Screen
- Dark canvas with subtle grid
- Glowing cyan path drawn in real-time as Person A walks
- Green dot at start position (center of canvas on first launch)
- White dot at current position with a direction indicator
- Compass rose (N marker) in top-left corner
- Step counter ("47 steps") in top-right
- Red "Stop Recording" button at bottom
- On stop: transitions to Navigate mode (no separate review screen needed for demo simplicity)

### 3. Map View (Navigation)
- Same dark canvas, full recorded path shown in faded cyan
- Green dot = start, Red dot = destination
- White dot = Person B's current estimated position (tracked via same dead reckoning from start)
- Distance to destination shown in header ("18 m to destination")
- "FINDER" toggle button switches to Screen 4

### 4. Finder View (Navigation)
- AirTag-style UI: three concentric rings on dark background
- Large arrow inside the rings pointing toward the destination
- Distance readout in meters below the rings (large text)
- Arrow updates continuously as Person B moves and turns
- "MAP" toggle button switches back to Screen 3
- When within ~3m of destination: rings pulse and show "You're close!"

---

## Technical Architecture

### Single HTML File
Self-contained — no npm, no build step, no external dependencies. Served via `python3 -m http.server 8080` from any directory.

### Sensor Loop
```
DeviceMotionEvent  →  step detector  →  path array [{x, y}]
DeviceOrientationEvent  →  compass heading  →  heading used by step detector + arrow direction
```

Both events fire continuously. The same sensor loop runs in both Record and Navigate modes.

### Step Detection
- Monitor `event.accelerationIncludingGravity` magnitude: `sqrt(x² + y² + z²)`
- A step fires when magnitude exceeds threshold (~1.8g) and then falls back below (~1.2g)
- Each step advances position by `stepLength` (default 0.75m) in the current compass heading
- Debounce: minimum 300ms between steps to avoid double-counting

### Compass Heading
- iOS: `event.webkitCompassHeading` from `DeviceOrientationEvent` (0° = North, clockwise)
- Convert to radians for trigonometry: `heading_rad = heading_deg * Math.PI / 180`
- Position update per step:
  ```
  x += stepLength * sin(heading_rad)
  y += stepLength * cos(heading_rad)  // y increases northward
  ```

### Path Data Structure
```js
// Stored in memory (no persistence needed for demo)
recordedPath = [{x: 0, y: 0}, {x: 0.5, y: 0.3}, ...]  // meters from origin
currentPos = {x, y}   // updated live during navigation
```

### Canvas Renderer
- Origin (start) mapped to canvas center
- Scale: auto-fit path to canvas with padding
- Glowing trail: draw path twice — wide stroke at low opacity (glow), narrow stroke at full opacity (core)
- Redraws on each animation frame (`requestAnimationFrame`)

### Finder Arrow
- Angle from currentPos to destination:
  ```js
  // x = east/west, y = north/south — atan2(dx, dy) gives bearing from north
  angle = atan2(dest.x - cur.x, dest.y - cur.y)  // bearing in radians from north
  arrowRotation = angle - compassHeading_rad       // relative to device orientation
  ```
- Applied as CSS `transform: rotate(Xdeg)` on the arrow SVG

### Mode State
A single variable `appMode` ∈ `{welcome, recording, navigating}` drives which UI is shown. The sensor loop is always running once permission is granted.

---

## Deployment

```bash
# On laptop — in the directory containing index.html
python3 -m http.server 8080

# Find laptop's local IP
ipconfig getifaddr en0   # macOS WiFi

# On iPad — open Safari and visit:
http://192.168.x.x:8080
```

Both devices must be on the same WiFi network.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Sensors unavailable (desktop browser) | Welcome screen shows "Please open on an iPad" message |
| Permission denied | Alert: "Motion access is required — please reload and allow" |
| `webkitCompassHeading` unavailable | Fall back to `alpha` from DeviceOrientation (less accurate) |
| Path has zero steps | Navigate button disabled until at least 5 steps recorded |

---

## Out of Scope (Demo Only)
- Persisting the path across page reloads
- Multiple saved paths
- Floor plan image overlay
- Bluetooth/WiFi-based positioning
- Any backend or server-side logic
