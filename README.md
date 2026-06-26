# Tamar Light

A light-meter for plein air painting and photography in the Tamar Valley, Tasmania. It answers one question — *is the light worth chasing, and when* — by combining locally-computed sun/moon astronomy with cloud-cover forecasts.

It is **not** a weather app and is built to stay quiet: if a change makes it quieter, it's right; if it makes it busier, it's wrong.

## Architecture

- **Single HTML file** (`tamar-light.html`) — no build step, no bundler, no npm.
- **No backend, no API keys.** Sun/moon position is computed client-side (a trimmed [SunCalc](https://github.com/mourner/suncalc) port, BSD-licensed). The only network call is to [Open-Meteo](https://open-meteo.com/) (no key required).
- **localStorage only**, offline-first. Sun/moon math renders with zero network; weather degrades gracefully to last cache.

## Running

Open `tamar-light.html` in any modern browser, or visit the GitHub Pages site. `index.html` is a redirect to the canonical app file.

## Status

v2 — shipped and working. Current focus is fine-tuning. See [`tamar-light-build-brief.md`](tamar-light-build-brief.md) for the full feature inventory, tunable knobs, and known gaps.

---

Signal9 Apps · Kirk
