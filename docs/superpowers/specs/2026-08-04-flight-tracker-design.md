# Fun & Simple Flight Tracker — Design

**Date:** 2026-08-04
**Status:** Approved

## Goal

Make waiting for a relative's or friend's flight fun. Enter a flight number,
watch the plane live on a map with a playful progress bar, get confetti when
it lands. One flight at a time, nothing saved, no accounts, no API keys.

## Form

A single self-contained `flight-tracker.html` file. Open it in any browser
(double-click, or serve it). External dependencies loaded from CDN at runtime:

- **Leaflet 1.9** (map library) + **OpenStreetMap** tiles — free, keyless
- **airplanes.live** open API — live aircraft positions (free, keyless, CORS `*`)
- **adsbdb.com** open API — callsign → route/airports (free, keyless, CORS `*`)

(adsb.lol was the original choice but its v2 API sends no CORS headers and its
routeset endpoint currently returns empty responses — verified 2026-08-04.)

## Flow

1. User types a flight number (`UA123`, `BA42`, or a raw callsign like `UAL123`)
   and presses **Track** (or Enter).
2. Input is uppercased and expanded into candidate ICAO callsigns via a small
   embedded table (~60 common airlines: `UA`→`UAL`, `BA`→`BAW`, `EK`→`UAE`, …).
   Leading zeros in the flight number are also tried stripped (`UA0123`→`UAL123`).
3. `GET https://api.airplanes.live/v2/callsign/{candidate}` for each candidate
   until an airborne aircraft is found.
4. `GET https://api.adsbdb.com/v0/callsign/{callsign}` returns the route's
   origin/destination airports with coordinates and names.
5. Poll position every 15 seconds and update the display.

## Screen

- **Header:** playful title + flight number input + Track button.
- **Map (Leaflet):** plane icon (SVG, rotated to actual heading) and
  origin/destination airport markers; auto-fits the route area on first load.
  (A dashed route line existed originally; removed 2026-08-04 by user request.)
- **One-screen layout:** the page is a flex column sized to the viewport — the
  map flexes to fill leftover space, the where-are-they pill overlays the map's
  bottom edge, and the landed banner floats above the page, so the normal
  tracking view never scrolls.
- **Fun strip:** cartoon progress bar — `SFO ✈️ ─────── 🛬 LHR` with the plane
  emoji sliding along; "62% there · about 3h 40m to go" (remaining great-circle
  distance ÷ current ground speed); live altitude and speed readouts.

## Landing celebration

When the transponder reports **on ground** within ~50 km of the destination
(or the flight disappears from the API while low and close to the destination),
show a "They've landed! 🎉" banner and fire hand-rolled canvas confetti.
On-ground at the *origin* (taxiing before takeoff) does not trigger it.
Polling stops after landing.

## Living sky & where-are-they (added 2026-08-04, approved)

- **Living sky:** emoji clouds drifting at three speeds behind the content, a
  bobbing ✈️ in the title, a sun/moon orb top-right, and a palette that follows
  the viewer's local clock — day (blue), dusk (violet/peach, 17:00–21:00 and
  05:00–07:00), night (navy + twinkling star field, 21:00–05:00). Pure CSS
  animation; `prefers-reduced-motion` disables all of it. Display font: Fredoka
  (Google Fonts CDN) for the title and fun line.
- **Where-are-they pill:** each refresh (only when the plane has moved > 25 km),
  the plane's coordinates go to BigDataCloud's keyless reverse-geocode API
  (CORS `*`); a pill overlaid on the map shows "🌍 Currently over Wales, United
  Kingdom" or "🌊 Somewhere over the Atlantic Ocean". Lookup failure hides the
  pill; tracking is unaffected.

## Error handling

- **No aircraft found:** "I can't see this flight in the sky right now — it may
  not have taken off yet (or already landed)."
- **Route unknown** (some flights): still show the live plane on the map; hide
  the progress bar.
- **Network hiccup:** keep last known position, quietly retry on next poll.

## Testing

Pure logic (callsign candidate expansion, haversine distance, progress %, ETA
formatting) lives in a clearly delimited block and has built-in self-tests:
open `flight-tracker.html?test` to see pass/fail results in the page. The same
block is extractable for a quick Node.js check. Live behaviour is verified by
tracking a real flight.

## Out of scope (YAGNI)

Multiple flights, saved history, gate/delay info, notifications, API keys.
