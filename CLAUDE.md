# Space Runner — CLAUDE.md

A space-themed endless runner built by Nate and Kadyn Sharpe with Claude.
Hosted on GitHub Pages; Firebase Firestore for the public leaderboard.

---

## Project Structure

Everything lives in a single self-contained file — no build step, no dependencies to install.

| File | Purpose |
|---|---|
| `index.html` | The entire game — HTML, CSS, and JS in one file |
| `CLAUDE.md` | This file |

**GitHub repo:** `https://github.com/nssharpe/space-runner`
**Live URL:** `https://nssharpe.github.io/space-runner/`

To test locally just open `index.html` in a browser. Firebase won't work from
`file://` — run `python -m http.server 8080` in the folder and open
`http://localhost:8080` instead (localhost must be in Firebase Authorized Domains).

---

## Architecture

### Canvas

- Logical size: **480 × 720** px — all game math uses these coordinates
- CSS `transform`/positioning scales it to fill any viewport
- `resizeCanvas()` recalculates CSS on every `window.resize`

### State machine

```
"username" → "start" → "playing" → "dead" → "gameover"
                ↑                              |
                └──────── "start" ←───────────┘
                               ↑
                            "shop" (HANGAR screen)
```

`gamePhase` is the single source of truth. `setPhase(phase)` manages all
screen show/hide and fires side effects (score submit, leaderboard load, etc.).

### Global game state `S`

Reset to this shape every new game via `resetGameState()`:

```js
S = {
  player:        { y, vy, hp, iFrames },
  obstacles:     [],   // { type, fieldX, y, width, height }
  coins:         [],   // { fieldX, y }
  powerups:      [],   // { id, fieldX, y }
  bullets:       [],   // { x, y }  — blaster projectiles (screen coords)
  stars:         [],   // background parallax dots
  scrollSpeed,         // px/frame, ramps up over time
  spawnInterval,       // frames between obstacle rows, ramps down
  score,               // seconds survived (incremented 1/60 per frame)
  frameCount,
  camera,              // field-x shown at screen left edge
  keys,                // held keyboard/touch state
  coinsThisRun,
  coinMultiplier,      // 1 or 2 (from 2x-coins power-up)
  activePowerup,       // null or { id, framesLeft, totalFrames }
  deathX, deathY, deathFrame,
}
```

### Coordinate system — IMPORTANT

The player is always fixed at `PLAYER_X = CANVAS_W / 2` horizontally.
Left/right arrow keys move the **camera**, not the player.

```
screenX = obs.fieldX - S.camera
```

Obstacles and coins store `fieldX` (world space). Only the camera moves.
`spawnRow()` spawns **3 tile copies** per row (at `camera - CANVAS_W`, `camera`,
`camera + CANVAS_W`) so obstacles always fill the viewport no matter how far
the player has shifted the field.

---

## Key Constants

```js
const CANVAS_W     = 480;
const CANVAS_H     = 720;
const PLAYER_X     = 240;   // fixed horizontal center
const PUSH_H       = 20;    // push-platform height (px)
const DMG_SZ       = 44;    // damage hazard bounding box
const PLAYER_HIT_R = 11;    // player collision circle radius
const COIN_R       = 16;    // coin draw radius
const COIN_HIT_R   = 20;    // coin collection radius (generous)
const BULLET_R     = 5;     // blaster projectile radius
const POWERUP_R    = 13;    // power-up capsule radius
```

### CFG object (tuning)

```js
CFG = {
  player: { w:32, h:40, speed:4, maxHp:3, iFrames:90 },
  scroll: { initial:2.5, increment:0.0005, max:14 },
  spawn:  { intervalInitial:80, intervalMin:25, intervalDecrement:0.012 },
  shift:  { speed:3 },
  stars:  { count:80 },
  death:  { frames:40 },
}
```

---

## Obstacle System

Two types, both use `fieldX` world coordinates:

| Type | Shape | Behavior |
|---|---|---|
| `push` | Gray rounded rect | AABB collision; snaps player **below** it (pushes down) |
| `damage` | Orange 6-point star | Circle collision; −1 HP, i-frames, knockback; obstacle removed on hit |

`PATTERNS` array holds named row configurations (mix of push/damage with
`xFrac` and `wFrac` fractions of `CANVAS_W`). Push widths are multiplied
by 1.1 at spawn time. New patterns go in `PATTERNS` — they're picked randomly.

### Collision notes

- Push uses AABB; lands player below when `obs.y + h/2 < p.y` (platform is above player center)
- Damage uses circle-circle with `PLAYER_HIT_R` vs `obsR = Math.min(w,h)/2 * 0.55`
- **Shield** power-up: `continue` skips the damage branch entirely
- **Blaster** power-up: contact destroys the hazard; bullets also destroy on hit

---

## Coins

- Spawn in rows of 3–5 every 150 frames via `spawnCoinRow()`
- Skip spawn positions that overlap any obstacle with `y` in `[-150, 150]`
- Collection uses circle-circle: `PLAYER_HIT_R + COIN_HIT_R`
- `S.coinMultiplier` (1 or 2) is applied at collection
- `totalCoins` persists in `localStorage` key `spaceRunnerCoins`
- `S.coinsThisRun` shown in HUD (top-right) and on game-over screen

---

## Power-ups

```js
POWERUPS = [
  { id:'shield',  label:'SHD', color:'#44AAFF', duration:360 },  // 6s
  { id:'blaster', label:'BLS', color:'#FF4444', duration:540 },  // 9s
  { id:'x2coins', label:'x2',  color:'#FFD700', duration:480 },  // 8s
]
```

- Spawn every 700 frames (guarded by `features.powerups` flag)
- Active power-up shown as a draining colored bar below the score
- Blasters fire a bullet from player nose every 15 frames; bullets move upward
  at 10 px/frame and destroy damage obstacles on contact

---

## Skins

```js
SKINS = [
  { id:'default', name:'CYAN',    price:0   },
  { id:'ember',   name:'EMBER',   price:100 },
  { id:'phantom', name:'PHANTOM', price:200 },
  { id:'solar',   name:'SOLAR',   price:300 },
  { id:'void',    name:'VOID',    price:400 },
  { id:'nova',    name:'NOVA',    price:500 },
  { id:'crimson', name:'CRIMSON', price:600 },
]
```

Each skin has: `body`, `outline`, `cockpit`, `flameA`, `flameB` color strings.
`drawPlayer()` looks up `equippedSkin` to pick colors.
`drawSkinPreview(canvasEl, skin)` renders a mini ship on a 56×68 canvas for the HANGAR.

Persistence:
- `spaceRunnerSkin` → equipped skin id
- `spaceRunnerOwned` → JSON array of owned skin ids

---

## Feature Flags (Mod Panel)

```js
FEATURE_DEFAULTS = {
  powerups:   true,
  coins:      true,
  breakTimer: true,
  hardMode:   false,
}
```

Each flag saved as `spaceRunnerFeature_<key>` in localStorage.
`loadFeatures()` called at init; `saveFeature(key, val)` persists a change.

Hard mode doubles the scroll/spawn ramp rate and raises the speed cap by 30%.

**Accessing the mod panel:** Set pilot name to `MODkadyn1`. A hidden orange
**⚙ MOD** button appears on the start screen. The panel has:
- Feature toggles
- MY ACCOUNT resets (coins, score, attempts, skins, full wipe)
- ALL ACCOUNTS reset (clears Firebase leaderboard via batch delete)
- Set-coins tool for testing

---

## Break Timer

Every 5 deaths a 60-second "RECHARGING" modal blocks the next game.
Guarded by `features.breakTimer` (can be disabled in mod panel).
Attempt count stored in `localStorage` key `spaceRunnerAttempts`.

---

## Firebase / Leaderboard

- SDK: Firebase compat v10 loaded via CDN `<script>` tags
- Project: `space-runner-a2352`
- Collection: `leaderboard` — docs: `{ username, score (int), timestamp }`
- Submit: `fbSubmitScore(name, score)` — called via SUBMIT SCORE button on game-over
- Fetch: `fbFetchLeaderboard()` — top 10 by score desc
- All Firebase calls wrapped in try/catch; game works fine offline
- `FB_CONFIGURED` flag prevents errors when config is placeholder

**Firestore security rules** (set in Firebase console):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /leaderboard/{doc} {
      allow read: if true;
      allow create: if request.resource.data.score is number
                    && request.resource.data.username.size() <= 20;
    }
  }
}
```

---

## Mobile Controls

On-screen arrow buttons shown only during gameplay and only on touch devices
(`'ontouchstart' in window || navigator.maxTouchPoints > 0`).

Layout: ↑/↓ pad bottom-left, ←/→ pad bottom-right.
Each button uses `touchstart`/`touchend`/`touchcancel` + mouse equivalents
to set/clear entries in `S.keys`, identical to keyboard input.
Visibility toggled by `setPhase()` via `#mobileControls.active` class.

---

## Render Order (each frame)

1. Background gradient
2. Stars (parallax, wrap at bottom)
3. Obstacles (push then damage)
4. Coins
5. Power-up capsules
6. Blaster bullets
7. Player ship (flickers during i-frames) — playing phase only
8. HUD: score (top-center), hearts (top-left), coin count (top-right),
   active power-up bar (below score) — playing phase only
9. Death shockwave animation — dead phase only

---

## Deployment

1. Edit `index.html`
2. `git add index.html && git commit -m "..." && git push origin master`
3. GitHub Pages auto-deploys within ~1 minute
4. Live at `https://nssharpe.github.io/space-runner/`

The Firebase config is already filled in with real values — no extra setup needed.

---

## Ideas / Future Work

Things discussed but not yet built:
- Auto-submit score on death (remove the manual SUBMIT button)
- Enemy spaceships that appear after 60s and get more aggressive after 120s
- Go to try-again screen automatically after the battery recharge countdown
- More power-up types
- More skins
- Additional mod panel feature flags as new content is pre-built
