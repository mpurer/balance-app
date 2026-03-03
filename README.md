# Gibbon Balance App

A mobile web game that uses your device's gyroscope to control a glowing ball across 4 balance-based game modes. Built for the Gibbon brand as a fun, competitive experience for events and slackline enthusiasts.

## Play

Open `index.html` on a mobile device (or serve it from any static host). No installation required.

> Best experienced in **landscape orientation** on a smartphone or tablet.

## Game Modes

### Level 1 — Static Target
Keep the ball steady on the center target for 30 seconds. Points are awarded based on how close you stay to the bull's-eye.

### Level 2 — Moving Target
The target jumps to a new position every 3 seconds. Chase it and stay as close as possible. Scores post to the online leaderboard.

### Level 3 — Survival Arena
Dodge an ever-growing swarm of obstacles. One hit and it's over. How long can you last?

### Level 4 — Reaction Gates
Touch as many yellow circular gates as you can in 30 seconds. Each gate disappears the moment you hit it and a new one spawns elsewhere.

## How It Works

1. **Enable Gyro** — grants permission for device motion sensors (required on iOS)
2. **Select a mode** — choose your level
3. **Start** — sets your current tilt as the neutral position, so you can play from any angle
4. **3-2-1-GO** — the countdown plays and the game begins
5. **Game over** — your score is published to the leaderboard automatically

## Event Setup

There are two operating modes for running the game at an event. Switch between them by opening a different file on the game device — `index.html` is always kept as a safe fallback.

### Standard Mode
Players register on the game device itself.

| Device | File |
|---|---|
| Game device (phone/tablet) | `index.html` |
| TV / screen | `firebase-leaderboard.html` |

### Queue Mode (karaoke-style waiting list)
Players sign up on their own phones by scanning a QR code. They join a live queue and watch the screen for their turn. Staff controls the game device.

| Device | File |
|---|---|
| Players' phones | `signup.html` (link via QR code) |
| Game device | `index-queue.html` |
| TV / screen | `firebase-leaderboard-queue.html` |

**Game device controls in queue mode:**
- **▶ Play** — loads the next player and starts the level selection
- **Skip** — skips the current player, moves to the next one
- **↑ Front** — bumps any waiting player to the front of the queue

## Leaderboard

Scores are saved **online** (Firebase Firestore) and displayed automatically after each game. The player's row is highlighted in the leaderboard so they can see their rank.

The TV display pages show the top scores for Moving Target and Survival Arena in real time, updating live without a page refresh.

## Registration & Newsletter

Players register with their email, first name, and a username. Registration includes a newsletter consent checkbox for the Gibbon Slacklines newsletter. All registrations are stored in Firebase Firestore.

Use `export.html` to load registrations and download a Klaviyo-compatible CSV for newsletter import. Supports optional date filtering for multi-event use.

## Tech Stack

- Plain HTML, CSS, and JavaScript — no framework, no build step
- Canvas API for game rendering
- `DeviceOrientationEvent` for gyroscope input
- Firebase Firestore (v9 compat SDK) for online leaderboards, registrations, and queue
- [`canvas-confetti`](https://github.com/catdad/canvas-confetti) for end-game celebration
- PWA-ready (manifest + Apple meta tags)

## Files

| File | Purpose |
|---|---|
| `index.html` | Standard mode — full game with on-device registration |
| `index-queue.html` | Queue mode — game device; auto-loads players from queue |
| `signup.html` | Queue mode — players' phones; register and join the queue |
| `firebase-leaderboard.html` | TV display for standard mode |
| `firebase-leaderboard-queue.html` | TV display for queue mode (includes live queue panel) |
| `export.html` | Admin tool — export registrations as CSV |
| `gibbon_logo.png` | Brand logo |
| `QR_balance_app.png` | QR code linking to the hosted game |

## Hosting

All files are static HTML and can be hosted anywhere:
- GitHub Pages
- Netlify / Vercel (drag and drop)
- Any static file server

> **iOS note**: Gyroscope permission (`DeviceOrientationEvent.requestPermission`) requires the page to be served over **HTTPS**. Local `file://` won't work on iOS Safari.

## Development

All code lives directly in the HTML files — there is no build process.

Key areas inside `index.html` / `index-queue.html`:
- **Styles**: `<style>` block in `<head>`
- **Game constants**: top of `<script>` block
- **Game loop**: `updateLoop()` function
- **Gyro handling**: `handleOrientation()` and `mapOrientationToXY()`
- **Firebase config**: near the top of the `<script>` block
- **Queue logic** *(index-queue.html only)*: `subscribeToQueue()`, `updateQueueUI()`, `markCurrentDone()`, `bumpToFront()`
