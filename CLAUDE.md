# CLAUDE.md — Gibbon Balance App

## Project Overview

Mobile web game for the Gibbon brand. Players tilt their device to control a glowing blue ball and compete across 4 game modes. No build step, no framework, no package manager — everything is plain HTML/CSS/JS.

## File Structure

```
balance-app/
├── index.html                       # Standard mode: full game with on-device registration
├── index-queue.html                 # Queue mode: game device; auto-loads players from Firestore queue
├── signup.html                      # Queue mode: players' phones; register + join queue via QR code
├── firebase-leaderboard.html        # TV display: live leaderboard (standard mode)
├── firebase-leaderboard-queue.html  # TV display: live leaderboard + queue panel (queue mode)
├── export.html                      # Admin: load registrations from Firestore, download CSV
├── gibbon_logo.png
└── QR_balance_app.png
```

## Two Operating Modes

### Standard Mode (index.html + firebase-leaderboard.html)
Players register on the game device itself. Staff opens `index.html` on the game device and `firebase-leaderboard.html` on the TV.

### Queue Mode (index-queue.html + signup.html + firebase-leaderboard-queue.html)
Players sign up on their own phones by scanning a QR code pointing to `signup.html`. They join a live waiting list. Staff opens `index-queue.html` on the game device — it auto-loads the next player and shows Skip / Bump-to-Front controls. The TV shows `firebase-leaderboard-queue.html` which includes a queue panel beside the leaderboards.

**Switching modes = opening a different file on the game device. `index.html` is never modified.**

## Architecture

- **Pure vanilla JS** — no bundler, no npm, no TypeScript
- **Single RAF game loop**: `updateLoop()` drives all levels via `requestAnimationFrame`
- **Canvas rendering**: `<canvas id="targetCanvas">` for game graphics; the ball (`#dot`) is a CSS element
- **Gyroscope input**: `DeviceOrientationEvent` — requires permission on iOS 13+
- **Calibration**: Records `beta0`/`gamma0` at game start so movement is relative to the player's starting position

## Game Modes

| Level | Name | Duration | Mechanic |
|---|---|---|---|
| 1 | Static Target | 30s | Hold ball on fixed center target *(hidden at fairs)* |
| 2 | Moving Target | 30s | Chase target that repositions every 3s |
| 3 | Survival Arena | Endless | Dodge spawning obstacles (shapes + lines) |
| 4 | Reaction Gates | 30s | Touch yellow circular gates to score *(hidden at fairs)* |

Levels 1 and 4 are hidden at fairs via `data-fair-hidden="true"` on the level buttons + CSS `display:none`. The JS for these levels is still intact.

## Key Constants (index.html / index-queue.html)

| Constant | Value | Purpose |
|---|---|---|
| `GAME_DURATION` | 30 | Seconds for timed modes |
| `DOT_RADIUS` | 18 | Ball collision radius (px) |
| `MAX_TILT` | 30 | Max gyro angle mapped to screen edge (degrees) |
| `TARGET_MOVE_SPEED` | 110 | Level 2 target velocity (px/s) |
| `SURVIVAL_SEED` | 123456 | Seeded RNG for reproducible obstacle patterns |

## State Variables (game files)

```js
score, timeLeft, gameActive, currentLevel
currentPlayer        // { email, firstName, username } — null when no player loaded
dotPos { x, y }      // current ball position
beta0, gamma0        // calibration gyro angles
survival { time, shapes[], lines[], rng, ... }
reactionGates { currentGate, previousGatePos, gatesCleared }
// Queue mode only (index-queue.html):
currentQueueDocId    // Firestore doc ID of the player currently on device
pendingQueueEntries  // sorted waiting list from onSnapshot
```

## Firebase / Firestore

- **Project**: `gibbon-balance-game`
- **SDK**: Firebase compat v9.0.0 loaded from CDN
- **API key**: embedded in every HTML file (intentional — client-side Firestore app; Firestore rules handle security)

### Collections

| Collection | Used by | Fields |
|---|---|---|
| `leaderboardLevel2` | Game files, TV display | `playerName, score, timestamp` |
| `leaderboardSurvival` | Game files, TV display | `playerName, score, timestamp` |
| `leaderboardLevel1` | Game files | `playerName, score, timestamp` |
| `leaderboardReactionGates` | Game files | `playerName, score, timestamp` |
| `registrations` | Both game files, export.html | `firstName, email, username, newsletterConsent, createdAt, lastPlayedAt, gamesPlayed` |
| `queue` | Queue mode files | `username, firstName, email, status, joinedAt, order` |

### Queue Collection — `status` values
- `waiting` — in the waiting list
- `playing` — currently on the game device (shown as "Now Playing" on TV)
- `done` — finished playing
- `skipped` — skipped by staff

### Queue Ordering
`order` is a float set to `Date.now()` on join. FIFO natural sort. Bump to front sets `order = currentMinOrder - 1000`. **All queue queries use client-side sort** (no `orderBy` in Firestore) to avoid needing a composite index.

## Registration Flow (both game modes)

1. **Email lookup** → `registrations` collection
2. **Returning user** → fast-path, skip form
3. **New user** → firstName + username + newsletter consent checkbox → saved to `registrations`
4. Username uniqueness is checked against `registrations` before saving

## Fair Mode (index.html) — Game Over Flow

```
endGame() → save score (local + Firestore) → show rank → render online leaderboard
         → highlight player's row (by Firestore doc ID) → 30s auto-reset → registration screen
```
`nextPlayerBtn` skips the timer and goes straight to registration.

## Queue Mode (index-queue.html) — Key Functions

| Function | Purpose |
|---|---|
| `showQueueScreen()` | Show queue overlay, start `subscribeToQueue()` |
| `subscribeToQueue()` | `onSnapshot` on `status=='waiting'`, sorts by `order` client-side |
| `updateQueueUI(entries)` | Renders Next Up player + waiting list with Bump buttons |
| `markCurrentDone()` | Sets current queue doc to `done`, clears `currentQueueDocId` |
| `bumpToFront(docId)` | Sets `order = pendingQueueEntries[0].order - 1000` |
| `goToNewPlayer()` | Calls `markCurrentDone()` then `showQueueScreen()` |

## UI Flow

**Standard mode:**
```
Enable Gyro → Registration → Level Select → Start → 3-2-1-GO → Game → Game Over → Registration
```

**Queue mode:**
```
Enable Gyro → Queue Screen → [Play] → Level Select → Start → 3-2-1-GO → Game → Game Over → Queue Screen
                           → [Skip] → next player shown
```

## Conventions & Patterns

- All game logic is in one `<script>` block at the bottom of `index.html` / `index-queue.html`
- Level-specific draw/update logic is branched inside `updateLoop()` with `if (currentLevel === N)`
- HTML escaping via `escapeHtml()` applied to all player names before rendering (XSS prevention)
- iPad and phone tilt mapping are handled differently inside `mapOrientationToXY()`
- `mulberry32` seeded RNG is used for survival obstacles (deterministic patterns)
- Confetti on game end via `canvas-confetti` CDN

## Things to Watch Out For

- **No build step** — edit HTML files directly
- `firebase-leaderboard.html` and `firebase-leaderboard-queue.html` each have their own copy of the Firebase config
- Screen orientation (portrait vs landscape, 0/90/180/270) affects tilt axis mapping — test gyro changes on device
- Firestore queries on `queue` must **not** use `orderBy` on a field different from the `where` field, or a composite index is required (which we avoid by sorting client-side)
- `index.html` must never be modified as part of queue mode work — it is the fallback
