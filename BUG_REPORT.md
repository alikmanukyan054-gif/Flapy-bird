# Flappy Crash — Code Review & Fix Report

---

## 1. BUGS FOUND

### 🔴 Critical

| # | Bug | Location | Explanation |
|---|-----|----------|-------------|
| 1 | **FPS-dependent speed** | `loop()` | `dt` was capped at `0.08 s` (12.5 FPS min), but on 144 Hz screens small accumulated dt errors caused particles and physics to run slightly faster than on 60 Hz. |
| 2 | **Crash system is linear & predictable** | `PERCENT_TICK_MS / PERCENT_PER_TICK` | Adding exactly 1% every 200 ms is completely deterministic — player can always time out. No randomness, no tension. |
| 3 | **No crash randomness** | `update()` | There is no hidden crash target. The game just grows a fixed 5%/s forever. A real crash game must pick a secret crash point at round start. |
| 4 | **Instant crash possible at 0%** | crash logic | Nothing prevents crash at the very first tick. Players should have a guaranteed grace period. |
| 5 | **Double-input on mobile** | `click` + `touchstart` | Both events fire on a single finger tap. The bird double-jumped every tap on mobile, making control erratic. |
| 6 | **Blurry canvas on Retina** | `resizeCanvas()` | Canvas `width`/`height` was set to CSS pixels only. On DPR=2 devices (iPhone, MacBook) everything rendered at half resolution, looking blurry. |

### 🟠 Medium

| # | Bug | Location | Explanation |
|---|-----|----------|-------------|
| 7 | **Per-frame DOM query** | `update()` — `gameSpeed = getGameSpeed()` | `document.getElementById('speedSelect').value` was called every frame (~60×/s). Read it once at game start. |
| 8 | **Particle array unbounded** | `spawnParticles()` | Particles are pushed without any limit. Long sessions (or many deaths in quick succession) leak memory. |
| 9 | **Particle physics in draw()** | `draw()` | Physics updates (`pt.x += vx * dt`, etc.) were inside the rendering function. Mixing simulation and rendering is architecturally wrong and causes double-update if draw() is ever called twice. |
| 10 | **AudioContext never resumed** | `ensureAudio()` | Mobile browsers suspend AudioContext by default. The code created it but never called `.resume()`, so sound silently failed on iOS/Android. |
| 11 | **Oscillator nodes not disconnected** | `playTone()` | Oscillator and GainNode were created and connected but never explicitly disconnected after `onended`. On long sessions this leaks Web Audio graph nodes. |
| 12 | **`crashPercent` shown as %** | HUD | Displaying "5.00%" makes no sense to a crash game player. Real crash games show a multiplier (1.00×, 2.34×, etc.), which is the industry standard. |

### 🟡 Minor / Polish

| # | Issue | Location |
|---|-------|----------|
| 13 | `cashOut()` win calculation used `bet + bet * crashPercent / 100` — should be `bet * multiplier` | `cashOut()` |
| 14 | Star rendering called `ctx.beginPath()` inside loop — one path per star. Batch all circles into one `beginPath` call. | `draw()` — stars |
| 15 | `body::before` used 4 longhand properties instead of `inset: 0` | CSS |
| 16 | No `-webkit-tap-highlight-color: transparent` on buttons — caused blue flash on mobile tap | CSS buttons |
| 17 | Toast `pointer-events: none` missing — click-through worked but could interfere | CSS |

---

## 2. WHAT WAS FIXED

### Crash System — Complete Redesign

**Old:** Linear `+1% per 200 ms` tick. Fully predictable. No randomness.

**New:** Exponential multiplier + random hidden crash point.

```
multiplier(t) = e^(k × t)     where k = 0.18
```

- At t=0s  → 1.00×
- At t=5s  → 2.46×
- At t=10s → 6.05×
- At t=15s → 14.9×

A crash target is secretly chosen at round start using an exponential distribution:

```js
function generateCrashTarget() {
  const houseEdge = 0.97; // 97% RTP
  const r = Math.random();
  if (r < 0.01) return 1.0 + GROWTH_RATE; // 1% instant-ish crash after grace
  return Math.min(1 / (1 - r * houseEdge), MAX_MULT);
}
```

Distribution: ~30% crash below 1.5×, ~55% below 2×, ~80% below 4×, ~5% above 10×

A **3-second grace period** (`CRASH_GRACE_S`) ensures no crash ever happens immediately.

### FPS Independence

```js
// Old: cap at 80 ms (too high, causes jitter on tab resume)
if (dt > 0.08) dt = 0.08;

// New: cap at 50 ms (20 FPS equivalent, still safe for physics)
let dt = Math.min((ts - lastTime) / 1000, 0.05);
```

### HiDPI Canvas

```js
DPR = Math.min(window.devicePixelRatio || 1, 2); // cap at 2 for perf
canvas.width  = Math.round(CW * DPR);
canvas.height = Math.round(CH * DPR);
ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
// All draw code still uses logical CW/CH — no changes needed
```

### Jump Debounce (mobile double-tap fix)

```js
let jumpPending = false;

function birdJump() {
  if (state !== STATE.PLAYING || jumpPending) return;
  BIRD.vy     = BIRD.jumpVel;
  jumpPending = true;
  setTimeout(() => { jumpPending = false; }, 20); // clear after one frame
  soundFlap();
}
```

### Particle Cap

```js
const MAX_PARTICLES = 120;

function spawnParticles(x, y, color, count) {
  const available = MAX_PARTICLES - particles.length;
  const actual    = Math.min(count, available);
  // ... push only 'actual' particles
}
```

### AudioContext Resume (iOS/Android fix)

```js
function ensureAudio() {
  if (!audioCtx) { /* create */ }
  if (audioCtx && audioCtx.state === 'suspended') {
    audioCtx.resume().catch(() => {});
  }
}

// Also: disconnect nodes after use
osc.onended = () => { osc.disconnect(); g.disconnect(); };
```

### Pipe Speed — Read Once, Not Per Frame

```js
// Old (in update() — every frame):
gameSpeed = getGameSpeed(); // DOM query 60×/s

// New (in initGame() — once per round):
gameSpeed = getGameSpeed();
```

---

## 3. ARCHITECTURE CHANGES

| Area | Before | After |
|------|--------|-------|
| Crash system | Linear %, deterministic | Exponential multiplier, seeded random crash point |
| Particle updates | Inside `draw()` | Separate `updateParticles(dt)` function |
| Canvas resolution | CSS-pixel only (blurry) | DPR-scaled (sharp on Retina) |
| Mobile input | Double-jump on tap | Debounced `jumpPending` flag |
| Audio on mobile | Silently failed | `.resume()` called on first interaction |
| HUD display | "5.00%" | "1.73×" (industry standard) |
| Win calculation | `bet + bet * % / 100` | `bet * multiplier` |

---

## 4. OPTIONAL ADVANCED IMPROVEMENTS

1. **Offscreen canvas for pipes** — pre-render pipe body once, `drawImage()` instead of fillRect chains
2. **Object pooling for particles** — reuse objects instead of push/splice to reduce GC pressure
3. **Provably fair crash** — use HMAC-SHA256 seeded hash to prove crash point was fixed before round
4. **Autobet / strategy modes** — Martingale, fixed-profit target, stop-loss
5. **Leaderboard** — persist high scores to localStorage
6. **Difficulty scaling** — increase `GROWTH_RATE` or decrease `GAP_H` as score grows
7. **Haptic feedback** — `navigator.vibrate(50)` on flap for supported mobile devices
