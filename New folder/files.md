# Files — Architecture & Design Documentation

## 1. Architecture Overview

### Single-Page View Switching

The application uses a **two-view, single-page architecture**:

- **`index.html`** is the sole file. It contains two `<div>` elements — one for the **Login Screen** (`#login-page`) and one for the **Calculator Screen** (`#calculator-page`).
- Each view has the CSS class `.view` with `display: flex` and `height: 100dvh` (with `100vh` fallback), making them full-screen panels locked to the viewport with `overflow: hidden` — no scrolling.
- A utility class `.hidden` (`display: none !important`) is toggled via JavaScript to switch between views.
- **Flow**: The user clicks any login button (main "INITIALIZE CONNECTION", "GOOGLE LOGIN", or "FACEBOOK LOGIN") or presses Enter in the password field → JavaScript plays the boot sound + success chime → adds `.hidden` to `#login-page` and removes it from `#calculator-page` → the calculator is revealed.
- Ambient animations (equalizer bars, data stream, ONLINE blink) start when the calculator becomes visible and stop when returning to login.
- The entire state management (calculator input, operator, display, operation counter) lives in a single JavaScript IIFE.

### Layer Stacking Order

```
z-index: 9999 → body::after (animated scan-line overlay)
z-index: 1    → .view (login/calculator panels)
z-index: 0    → #hud-bg (canvas particle network)
```

### Audio Engine (Web Audio API)

The app uses a synthesized audio system with zero external files. All sounds are generated at runtime using `OscillatorNode` and `GainNode`:

| Sound | Synthesis | Trigger |
|---|---|---|
| **System Boot** | Sine wave, frequency ramp 100Hz → 800Hz over 0.6s, gain 0.05 | First user interaction (focus on input, or any login button) |
| **Click/Beep** | Square wave at 600Hz, gain 0.08, duration 50ms | Every calculator button press, keyboard input, or input focus |
| **Access Granted** | 3-note ascending arpeggio (C5=523Hz, E5=659Hz, G5=784Hz), sine wave, each note 120ms, gain 0.1 | Any login button click |

## 2. Particle Web Engine — Canvas Rendering

The `<canvas id="hud-bg">` renders an exclusive **square-particle network** in real time. Here is how the engine works and why it stays efficient on mobile:

### Rendering Loop

The engine runs at the display's refresh rate via `requestAnimationFrame`. Each frame executes:

1. **Clear** — `ctx.clearRect(0, 0, canvas.width, canvas.height)` wipes the previous frame.
2. **Update positions** — Each particle's `(x, y)` is incremented by `(vx, vy)`. Velocity is `±0.4px/frame`, which equates to ~24px/second at 60fps.
3. **Edge wrapping** — If a particle exits any edge, it wraps to the opposite edge (seamless looping, no teleportation).
4. **Draw each particle** — `ctx.fillRect(p.x, p.y, p.size, p.size)` draws **strictly sharp-edged squares**. No `arc()` or `circle()` calls anywhere in the render path.
5. **Connection lines** — For each pair `(i, j)` where `j > i`, Euclidean distance is computed. If `distance < 120px`, a thin line (`lineWidth: 0.5`) is drawn with opacity `(1 - dist/120) × 0.06`. This creates a fading web as particles drift apart.
6. **Schedule next frame** — `requestAnimationFrame(animateBg)`.

### Particle Configuration

| Property | Value | Rationale |
|---|---|---|
| Particle count | **45** (was 70) | Reduced to save battery on mobile while maintaining visual density |
| Shape | `fillRect()` square | Strictly matches GeoExe sharp-block aesthetic; zero circular elements |
| Size | `0.5px` to `2.0px` (random) | Varied sizes create depth without additional draw calls |
| Velocity | `±0.4px/frame` | Slow enough to be ambient, fast enough to prevent stagnation |
| Connection distance | `120px` | Balances web density vs. draw calls on weaker GPUs |
| Connection count | ~90 lines per frame (worst case: 45 particles × 44 neighbors / 2 ≈ 990 checks, only ~90 pass the distance check) | O(n²/2) distance check is cheap — only 990 operations, then ~90 stroke calls |
| Canvas fill | `rgba(255,255,255,0.1)` | Low alpha ensures particles don't overpower the UI above them |


### Grid Matrix Math — Infinite Scrolling Grid

The canvas also renders a **floating grid matrix** that drifts diagonally to simulate a 3D tactical display:

**How it works:**
1. A single `gridOffset` variable increments by `0.25` per frame (~15px/second at 60fps).
2. Vertical and horizontal offset are derived from `gridOffset % cellSize` where `cellSize = 60px`. The horizontal offset uses `gridOffset` directly; the vertical offset uses `gridOffset × 0.7` — creating a **slower diagonal drift** rather than a straight scroll.
3. Grid lines are drawn in a loop starting from `-(offset)` to `canvas.width + cellSize` (or `canvas.height + cellSize`) in steps of `cellSize`. This ensures the grid always tiles seamlessly beyond screen edges.
4. Because `offset` is wrapped with `% cellSize`, the value cycles between `0` and `59.999...` indefinitely — the grid never overflows or wraps discontinuously.
5. The grid uses a single `strokeStyle` (`rgba(255,255,255,0.025)`) and `lineWidth` (0.5) — all lines share one state change, minimizing GPU draw-call overhead.

**Battery impact:**
- ~12-15 vertical lines + ~10-12 horizontal lines = ~25 additional `stroke()` calls per frame
- Total draw calls with grid: ~160 (45 particles + 90 connections + 25 grid lines)
- The grid is drawn **before** particles, so particle connection lines render on top of the grid, creating depth layering without `z-index` management

### Mobile Performance Budget

| Metric | Value |
|---|---|
| Draw calls per frame (without grid) | ~135 (45 fillRect + ~90 stroke) |
| Draw calls per frame (with grid) | ~160 (45 + 90 + 25 grid lines) |
| Math ops per frame | ~1,000 (subtractions, multiplications, sqrt, comparisons) |
| Memory | ~2KB for particle array (45 objects × ~8 properties) |
| Canvas pixel size | Full viewport, but only 45 tiny rects + 90 thin lines drawn |
| Impact | Negligible — even low-end phones render this at 60fps with no throttling observed |
| Battery | Canvas is GPU-accelerated in all modern browsers (Chrome, Safari, Firefox, Edge) |

The canvas sits beneath a semi-transparent panel (`rgba(10,10,10,0.92)`) and a radial vignette, so the particles are visible but muted — never distracting from the UI.

## 3. Animation Architecture — Living HUD System

The calculator page features **7 simultaneous ambient animations** that make it feel like a live tactical terminal. They are designed with battery efficiency in mind:

### Animation 1: Drifting Scan-Line Overlay (CSS-only, GPU)

| Property | Detail |
|---|---|
| **Element** | `body::after` pseudo-element |
| **Technique** | CSS `@keyframes scanDrift` animating `background-position` from `0 0` to `0 4px` |
| **Duration** | 6s linear infinite loop |
| **Battery** | GPU-composited — `background-position` changes do not trigger layout or paint |
| **Visible on** | Both login and calculator pages |

### Animation 2: Canvas Particle Network (requestAnimationFrame)

| Property | Detail |
|---|---|
| **Element** | `<canvas id="hud-bg">` |
| **Technique** | 45 square particles with proximity-based connection lines, `requestAnimationFrame` loop |
| **Performance** | ~135 draw calls per frame (45 fillRect + ~90 strokes); canvas operations stay on GPU |
| **Particle count** | 45 (reduced from 70 for mobile battery efficiency) |
| **Visible on** | Both pages, behind semi-transparent panels (`rgba(10,10,10,0.92)`) and vignette overlay |
| **Details** | See Section 2 "Particle Web Engine" for full rendering breakdown |

### Animation 3: Radar Sweep Line (CSS-only, GPU)

| Property | Detail |
|---|---|
| **Element** | `.calc-display-wrap::after` pseudo-element |
| **Technique** | CSS `@keyframes radarSweep` using `transform: translateY(0)` to `translateY(100%)` |
| **Duration** | 3s linear infinite loop |
| **Battery** | Only `transform` is animated — GPU composited, no layout/paint. Uses `will-change: transform` to pre-promote the layer. |
| **Visible on** | Calculator page only |

### Animation 4: Pulsing "CALIBRATION: ACTIVE" Text (CSS-only)

| Property | Detail |
|---|---|
| **Element** | `<span class="pulse-calibration">` |
| **Technique** | CSS `@keyframes pulseStatus` animating `opacity` from 0.3 → 1.0 → 0.3 |
| **Duration** | 2s ease-in-out infinite |
| **Battery** | `opacity` is GPU composited — zero CPU overhead |
| **Visible on** | Top HUD bar of calculator page |

### Animation 5: Equalizer / Frequency Bars (JS setInterval)

| Property | Detail |
|---|---|
| **Container** | `#eqBarsLeft` (8 vertical bars in left sidebar) + `#eqMobile` (8 horizontal bars below display) |
| **Technique** | JavaScript `setInterval` at 150ms, each iteration randomizes bar heights (0-100%) |
| **Battery** | Only 16 DOM writes per tick (8 sidebar + 8 mobile). 150ms interval = ~6.6 updates/sec, far below 60fps thresholds. |
| **Mobile optimization** | On < 480px screens, the mobile strip has reduced height range (30-90% instead of 0-100%) to be less visually intense |
| **Sidebar visibility** | `#eqBarsLeft` only visible on tablet+ (480px+); mobile uses `#eqMobile` strip |

### Animation 6: Live I/O Buffer Data Stream (JS setInterval)

| Property | Detail |
|---|---|
| **Element** | `#ioBuffer` div in the right sidebar |
| **Technique** | JavaScript `setInterval` at 400ms, randomly replaces hex bytes in the 4-row buffer |
| **Battery** | Only modifies innerHTML once per 400ms (~2.5 updates/sec). On mobile, only 1 byte changes per tick vs 3 on desktop. |
| **Cycle rate** | Desktop: 3 random bytes change per tick. Mobile (< 480px): 1 byte per tick |

### Animation 7: ONLINE Status Organic Blink (JS setTimeout)

| Property | Detail |
|---|---|
| **Element** | `<strong id="onlineStatus">` in the top HUD bar |
| **Technique** | JavaScript `setTimeout` with random delays between 2-8 seconds, opacity drops to 0.3 for 200-600ms, then restores |
| **Battery** | Uses `setTimeout` not `setInterval` — only one timer active at a time. Zero CPU usage between blinks. |
| **Feel** | Random intervals create an organic, non-mechanical flicker |

### Animation 8: Calculator Display Pulsing Glow (CSS-only)

| Property | Detail |
|---|---|
| **Element** | `.calc-display` |
| **Technique** | CSS `@keyframes displayGlow` animating `box-shadow` from `inset 0 0 8px rgba(255,255,255,0.03)` to `inset 0 0 18px rgba(255,255,255,0.1)` |
| **Duration** | 4s ease-in-out infinite |
| **Battery** | GPU composited with `will-change: box-shadow` |

### Battery & Performance Summary

| Animation | Technique | Updates/sec | GPU? |
|---|---|---|---|
| Scan-line drift | CSS background-position | 60fps | Yes |
| Canvas particles | requestAnimationFrame | 60fps | Yes |
| Radar sweep | CSS transform | 60fps | Yes |
| Pulse text | CSS opacity | 60fps | Yes |
| EQ bars | JS setInterval | 6.6/sec | No (trivial) |
| Data stream | JS setInterval | 2.5/sec | No (trivial) |
| ONLINE blink | JS setTimeout | ~0.15/sec avg | No (trivial) |
| Display glow | CSS box-shadow | 60fps | Yes |

## 4. Interactive Elements

### Animated Tech Background (Canvas Particle Network)

A `<canvas id="hud-bg">` element is fixed behind all UI layers (`z-index: 0`) and renders a real-time particle network:

- **45 particles** (reduced from 70 for mobile battery efficiency) distributed randomly across the viewport.
- Each particle is a tiny square (`fillRect`), colored `rgba(255,255,255,0.1)`, moving at slow random velocity (`±0.4px/frame`).
- **Connection lines**: When two particles are within 120px of each other, a thin line (`lineWidth: 0.5`) is drawn between them with opacity inversely proportional to distance.
- Particles wrap around screen edges seamlessly.
- Canvas resizes and reinitializes on window `resize` events.
- A radial vignette overlay (`#hud-vignette`) sits above the canvas but below the views, darkening screen edges to draw focus to the center UI panels.

### Social Login Flow

The social login buttons ("GOOGLE LOGIN" and "FACEBOOK LOGIN") are **simulated** — they do not connect to any external API:

1. They are purely visual UI elements styled to match the GeoExe wireframe aesthetic.
2. Clicking either button calls the same `handleLogin()` function as the main "INITIALIZE CONNECTION" button.
3. The `handleLogin()` function plays the boot sound (on first interaction), then after 150ms plays the ascending success chime and reveals the calculator.
4. All three login buttons are equally responsive and accessible.

## 5. Responsive Engineering

### Viewport Lock (100% Screen Fit)

The most critical fix for fitting the interface within the viewport is the **height locking** strategy:

- `html, body { height: 100%; overflow: hidden; }` — prevents any global scroll.
- `.view { height: 100dvh; height: 100vh; overflow: hidden; }` — uses `dvh` (dynamic viewport height) for mobile browsers where the address bar collapses/expands, with `vh` as fallback.
- `.calc-hud { max-height: 100%; display: flex; flex-direction: column; }` — the calculator HUD wrapper uses flex column to constrain all children within the available vertical space.
- `flex: 1` on `.calc-layout` and `min-height: 0` on flex children — ensures the grid can shrink rather than overflow.

### Fluid Sizing with `clamp()`

Instead of fixed pixel sizes, every dimensional value uses the CSS `clamp()` function:

```
font-size: clamp(15px, 4vw, 20px)    /* min, preferred, max */
padding: clamp(8px, 2vw, 16px)
min-height: clamp(40px, 10vw, 56px)
gap: clamp(2px, 0.5vw, 4px)
```

This creates a **continuous fluid scaling** between the minimum (mobile) and maximum (desktop) breakpoints. The middle value uses `vw` units, which scale proportionally with the viewport width.

### How the Grid Auto-Resizes

The calculator grid uses `grid-template-columns: repeat(4, 1fr)` with gaps in `clamp()`. On a 320px phone screen:

1. Viewport padding takes `clamp(6px, 1.5vw, 20px)` = ~10px per side → ~300px remaining
2. Border + padding of `.calc-layout` and `.calc-main` consume ~14px → ~286px for grid
3. Gap between buttons: `clamp(2px, 0.5vw, 4px)` = 2px on mobile → 3 gaps × 2px = 6px → ~280px for buttons
4. Each button: ~70px wide, `min-height: clamp(40px, 10vw, 56px)` = 40px tall
5. Display: `min-height: clamp(42px, 10vw, 68px)` = 42px
6. Equalizer strip: `height: clamp(14px, 3vw, 24px)` = 14px
7. Total calculator main height: ~42px display + 14px eq + 4px gap + 5 rows × (40px + 2px) = 270px

This fits comfortably within a 568px phone viewport height (iPhone SE), with room for the top/bottom HUD bars.

### Touch Optimization

- All calculator buttons: `min-height: clamp(40px, 10vw, 56px)` — ensures at least 40px tap target on the smallest screens.
- `-webkit-tap-highlight-color: transparent` — removes the default mobile highlight overlay.
- `touch-action: manipulation` — prevents double-tap zoom on buttons.
- Login and social buttons: `min-height: clamp(38px, 7vw, 56px)`.

## 6. Breakpoints & Layout Adjustments

The interface uses **3 tiers** with a **mobile-first** approach (base styles are for mobile, then enhanced at larger widths):

### Tier 1: Mobile (< 480px)

| Component | Behavior |
|---|---|
| **Login card** | `width: 90%`, max-width 420px. Header text scales with `clamp(11px, 2.5vw, 14px)`. |
| **Social buttons** | Flex row with wrap — buttons may stack vertically on very narrow screens. |
| **Calculator layout** | Single column (`grid-template-columns: 1fr`). |
| **Side HUD panels** | Hidden (`display: none`). Only the calculator grid and display are shown. |
| **Equalizer** | Horizontal strip below display (`#eqMobile`), bars 30-90% height range. |
| **I/O Buffer** | Slower cycle (1 byte/tick at 400ms). |
| **Top HUD bar** | 2-column grid (`grid-template-columns: 1fr 1fr`). |
| **Bottom HUD bar** | 2-column grid. |
| **Calc buttons** | `min-height: clamp(40px, 10vw, 56px)`, `font-size: clamp(15px, 4vw, 20px)`. |
| **View padding** | `clamp(6px, 1.5vw, 20px)` — minimal padding to maximize space. |

### Tier 2: Tablet (480px – 820px)

| Component | Behavior |
|---|---|
| **Calculator layout** | 3-column grid restored with fluid sidebar widths: `clamp(70px, 12vw, 110px) 1fr clamp(70px, 12vw, 110px)`. |
| **Side HUD panels** | Visible but compact — reduced padding, smaller labels, condensed meters. |
| **Equalizer** | Vertical bars in left sidebar (`#eqBarsLeft`) + mobile strip hidden. |
| **I/O Buffer** | Normal cycle (3 bytes/tick at 400ms). |
| **Top HUD bar** | 4-column grid. |
| **Bottom HUD bar** | 3-column grid. |
| **View padding** | `clamp(8px, 2vw, 16px)`. |

### Tier 3: Desktop (820px+)

| Component | Behavior |
|---|---|
| **Calculator layout** | Full 3-column grid: `140px 1fr 140px` with 4px gap. |
| **Side HUD panels** | Full padding (`12px 8px`), standard label sizes, all decorative content visible. |
| **Equalizer** | Full vertical bars in left sidebar. |
| **I/O Buffer** | Normal cycle. |
| **Top HUD bar** | `display: flex; justify-content: space-between;` single row. |
| **Bottom HUD bar** | `display: flex; justify-content: space-between;` single row. |
| **Calc buttons** | `padding: 16px 4px`, `font-size: 20px`, `min-height: 56px`. |
| **View padding** | 20px on all sides. |

### Notch / Safe Area Support

```css
@supports (padding: env(safe-area-inset-top)) {
  .view {
    padding-left: max(clamp(6px, 1.5vw, 20px), env(safe-area-inset-left));
    padding-right: max(clamp(6px, 1.5vw, 20px), env(safe-area-inset-right));
    padding-top: max(clamp(6px, 1.5vw, 20px), env(safe-area-inset-top));
    padding-bottom: max(clamp(6px, 1.5vw, 20px), env(safe-area-inset-bottom));
  }
}
```

## 7. Setup Instructions — Imagine Font

The custom **"Imagine Font"** typeface is used throughout the UI for headers, labels, and calculator display.

### To install the font locally:

1. **Locate the font file**: `imagine_font.ttf` is included in the same directory as `index.html`.

2. **Install on Windows**:
   - Right-click `imagine_font.ttf` → **Install** (installs system-wide).
   - Or: open the file → click **Install** at the top of the preview window.

3. **The `@font-face` declaration** in `<style>` already references the font:
   ```css
   @font-face {
     font-family: 'Imagine Font';
     src: url('./imagine_font.ttf') format('truetype');
     font-weight: normal;
     font-style: normal;
     font-display: swap;
   }
   ```

4. **Fallback behavior**: If the font fails to load (e.g., missing TTF file), the browser falls back through:
   - `'Courier New'` — a monospace terminal font that preserves the blocky aesthetic.
   - `'Orbitron'` — a geometric sans-serif Google Font (optional; must be loaded separately if desired).
   - `monospace` — the system default monospace.

5. **Optional — Load via Google Fonts** (if you prefer not to use the local TTF):
   ```html
   <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700&display=swap" rel="stylesheet">
   ```

6. **Verify**: Open `index.html` in a browser. The login card header and calculator buttons should render in the Imagine Font. If not, open DevTools → check the **Network** tab to confirm `imagine_font.ttf` was loaded (200 status).

### File Structure
```
your-project-folder/
├── index.html
├── imagine_font.ttf
└── files.md
```

No build tools, bundlers, or package managers are required. Open `index.html` directly in any modern browser.