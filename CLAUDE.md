# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PesanHaloCoko is a single-page PWA (Progressive Web App) deployed on Netlify. It serves as an unofficial order manager for ice cream product agents/stores, allowing them to compile orders and generate WhatsApp-ready formatted text.

## Tech Stack

- **No build system** — everything is vanilla HTML/CSS/JS served statically
- **Alpine.js 3.x** (CDN) — reactive state management and UI (via `Alpine.data()`)
- **Tailwind CSS** (CDN) — all styling
- **Service Worker** (`sw.js`) — offline caching via Cache API
- **Netlify** — hosting platform

## Architecture

The entire application lives in **`index.html`** (~525 lines). There is no framework, no router, and no separate JS/CSS files:

- **`<style>` block** — custom CSS for `.app-canvas` (mobile-first 480px max-width container), dialog animations, and scrollbar utilities
- **`<script>` block** — Alpine.js app component (`app()`) containing all business logic:
  - `PRICE_MAP` — maps product IDs to pieces-per-box and price-per-box
  - `initialDB` — product catalog (23 items with name, unit price, and initial qty)
  - Alpine component state: `currentTab` (order/history), `store` (name/address/date), `items` (catalog with qtys), `result` (generated WhatsApp text), `history` (saved orders)
  - `localStorage` keys: `hc_app_v5_final` (draft state), `hc_history` (order history, max 50 entries)
  - Minimum 6 dus (boxes) required before checkout button enables

- **`sw.js`** — Service worker caching `index.html`, `manifest.json`, and icon images for offline use
- **`manifest.json`** — PWA manifest (display: standalone, portrait-only)

## No Build/Dev Server

This project has no build step, package manager, or dev server. To preview locally, serve the root directory with any static file server:

```bash
npx serve .        # or: python3 -m http.server 8000
```

There are no tests and no linting configuration.

## Deployment

Deployed via Netlify. The site is a static root with no build command — Netlify simply serves the files as-is from the repository root.
