# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MikiPastebin is a single-page app for real-time text synchronization and temporary file sharing between two devices. No build system — the entire application lives in `index.html` with Firebase loaded from CDN.

## Running Locally

Open `index.html` directly in a browser. For local dev, secrets must be provided manually since CI normally injects them. The placeholders in `index.html` (`PLACEHOLDER_API_KEY`, etc.) need to be replaced with real values.

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml`, which:
1. Uses `sed` to inject secrets directly into `index.html` (replacing `PLACEHOLDER_*` strings)
2. Deploys to GitHub Pages

Required GitHub Actions secrets: `FIREBASE_API_KEY`, `FIREBASE_AUTH_DOMAIN`, `FIREBASE_DATABASE_URL`, `FIREBASE_PROJECT_ID`, `FIREBASE_STORAGE_BUCKET`.

The injected Firebase web config is **not** sensitive — the `apiKey` only identifies the project. It is injected via secrets purely to keep it out of the committed source; the real access control is the security rules (see below). `config.js.example` is just a reference listing the values.

## Security model

Access is controlled by **Firebase Authentication (Google sign-in)** plus **security rules**, NOT by a client-side password. The earlier `PASSWORD_HASH` gate was cosmetic — a SHA-256 hash shipped in public HTML, trivially bypassed (the data lives in Firebase, not behind the JS check). It has been removed.

- **Auth**: the overlay offers "Entrar con Google" (`signInWithPopup`). `onAuthStateChanged` reveals the app only when signed in.
- **Authorisation**: enforced server-side. `database.rules.json` and `storage.rules` restrict read+write to a single `auth.uid` (the owner). These files are reference copies — apply them in the Firebase Console (Realtime Database → Rules, Storage → Rules) with your own UID.
- **Setup**: in Firebase Console enable Google as a sign-in provider, add the GitHub Pages domain to Authorized domains, sign in once to obtain your UID (Authentication → Users), then paste that UID into both rule sets.

> Never put a privileged token (e.g. a GitHub PAT) into the client HTML. A prior
> "self-destruct" feature injected a repo-deleting token into the public page —
> any visitor could have read it and deleted the repo. It has been removed.

## Architecture

All code is in `index.html`. Key sections:

- **Auth gate**: Firebase Authentication with Google sign-in. App is shown via `onAuthStateChanged`; real authorisation is enforced by the security rules (single owner `auth.uid`).
- **Paste sync**: Firebase Realtime Database paths `paste/ordenador1` and `paste/ordenador2`. `onValue()` listeners for reads; debounced (500ms) `set()` writes. A `syncing` flag per pane prevents write-loops.
- **File hosting**: Firebase Storage under `files/{uuid}/{filename}`. Metadata (name, size, path, url, uploadedAt, expiresAt) stored in Realtime Database under `files/`. TTL is 10 minutes, max 100 MB, max 1 file active at a time. Cleanup runs via `cleanupExpired()`, called both from the `onValue` listener (on load / data changes) and from the 1s countdown `setInterval`, so files are removed ~1s after expiry while the app is open (client-side only — nothing runs when no client is open). Download URLs come from `getDownloadURL()`, which returns a URL with a `?token=...` that grants read access despite the locked-down Storage rules; the hand-built `firebasestorage.googleapis.com/v0/b/...?alt=media` URL is an unauthenticated request and returns 403 under those rules.
- **UI**: Dual-pane split screen + bottom files panel. Dark theme, yellow/cyan accents, JetBrains Mono. UI language is Spanish.

Firebase SDK v10.12.2 loaded as ES modules from the Google CDN — no npm involved.
