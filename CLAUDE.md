# CLAUDE.md — Gibbon Balance App

## Project Overview

Single-file HTML mobile web game for the Gibbon brand. Players tilt their device to control a glowing blue ball and compete across 4 game modes. Everything lives in `index.html` — there is no build step, no framework, no package manager.

## File Structure

```
balance-app/
├── index.html               # Entire app (~1,223 lines): HTML + CSS + JS
├── firebase-leaderboard.html # Standalone public leaderboard display page (for events/screens)
├── gibbon_logo.png
└── QR_balance_app.png
```

## Architecture

- **Pure vanilla JS** — no bundler, no npm, no TypeScript
- **Single RAF game loop**: `updateLoop()` drives all levels via `requestAnimationFrame`
- **Canvas rendering**: `<canvas id="targetCanvas">` for game graphics; the ball (`#dot`) is a CSS element
- **Gyroscope input**: `DeviceOrientationEvent` — requires permission on iOS 13+
- **Calibration**: Records `beta0`/`gamma0` at game start so movement is relative to the player's starting position

## Game Modes

| Level | Name | Duration | Mechanic |
|---|---|---|---|
| 1 | Static Target | 30s | Hold ball on fixed center target |
| 2 | Moving Target | 30s | Chase target that repositions every 3s |
| 3 | Survival Arena | Endless | Dodge spawning obstacles (shapes + lines) |
| 4 | Reaction Gates | 30s | Touch yellow circular gates to score |

## Key Constants (index.html)

| Constant | Value | Purpose |
|---|---|---|
| `GAME_DURATION` | 30 | Seconds for timed modes |
| `DOT_RADIUS` | 18 | Ball collision radius (px) |
| `MAX_TILT` | 30 | Max gyro angle mapped to screen edge (degrees) |
| `TARGET_MOVE_SPEED` | 110 | Level 2 target velocity (px/s) |
| `SURVIVAL_SEED` | 123456 | Seeded RNG for reproducible obstacle patterns |

## State Variables

```js
score, timeLeft, gameActive, currentLevel
dotPos { x, y }       // current ball position
dotPos0 { x, y }      // calibration reference
beta0, gamma0         // calibration gyro angles
survival { time, shapes[], lines[], rng, ... }
reactionGates { currentGate, previousGatePos, gatesCleared }
currentLeaderboardMode  // 'local' | 'online'
```

## Firebase / Leaderboard

- **Project**: `gibbon-balance-game`
- **SDK**: Firebase compat v9.0.0 loaded from CDN
- **Firestore collections**: `leaderboardLevel1`, `leaderboardLevel2`, `leaderboardSurvival`, `leaderboardReactionGates`
- **Document fields**: `{ playerName, score, timestamp }`
- **Local storage keys**: `tilt_dot_leaderboard_level1_v1` through `_level4_v1`
- Max 50 records stored per level (both local and Firestore queries)
- API key is embedded directly in `index.html` (intentional — client-side only app)

## UI Flow

```
Enable Gyro → Level Select → Calibrate → 3-2-1-GO Countdown → Game → Game Over (save / leaderboard)
```

## Conventions & Patterns

- All game logic is in one `<script>` block at the bottom of `index.html`
- Level-specific draw/update logic is branched inside `updateLoop()` with `if (currentLevel === N)`
- HTML escaping is applied to player names before rendering (XSS prevention)
- iPad and phone tilt mapping are handled differently inside `mapOrientationToXY()`
- `mulberry32` seeded RNG is used for survival obstacles (deterministic patterns)
- Confetti on game end via `canvas-confetti` CDN

## Things to Watch Out For

- There is **no build step** — editing `index.html` is the only way to change the app
- The Firebase API key is public (by design for a client-side Firestore app — Firestore rules handle security)
- Screen orientation (portrait vs landscape, 0/90/180/270) affects tilt axis mapping — test changes on device
- `firebase-leaderboard.html` has its own copy of Firebase config and is **independent** of `index.html`
