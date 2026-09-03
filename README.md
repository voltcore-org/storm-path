# Storm Path

Storm-aware navigation: live device GPS, National Weather Service alerts, and NOAA base-reflectivity radar. Static single-file web app. Zero paid APIs.

## Live site

https://voltcore-org.github.io/storm-path/

## What it does

- Opens on the **map**. Default view is Murphysboro, Illinois — **view only**.
- DRIVER, MAP, WEATHER, and SETTINGS — only one screen is active at a time.
- Device GPS is the only LIVE location. Residential IP may recenter the map as VIEW. IP is never GPS.
- After the first live GPS fix, fetches NWS `/points`, hourly forecast, and `/alerts/active`, plus NOAA OpenGeo WMS radar. Those three together are the AND-gate for HIGH / LIVE driver state.
- VIEW weather/radar can paint Murphysboro (or a network view) without flipping the AND-gate.
- Alert ranking is conservative: empty list → NORMAL; unclassified active alert → CAUTION; tornado / extreme → DANGER or STOP.
- Prototype-only sources cannot produce HIGH or MEDIUM confidence.
- Destination search (Photon / Nominatim / Open-Meteo), OSRM routing, Overpass speed limits, OpenStreetMap tiles — public, unauthenticated endpoints. No Carto. No paid weather vendor.

This is **not** a life-safety system. NWS/NOAA only. Never treat a map view or stale/IP location as live GPS.

## Run locally

Serve the repo root over HTTPS or `localhost` (Geolocation requires a secure context):

```
python3 -m http.server 8080
```

Open `/` (serves `index.html`). Assets live in `intel/` and `vehicles/`.

## Verify

```
node scripts/run-vector-score.mjs
```

Engine checks are real (AND-gate, Murphysboro view default, OSM, GPS API). Runtime checks hit live NWS, NOAA WMS, and OSM tiles. A PASS file that only greps strings is not enough — this runner fails if the live endpoints do not answer.

## Deploy

Push to `main`. `.github/workflows/deploy.yml` publishes the static tree to GitHub Pages. No tokens in the tree, no build step, no paid hosting.

## Files

| Path | Role |
|---|---|
| `index.html` | GitHub Pages root |
| `StormpathV1_3_5.html` | Canonical application source (same payload as `index.html`) |
| `404.html` | Same payload so a deep-link miss still boots the app |
| `intel/` `vehicles/` | Marker artwork |
| `scripts/run-vector-score.mjs` | Honest engine + runtime vector score |
| `.github/workflows/deploy.yml` | Pages pipeline |
| `.github/workflows/vector.yml` | Vector score on every push |

User-Agent for `api.weather.gov`: identify the app and a contact mailbox. No API keys.
