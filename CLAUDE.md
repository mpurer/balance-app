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
├── play-fibo.html                   # FIBO mode: players' phones; register + queue + play all-in-one
├── firebase-leaderboard-fibo.html   # TV display: live leaderboard + queue + staff Skip button (FIBO mode)
├── export.html                      # Admin: load registrations from Firestore, download CSV
├── gibbon_logo.png
├── QR_balance_app.png
└── QR_fibo.png                      # QR code pointing to play-fibo.html (must be added manually)
```

## Three Operating Modes

### Standard Mode (index.html + firebase-leaderboard.html)
Players register on the game device itself. Staff opens `index.html` on the game device and `firebase-leaderboard.html` on the TV.

### Queue Mode (index-queue.html + signup.html + firebase-leaderboard-queue.html)
Players sign up on their own phones by scanning a QR code pointing to `signup.html`. They join a live waiting list. Staff opens `index-queue.html` on the game device — it auto-loads the next player and shows Skip / Bump-to-Front controls. The TV shows `firebase-leaderboard-queue.html` which includes a queue panel beside the leaderboards.

### FIBO Mode (play-fibo.html + firebase-leaderboard-fibo.html)
Designed for trade fairs with a physical balance board. Players scan a QR code, register and join the waiting list **on their own phone**, then play the game on their own phone when their turn comes. No dedicated game device needed. Only Survival Arena (Level 3) is available. The TV shows `firebase-leaderboard-fibo.html` which has a staff Skip button.

**Switching modes = opening a different file. `index.html` is never modified.**

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
| `leaderboardFiboSurvival` | play-fibo.html, firebase-leaderboard-fibo.html | `playerName, score, timestamp` |
| `registrations` | All game files, export.html | `firstName, email, username, newsletterConsent, createdAt, lastPlayedAt, gamesPlayed` |
| `queue` | Queue mode files | `username, firstName, email, status, joinedAt, order` |
| `queueFibo` | FIBO mode files | `username, firstName, email, status, joinedAt, order, readyAt, attemptsUsed, liveScore, gameStatus, finalRank, lastScoreDocId` |

### Queue Collection — `status` values (both `queue` and `queueFibo`)
- `waiting` — in the waiting list
- `ready` — it's their turn; 60s countdown started (`queueFibo` only)
- `playing` — currently playing (includes all attempts; doc stays `playing` between attempts)
- `done` — finished all attempts or voluntarily ended
- `skipped` — timed out (ready screen or game-over screen) or skipped by staff

### `queueFibo` — extra fields written during gameplay
| Field | When written | Purpose |
|---|---|---|
| `attemptsUsed` | After each game | Incremented via `FieldValue.increment(1)`; max 3 |
| `liveScore` | Every ~3s during play | Current score for TV animation; 0 at countdown start |
| `gameStatus` | Throughout gameplay | `'countdown'` → `'playing'` → `'gameover'` (mid-session) or `'done'` (final) |
| `finalRank` | After each game | Leaderboard rank at that moment |
| `lastScoreDocId` | After each game | Firestore doc ID of newly saved score row — TV uses this to flash the exact row |

**Two-step final write**: on the last attempt, `gameStatus:'done'` is written first (while `status` stays `'playing'` so the TV subscription still sees it), then after 2.5s `status:'done'` is written and `advanceQueue()` is called. This gives the TV time to read the final score and trigger the flash before the doc leaves the subscription.

### Queue Ordering
`order` is a float set to `Date.now()` on join. FIFO natural sort. Bump to front sets `order = currentMinOrder - 1000`. **All queue queries use client-side sort** (no `orderBy` in Firestore) to avoid needing a composite index.

### Composite Index Note
Leaderboard queries that use two `orderBy` fields (e.g. `orderBy('score').orderBy('timestamp')`) require a composite index in Firestore. **Only use a single `orderBy('score','desc')`** on new collections to avoid this — the `leaderboardFiboSurvival` queries are intentionally single-field for this reason.

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

**FIBO mode (play-fibo.html — all on player's own phone):**
```
Registration (Vorname + Nachname + Benutzername + Email + Newsletter)
  → Waiting (live queue position) → "Du bist dran!" (60s SVG ring countdown)
  → tap Start → [iOS gyro permission] → [rotate overlay enforced in landscape]
  → 3-2-1-GO → Survival Game → Game Over (score + leaderboard + rank)
  → [Nochmal →] up to 3 attempts total, or [Beenden] to end early
  → after 3 attempts: "Versuche aufgebraucht" screen → re-register
  → 60s idle timeout on game-over screen → auto-kick → "Zeit abgelaufen"
```

## Conventions & Patterns

- All game logic is in one `<script>` block at the bottom of `index.html` / `index-queue.html`
- Level-specific draw/update logic is branched inside `updateLoop()` with `if (currentLevel === N)`
- HTML escaping via `escapeHtml()` applied to all player names before rendering (XSS prevention)
- iPad and phone tilt mapping are handled differently inside `mapOrientationToXY()`
- `mulberry32` seeded RNG is used for survival obstacles (deterministic patterns)
- Confetti on game end via `canvas-confetti` CDN

## FIBO Mode — Key Functions (play-fibo.html)

| Function | Purpose |
|---|---|
| `joinQueue(player)` | Creates `queueFibo` doc with `attemptsUsed:0`; reattaches on page refresh; calls `advanceQueue()` |
| `subscribeToOwnDoc()` | `onSnapshot` on player's own doc; reacts to `ready` / `skipped` status changes |
| `subscribeToQueuePosition()` | `onSnapshot` on all `waiting` docs; updates live position display |
| `advanceQueue()` | Guards for existing active player, then sets next `waiting` player to `ready` |
| `startReadyCountdown()` | 60s SVG ring countdown; calls `handleTimeout()` on expiry |
| `handleTimeout()` | Sets own doc to `skipped`, calls `advanceQueue()`, shows skipped screen |
| `beginGame()` | Hardcodes `beta0=gamma0=0` (flat-phone centre), writes `gameStatus:'countdown'`, runs 3-2-1-GO, then switches to `gameStatus:'playing'` |
| `endGame()` | Saves score, increments `attemptsUsed`, does two-step Firestore write; only marks `status:'done'` + advances queue after attempt 3 or Beenden |
| `startGameOverTimeout()` | Starts 60s `setInterval` on game-over screen; calls `handleGameOverTimeout()` if player idles |
| `handleGameOverTimeout()` | Cancels `_doneMarkTimer`, marks doc `done`, advances queue, shows "Zeit abgelaufen" |
| `sizeCanvas()` | Computes `canvasScale = min(w,h)/390` — normalises obstacle sizes/speeds across screen sizes |

## Things to Watch Out For

- **No build step** — edit HTML files directly
- Every HTML file has its own copy of the Firebase config — keep them in sync if credentials change
- Screen orientation (portrait vs landscape, 0/90/180/270) affects tilt axis mapping — test gyro changes on device
- Firestore queries on `queue` / `queueFibo` must **not** use `orderBy` on a field different from the `where` field, or a composite index is required (which we avoid by sorting client-side)
- `index.html` must never be modified as part of queue or FIBO mode work — it is the fallback
- `play-fibo.html` and `index-queue.html` share the same survival game constants and logic — if you change game balance, update both files
- `queueFibo` uses an extra `ready` status (not present in `queue`) — don't confuse the two collections
- `QR_fibo.png` must be placed in the project root manually; `firebase-leaderboard-fibo.html` shows a placeholder if it's missing
- `firebase-leaderboard-fibo.html` uses `260326_GIB_Screendesign_YF_V2.jpg` as a 1920×1080 background image; panels are absolutely positioned over the template's static placeholders
- **Parallel play guard**: `queueFibo` doc stays `status:'playing'` between attempts (attempt 1→2→3); `status:'done'` is only written after all attempts used or Beenden tapped. Do not mark done after individual games.
- **`_doneMarkTimer`**: a 2.5s `setTimeout` that writes `status:'done'` after the final attempt's two-step write. Always cancel this timer (`clearTimeout(_doneMarkTimer)`) before any forced early exit.
- **`_gameOverTimer`**: a 60s `setInterval` that auto-kicks idle players from the game-over screen. Cancel with `clearGameOverTimeout()` whenever a button is pressed.
- **Firestore write budget**: liveScore is written every ~3s during play. At a full 8-hour fair this is ~5,700 writes/day — well within the 20K free tier limit. Do not reduce the interval below 2s.
- **Live score animation on TV**: the leaderboard RAF loop counts at `+10 pts/sec` (matching `SURVIVAL_SCORE_MULT`). It is rebased on each Firestore update. On `gameStatus:'done'` it snaps to the exact final score.
- **Flash precision**: the TV leaderboard identifies the exact row to flash via `tr[data-docid]` matched against `lastScoreDocId` from the queueFibo doc. Name-based matching is only the fallback for skip cases.
- **`canvasScale`**: computed in `sizeCanvas()` as `min(canvas.width, canvas.height) / 390`. Applied to obstacle size, speed, and spawn margins so difficulty is consistent across screen sizes.

## FIBO Improvements

### Planned feature set
| # | Description | Status |
|---|---|---|
| 1 | Screen-size-normalised difficulty (`canvasScale`) | ✅ Done |
| 2 | Live score in TV leaderboard NOW PLAYING panel | ✅ Done |
| 4 | White-box bug on waiting/ready screens (removed logo img) | ✅ Done |
| 5 | Force landscape from ready screen onwards | ✅ Done |
| 6 | Max 3 attempts per queue slot | ✅ Done |
| 7 | Reset / delete button in export.html | ✅ Done |
| 8 | Add Nachname field (combined into firstName) | ✅ Done |
| 9 | Disclaimer text on TV leaderboard | ✅ Done |
| 10 | Landscape game-over card + rename "Warteschlange" → "Beenden" | ✅ Done |
| 11 | Disable gyro calibration (hardcode `beta0=gamma0=0`) | ✅ Done |

### Post-launch bug fixes & enhancements
| # | Description | Status |
|---|---|---|
| 12 | Fix parallel playing: doc stays `playing` between attempts; `done` only after attempt 3 or Beenden | ✅ Done |
| 13 | iPhone notch / Dynamic Island safe area: `env(safe-area-inset-*)` padding on `#gameOverCard` | ✅ Done |
| 14 | Live score write frequency: 3s interval (safe within Firebase free tier) + RAF interpolation on TV | ✅ Done |
| 15 | Export HTML: rename "First Name" column → "Name" | ✅ Done |
| 16 | Two-step final write + `lastScoreDocId` for precise leaderboard row flash | ✅ Done |
| 17 | `gameStatus:'countdown'` prevents TV score counter starting during 3-2-1 | ✅ Done |
| 18 | TV leaderboard: animated 3→2→1 countdown (JS timer + CSS `countpop` keyframe) | ✅ Done |
| 19 | TV leaderboard: flash exact row by `data-docid` instead of all rows matching player name | ✅ Done |
| 20 | Flash row: 9s duration + 3 blinks in first ~2s | ✅ Done |
| 21 | 60s idle timeout on game-over screen (all attempts); auto-kick → "Zeit abgelaufen" | ✅ Done |
