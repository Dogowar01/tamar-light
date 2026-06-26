# Tamar Light — Claude Code Build Brief

**App:** Tamar Light (Signal9 Apps)
**File:** single-file HTML PWA — `tamar-light.html` (~650 lines, no build step)
**Status:** v2 shipped and working. This brief is for fine-tuning, not a from-scratch build.
**Owner:** Kirk — Signal9 Studio (fine art print practice) + Signal9 Apps

---

## 1. What this app is

A light-meter for plein air painting and photography in the Tamar Valley, Tasmania. It answers one question — *is the light worth chasing, and when* — by combining locally-computed sun/moon astronomy with cloud-cover forecasts.

It is **not** a weather app and should never grow into one. The design discipline throughout has been: if a change makes it quieter, it's right; if it makes it busier, it's wrong. Any fine-tuning should preserve that.

## 2. Architecture (do not change without reason)

- **Single HTML file.** No build step, no bundler, no npm. Everything — HTML, CSS, JS — lives in one file, deployed as-is to GitHub Pages.
- **No backend, no API keys.** Sun/moon position is computed entirely client-side (a trimmed SunCalc port, BSD-licensed, credited in code comments — see `astronomy` section). The only network call is to **Open-Meteo** (`api.open-meteo.com/v1/forecast`), which requires no key.
- **localStorage only**, four keys:
  - `tamarlight.place` — active preset/spot key (`"lat.lng"` string)
  - `tamarlight.wx.<lat>_<lng>` — cached weather response, keyed per location, 2-hour freshness window
  - `tamarlight.spots` — array of user-saved custom locations
  - `tamarlight.ridge` — map of `{ "lat,lng": {e, w} }` horizon-blocking angles per location
- **Offline-first.** Sun/moon math and the meridian render with zero network. Weather degrades gracefully to last cache with a "stale" indicator; if there's no cache at all, conditions-dependent UI (verdict, fog line, conditions row) just says so rather than guessing.
- **No service worker yet.** The page works offline once loaded in a tab, but isn't installable for guaranteed offline-first launch. Flagged as a known gap in §6.

## 3. File map (where to find things)

| Lines (approx) | Section |
|---|---|
| 1–186 | `<head>` + all CSS (custom properties for the whole colour system live in `:root`) |
| 187–230 | Body markup |
| 191–235 | Astronomy core — SunCalc port: `sunPos`, `sunTimes`, `riseSetForAngle`, `moonCoords`, `moonIllum`, `moonName` |
| 237–258 | Config/state — `PRESETS` array, localStorage keys, `placeNow()`, `ridgeOf()` |
| 261–296 | Light-quality logic — `phaseOf()`, `phaseLabel()`, `painterRead()`, `shadow()` |
| 298–347 | Weather — `fetchWx()`, `fogFor()`, `trueDark()` |
| 348–359 | Moon glyph SVG renderer |
| 360–428 | Meridian spine + events + golden-hour `verdictFor()` |
| ~430–520 | Main `render()` and all sub-renderers (`renderShadow`, `renderCond`, `renderFog`, `renderDark`, `renderPlanner`) |
| ~520–600 | Location sheet — presets, ridge stepper, GPS, save/delete custom spots |
| ~600–650 | Share card (canvas-drawn PNG) + boot/wiring |

## 4. Feature inventory (all working, all validated)

1. **Light meridian** — the day read top-to-bottom as a true-colour spine (twilight → blue → golden → day → golden → blue → twilight), with event ticks and a live "now" line.
2. **Current phase + painter's read** — names the light plainly, gives time remaining in the current phase, and a one-line qualitative read combining sun angle *and* cloud (e.g. "Warm, low, raking light... the hour worth chasing").
3. **Shadow read** — direction (16-point compass) and length multiple, computed from sun azimuth/altitude. Hidden when the sun's below the horizon.
4. **Golden-hour verdict** — picks the next upcoming golden window (morning or evening), averages forecast cloud across it, and gives a plain verdict + set-up time.
5. **7-day golden planner** — replaced the old bare day-stepper. Seven dots, one per evening, coloured by forecast clarity; doubles as the day-selector for the whole app.
6. **Valley fog flag** — heuristic over humidity/dew-point spread/wind/cloud (with visibility as a stronger signal when present), scanning 04:00–10:00 for the selected day. Silent when there's no signal.
7. **True-dark window** — intersects astronomical-twilight boundaries with moon-below-horizon, for nocturne/astro use. Silent (well, shows "moon up all night") when there's no real dark window.
8. **Per-spot ridge horizon** — user sets an east/west blocking angle (0–20°) per saved location in the sheet; this shifts the *displayed* sunrise/sunset event (labelled "Clears E ridge" / "Hits W ridge") and shades a hatched band on the meridian. Does not change the underlying astronomical sunrise — only the visible-from-here event.
9. **GPS + saved spots** — "Use my current location" for a one-off reading (not persisted), or save a named spot with manually entered lat/long. Six hardcoded presets (Grindelwald, Rosevears, Cataract Gorge, Beauty Point, Low Head, Narawntapu) cannot be deleted; custom spots can.
10. **Shareable light card** — renders today's meridian + phase + verdict to a 1080×1350 PNG via canvas, hands off to the Web Share API or falls back to direct download.

## 5. What's already been validated (so Claude Code doesn't need to re-derive it)

- Astronomy was checked against expected late-June Hobart-latitude values (sunrise ~07:40, sunset ~16:51, solar noon ~12:16, peak altitude ~25°) — correct.
- Shadow geometry checked at noon (due south, short) and near sunrise/sunset (WSW/SE, ×20–34 length) — correct.
- Ridge-adjusted rise/set checked against an 8°/6° horizon — correctly shifts displayed events by ~1 hour without altering true astronomical times.
- True-dark sampling checked near full moon (correctly returns a near-zero or absent dark window) — correct.
- Fog heuristic checked against a synthetic damp/still morning (fires) vs. a clear morning (silent) — correct.
- Full render pipeline (boot, phase, shadow, verdict, conditions, fog, true-dark, planner, ridge-adjusted events, save-spot, GPS, day-stepping, share-card canvas draw) was run end-to-end in a scripted Node harness with a stubbed DOM and mocked Open-Meteo response. All paths executed without error and produced correct output.
- No browser/Puppeteer was available in the build environment, so **the actual rendered visual has never been seen** — only logic-verified. This is the single biggest thing worth Claude Code's attention: open it in a real browser first and sanity-check spacing, contrast, and the sky-tint transition before touching logic.

## 6. Known gaps / deliberately deferred

- **No service worker.** Single-file PWAs in this stack normally get a minimal versioned SW for guaranteed offline launch. Not done yet — flagged, not forgotten.
- **No tides.** Every tide data source for Low Head needs a paid key or an annual harmonic-constituent download (WorldTides, Stormglass, AusTides). Deliberately left out of v1/v2 to avoid putting a backend behind something that should stay frictionless. The clean path, when wanted: embed Low Head's published harmonic constituents and compute tide height locally the same way sun/moon are computed — not a live feed.
- **Never seen in a real browser.** See §5 — visual QA pass is the first thing to do, not logic changes.
- **Open-Meteo cloud-cover-only fog heuristic** is a best-effort flag, not a guarantee, same caveat as any forecast.

## 7. Tunable knobs (the actual point of this brief)

Everything below is a deliberate constant sitting in plain variables — safe to adjust without touching surrounding logic. Locations given as function/variable names, not line numbers (lines will drift).

**Light-phase boundaries** (`phaseOf()`) — sun-altitude cutoffs in degrees:
`night < -18`, `astro < -12`, `naut < -6`, `blue < -4`, `golden < 6`, else `day`. These match standard astronomical/nautical/civil twilight + a ±6° "golden hour" convention. Adjust the `6` if golden hour should run longer/shorter.

**Verdict cloud thresholds** (`verdictFor()`): `< 25` = clear, `< 60` = partly, else cloudy. Same thresholds reused in `eveningClarity()` for the 7-day planner dots — keep them in sync if changed.

**Painter's-read cloud bands** (`painterRead()`): `< 15` clear, `< 50` part, `< 85` soft, else over. Independent of the verdict thresholds above — currently a finer 4-band split for the prose description vs. the coarser 3-band verdict. Could be unified if that reads as more consistent.

**Fog heuristic** (`fogFor()`): triggers on `visibility < 1500m` OR (`humidity >= 93%` AND `temp − dewpoint <= 1.5°C` AND `wind < 9km/h` AND `cloud < 60%`), scanned only across hours 04:00–10:00. All five numbers are guesses tuned to "Tasmanian river valley morning fog," not measured against real Tamar fog events — worth calibrating against Kirk's own observed mornings if the flag fires wrong.

**Ridge angle range** (`stepper` in the location sheet): clamped 0–20°. Should be plenty for valley terrain; raise if a spot genuinely needs more.

**Weather cache freshness**: 2 hours (`fetchWx()`). Open-Meteo allows far more frequent calls — could shorten if Kirk wants tighter accuracy near a session, at the cost of more network calls.

**Colour system** — every colour is a CSS custom property in `:root` (`--night`, `--astro`, `--naut`, `--civil`, `--blue`, `--gold`, `--day`, `--day-pale`, etc.) plus the JS-side `tintMap` object in `render()` and `HEX`/`tintM` objects duplicated in the share-card canvas drawing code. **If any colour changes, it must be changed in three places**: the CSS variable, the JS `tintMap` (background tint), and the canvas `HEX`/`tintM` constants (share card). This duplication is the one piece of technical debt in the file — a refactor to a single shared colour source (e.g. read CSS custom properties at runtime instead of hardcoding hex twice) would be a reasonable cleanup if Claude Code has time.

**Six hardcoded presets** (`PRESETS` array) — straightforward to add/remove/rename Tamar Valley reference points here.

## 8. Things to leave alone unless specifically asked

- The single-file structure. Don't split into modules/bundles.
- The localStorage-only persistence. Don't add a backend.
- The "quiet" rule: no new always-visible UI chrome. New features should default to hidden/silent and only surface when genuinely true (matches the fog/true-dark pattern already established).
- The Signal9 aesthetic: serif display type (Fraunces) for phase/headline text, sans-serif for everything functional, mono for all timestamps, restrained colour, generous negative space, no drop shadows beyond the very subtle ones already present.

## 9. Suggested first session

1. Open in an actual browser (mobile width first — this is iPhone-companion software) and just look at it. Confirm spacing, the sky-tint transition, the meridian proportions, the share-card output actually rendering as a real PNG.
2. Calibrate the fog heuristic against a couple of real Tamar Valley mornings if Kirk has observations to compare against.
3. Decide whether to unify the verdict vs. painter's-read cloud-band thresholds (§7) or leave them deliberately distinct.
4. Pick up the service worker (quick) or tides (bigger, needs harmonic data decision) as the next real feature, per Kirk's priority.
