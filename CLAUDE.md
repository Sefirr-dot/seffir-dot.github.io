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

Required GitHub Actions secrets: `FIREBASE_API_KEY`, `FIREBASE_AUTH_DOMAIN`, `FIREBASE_DATABASE_URL`, `FIREBASE_PROJECT_ID`, `FIREBASE_STORAGE_BUCKET`, `PASSWORD_HASH`.

There is no `config.js` in production — secrets live inside `index.html` after the build step. `config.js.example` is just a reference listing the required values.

## Architecture

All code is in `index.html`. Key sections:

- **Password gate**: SHA-256 hash via Web Crypto API. `PASSWORD_HASH` is injected at build time.
- **Paste sync**: Firebase Realtime Database paths `paste/ordenador1` and `paste/ordenador2`. `onValue()` listeners for reads; debounced (500ms) `set()` writes. A `syncing` flag per pane prevents write-loops.
- **File hosting**: Firebase Storage under `files/{uuid}/{filename}`. Metadata (name, size, path, url, uploadedAt, expiresAt) stored in Realtime Database under `files/`. TTL is 10 minutes, max 100 MB, max 1 file active at a time. Cleanup runs on every app load via the `onValue` listener — expired entries are deleted from both Storage and DB. Download URLs are constructed directly (`firebasestorage.googleapis.com/v0/b/...`) instead of calling `getDownloadURL` to avoid auth issues.
- **UI**: Dual-pane split screen + bottom files panel. Dark theme, yellow/cyan accents, JetBrains Mono. UI language is Spanish.

Firebase SDK v10.12.2 loaded as ES modules from the Google CDN — no npm involved.
