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
3. **Calibrate** — sets your current tilt as the neutral position, so you can play from any angle
4. **3-2-1-GO** — the countdown plays and the game begins
5. **Game over** — see your score, save it, and compare on the leaderboard

## Leaderboard

Scores are saved both **locally** (in the browser) and **online** (Firebase Firestore). After a game you can enter your name to publish your score. The in-game leaderboard lets you toggle between your local history and the global online rankings.

The separate `firebase-leaderboard.html` page is a standalone display board designed for events and screens — it shows the top scores for all modes in real time.

## Tech Stack

- Plain HTML, CSS, and JavaScript — no framework, no build step
- Canvas API for game rendering
- `DeviceOrientationEvent` for gyroscope input
- Firebase Firestore (v9 compat SDK) for online leaderboards
- [`canvas-confetti`](https://github.com/catdad/canvas-confetti) for end-game celebration
- PWA-ready (manifest + Apple meta tags)

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire game — open this to play |
| `firebase-leaderboard.html` | Public leaderboard display page for events |
| `gibbon_logo.png` | Brand logo |
| `QR_balance_app.png` | QR code linking to the hosted game |

## Hosting

The app is a single static HTML file and can be hosted anywhere:
- GitHub Pages
- Netlify / Vercel (drag and drop)
- Any static file server

> **iOS note**: Gyroscope permission (`DeviceOrientationEvent.requestPermission`) requires the page to be served over **HTTPS**. Local `file://` won't work on iOS Safari.

## Development

All code is in `index.html`. Edit it directly — there is no build process.

Key areas inside the file:
- **Styles**: `<style>` block in `<head>`
- **Game constants**: top of `<script>` block
- **Game loop**: `updateLoop()` function
- **Gyro handling**: `handleOrientation()` and `mapOrientationToXY()`
- **Firebase config**: near the top of the `<script>` block
