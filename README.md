# Storm Path

Storm-aware navigation: live device GPS, National Weather Service alerts, and NOAA base-reflectivity radar. Static single-file web app. Zero paid APIs.

## Live site

https://dominiccalandro1991-byte.github.io/storm-path/

## What it does

- Opens on the live map with a GPS speedometer. The marker moves only from the device fix.
- DRIVER, MAP, WEATHER, and SETTINGS screens — only one is active at a time.
- After the first live GPS fix, fetches NWS `/points`, hourly forecast, and `/alerts/active`, plus NOAA OpenGeo WMS radar.
- Alert ranking is conservative: empty list → NORMAL; unclassified active alert → CAUTION; tornado / extreme → DANGER or STOP.
- Prototype-only sources cannot produce HIGH or MEDIUM confidence. Live NWS/NOAA success promotes those two chips to `connected` without inventing HIGH percentages.
- Destination search (Photon / Nominatim / Open-Meteo), OSRM routing, Overpass speed limits, Carto tiles — all public, unauthenticated endpoints.

## Run locally

Serve the repo root over HTTPS or `localhost` (Geolocation requires a secure context):

```
python3 -m http.server 8080
```

Open `/` (serves `index.html`). Assets live in `intel/` and `vehicles/`.

## Deploy

Push to `main`. `.github/workflows/deploy.yml` publishes the static tree to GitHub Pages. No tokens, no build step, no paid hosting.

## Files

| Path | Role |
|---|---|
| `index.html` | GitHub Pages root (hardened production bundle) |
| `StormpathV1_3_5.html` | Canonical application source (same payload as `index.html`) |
| `intel/` `vehicles/` | Marker artwork |
| `.github/workflows/deploy.yml` | Pages pipeline |

User-Agent for `api.weather.gov`: identify the app and a contact mailbox. No API keys.
