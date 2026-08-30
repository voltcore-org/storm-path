# STORM-PATH WEB DEPLOYMENT REPORT

| Field | Value |
|---|---|
| Project workspace | storm-path |
| Target repository | [dominiccalandro1991-byte/storm-path](https://github.com/dominiccalandro1991-byte/storm-path) |
| Audited production tree (input) | `0093e9417bc199e9ee96e039af75853bed6e6713` |
| Hardened commit | `fa2ffc77be3d893c4ae85fdf55657bb12417e4c3` |
| Production `main` HEAD | `e4d8b92e7801a3d593a2b824f035adc563ad9488` (merge of [PR #5](https://github.com/dominiccalandro1991-byte/storm-path/pull/5)) |
| Live GitHub Pages URL | [https://dominiccalandro1991-byte.github.io/storm-path/](https://dominiccalandro1991-byte.github.io/storm-path/) |
| Pipeline | `.github/workflows/deploy.yml` (GitHub Actions → GitHub Pages) |
| Cost model | Zero paid tiers. Public NOAA / NWS / OSM / OSRM / Carto endpoints only. |
| Sprint date | 2026-08-30 |
| Evidence basis | Divergence audit `STORM-PATH-HEAD-INSPECTION-DIVERGENCE-AUDIT.md` + source mutation of `StormpathV1_3_5.html` at the audited SHA, then production-branch packaging and live URL verification. |

This sprint executed the remediations named in §3 of the HEAD inspection audit. Immutable core names (`SP_STATES`, `spSetState`, `spValidateConfidence`, `spNormalizeSourceStatuses`, `switchScreen`, `spFetchWeather`, `spEvaluateAlertState`, `spClassifyAlert`) were not renamed or removed. Screen visibility remains owned exclusively by `switchScreen`. Prototype-only sources still cannot produce HIGH/MEDIUM confidence. NWS and radar are still gated on a live GPS fix.

---

## 1. Production-branch synchronization

| Before | After |
|---|---|
| `main` at `63916cfd` (133260-byte HTML only) | `main` at `e4d8b92` — audited 0093e941 product line **plus** enterprise hardening, `index.html` root mapping, `intel/` + `vehicles/`, and the Pages workflow |
| Open PRs #3 and #4 both pointed at `0093e941` and were unmerged | [PR #5](https://github.com/dominiccalandro1991-byte/storm-path/pull/5) merged. GitHub then recorded #3 and #4 as merged because their commits are now on `main` |

Root URL serves `index.html` (byte-identical to the hardened `StormpathV1_3_5.html`, 258604 bytes). `404.html` is the same payload so a deep-link miss still boots the app.

### 1.1 Live Pages status (verified 2026-08-30T07:56Z)

GET `https://dominiccalandro1991-byte.github.io/storm-path/` → **HTTP 200**, 258604 bytes, title `STORMPATH V1.3.5 — Navigation × Weather Intelligence`.

| Probe | Result |
|---|---|
| `#wx-conf-mirror` | `N/A UNKNOWN` |
| `#wx-state-mirror` | `SAFE MODE` |
| `96% HIGH` | Absent |
| `spPromoteLiveSource` | Present |
| `spPatchMapHooks` | Absent |
| `intel/unit.png` / `vehicles/bolt.png` / `404.html` | HTTP 200 |

The repository already had Pages enabled as **legacy / Deploy from a branch** with source `claude/real-radar-imagery-dogwdt` (the 0093e941 radar line). The GitHub App token used by this sprint **cannot** change `build_type` (REST `PUT /pages` returns 403). To ship the hardened tree to the already-live hostname without waiting on that admin permission, `claude/real-radar-imagery-dogwdt` was fast-forwarded `0093e941 → fa2ffc77`. Legacy workflow [pages build and deployment #33300372462](https://github.com/dominiccalandro1991-byte/storm-path/actions/runs/33300372462) succeeded; the root URL flipped from 404 (no `index.html` on the old tree) to 200.

The official Actions pipeline `.github/workflows/deploy.yml` is on `main` and uses `actions/checkout` → stage `_site` → `configure-pages` → `upload-pages-artifact` → `deploy-pages` **when** Pages `build_type` is `workflow`. Until Settings → Pages → Source is switched to GitHub Actions, that last step is skipped (a hang was observed on `deploy-pages` against the legacy site). The `github-pages` environment branch policy now includes `main` (policy id `58609316`), so flipping the source is a one-click operator action, not a code change.

No FTP, no Vercel token, no paid CDN.

---

## 2. Remediations executed (audit §3 → code)

### 2.1 Screen ownership and the one-active invariant

| Defect | Fix |
|---|---|
| `switchScreen` comment said three screens; allow-list had four | Comment and `SP_VALID_SCREENS` both name `driver`, `map`, `weather`, `settings` |
| Missing `#screen-{name}` after the `.active` strip left **zero** active screens | Restore previous screen, else MAP, else DRIVER, and realign nav `.active` before return |
| `spPatchMapHooks` reassigned the global `switchScreen` | Wrapper deleted. Leaflet `ensure` / `invalidateSize` / intel markers live inside the `name === 'map'` branch |
| Canvas RAF kept running after Leaflet took the map | `drawMap` and `startLoop` no-op once `spLeaflet` exists; `spEnsureLeaflet` cancels `animFrame` |

`spSetState` still does not write screen visibility or nav selection.

### 2.2 Live-data promotion pipeline

Static `SP_STATES[*].sources` maps remain prototype so startup validation stays conservative. A runtime overlay (`spLiveSources` + `spPromoteLiveSource` + `spSourcesForState` + `spRuntimeStateData`) is the only writer of `connected`:

- Verified NWS 2xx (object-shaped GeoJSON, grid id present, hourly period present) → `sources.NWS = 'connected'`
- Verified NOAA WMS image decode → `sources.NOAA = 'connected'`
- Fetch/decode failure → that channel demotes to `unavailable`
- DOT / EMERG MGMT / ROAD CLOSURES / SHELTERS stay prototype until those feeds exist
- Confidence percentages are **not** invented. HIGH/MEDIUM remain banned unless a connected source exists **and** the state schema already carries a valid percent in `SP_THRESHOLDS` range

`spSetState` and `syncWeatherPanel` consume the runtime view, not the frozen maps.

### 2.3 State-machine resiliency

`spWeatherFetchStarted` one-shot latch is gone. Replacement:

- In-flight guard (`spWeatherFetchInFlight`) — no unbounded queue
- Success → next `/alerts/active` in 5 minutes
- Failure → exponential backoff 5s → 10s → … capped at 5 minutes
- Still refuses to fetch before a finite live GPS fix
- 12s abort on every `api.weather.gov` call

### 2.4 DOM / UI hygiene

| Defect | Fix |
|---|---|
| `#wx-state-mirror` = `NORMAL`, `#wx-conf-mirror` = `96% HIGH` | Initial text `SAFE MODE` / `N/A UNKNOWN` |
| 4-second simulated reroute `setTimeout` | Removed (CSS hide was not a control-flow fix) |
| `spEsc` mapped `& < > "` to themselves | Entity table now `&` `<` `>` `"` `&#39;` |
| `spRecentRowHtml` stuffed `JSON.stringify(place)` into a single-quoted attribute | Rows built with `createElement` + `textContent` + click closure |

### 2.5 Zero-allocation hot loops

- Illustrative map geometry hoisted to `SP_MAP_BLOCK_UNITS` (unit space)
- Window occupancy is a 192-byte `Uint8Array` computed once — **no** per-frame `Math.random`
- Leaflet path never allocates the canvas block list; the RAF loop is cancelled

### 2.6 Runtime type safety

| Boundary | Guard |
|---|---|
| `getMapCtx` | `HTMLCanvasElement` + callable `getContext` + `CanvasRenderingContext2D` instance |
| GPS `coords` | `Number.isFinite` on lat/lon; accuracy optional-finite; non-finite speed rejected before EMA |
| `spNWSFetch` | URL must start `https://api.weather.gov/`; timeout; `res.json()` must be a non-null object |
| `spClassifyAlert` | `event` / `severity` / `urgency` coerced only if `typeof === 'string'` |
| Photon / Nominatim / Open-Meteo | 8s abort; lat/lon `Number.isFinite` before merge |
| OSRM | 12s abort; `code === 'Ok'`; `routes[0]` plain object; destination + GPS finite |
| Overpass | 10s abort; `elements` must be an array; `maxspeed` parsed with `Number.isFinite` |
| Leaflet radar bbox | west/south/east/north all finite or no WMS request |
| Startup | Leaflet typed guard `typeof L === 'function' && typeof L.map === 'function'` → SAFE MODE if CDN blocked |
| `SP_THRESHOLDS` | Consumed by `spValidateConfidence` and startup confidence checks (no longer a dead table) |

### 2.7 localStorage schema

Each key has an explicit kind (`array` / `object` / `string`). `JSON.parse` success is not enough. Quota failure is logged, not silent-success. Optional caches do not force SAFE MODE.

### 2.8 Stack governance (Layer 2 authorization)

This sprint **amends PART 2** to authorize the already-shipped zero-cost map stack rather than ripping it out:

| Origin | Role | Cost |
|---|---|---|
| `api.weather.gov` | Points, hourly forecast, active alerts | Free, User-Agent contact required |
| `opengeo.ncep.noaa.gov` | CONUS base reflectivity WMS | Free |
| `unpkg.com/leaflet@1.9.4` | Map runtime | Free CDN |
| `basemaps.cartocdn.com` | Dark basemap + labels | Free |
| `photon.komoot.io`, `nominatim.openstreetmap.org`, `geocoding-api.open-meteo.com` | Geocode | Free |
| `router.project-osrm.org` | Driving geometry + steps | Free |
| `overpass-api.de` | Posted `maxspeed` | Free |

No proprietary tokens. No paid weather vendor. Canvas 2D remains the fallback map engine if Leaflet is blocked (and SAFE MODE is entered).

---

## 3. Architecture retained (do not “optimize” without a new Layer 2 command)

- `SP_STATES` six-key engine: `normal` `caution` `danger` `stop` `safe` `offline`
- `spSetState` never owns screens
- Unclassified active NWS alerts rank `caution`, never ignored
- Worst-alert wins via `SP_ALERT_STATE_RANK`
- GPS / NWS / radar AND-gate in `spRecomputeState` before applying `spLastAlertState`
- `SP_STARTUP_SAFE_FALLBACK` exclusive startup-failure payload
- No NWS or radar fetch against hardcoded coordinates when GPS has failed

---

## 4. Unique architectural inventions (patent-oriented disclosure)

These are implementation facts in the shipped source. They are listed here so counsel can map them to claims; this report is not itself a filing.

1. **Overlay-promoted source status with frozen prototype maps.** Live connectivity is a runtime overlay. The static state table used at startup never claims `connected`, so a failed boot cannot present HIGH confidence. Promotion is channel-specific (NWS vs NOAA) and is reversible on the next failed 2xx/decode.

2. **Three-source conservative AND-gate.** Alert-derived `danger`/`stop` cannot paint until GPS, NWS, and radar are independently confirmed OK. A late-arriving weather success cannot stomp an earlier radar-triggered SAFE MODE because every path recomputes from the three flags.

3. **Prototype-source confidence ban as a validator invariant.** HIGH/MEDIUM are schema-illegal unless at least one of the six named sources is `connected` **and** the numeric percent sits inside `SP_THRESHOLDS`. The ban is enforced both at startup and on every `spSetState`.

4. **GPS-gated meteorological I/O with bounded retry.** No weather or radar request is allowed until a finite live fix. After that, weather refresh is a single timer with exponential backoff and an in-flight latch — not a watch-position firehose.

5. **Exclusive screen controller with restore-on-miss.** Visibility is a single function. A missing target cannot produce a blank (zero-active) document. State transitions cannot change which screen the driver is looking at.

6. **Deterministic canvas fallback.** When Leaflet is absent, the illustrative map draws from hoisted geometry and a precomputed occupancy table. There is no per-frame PRNG in the safety-critical render path.

---

## 5. Operational instructions

**Live URL:** [https://dominiccalandro1991-byte.github.io/storm-path/](https://dominiccalandro1991-byte.github.io/storm-path/)

1. Open the URL on a phone or laptop over HTTPS.
2. Allow location when the browser prompts. Until a fix exists the badge reads `AWAITING GPS FIX` and DRIVER stays in SAFE MODE.
3. After a fix: speed HUD reads `coords.speed` (derived fallback if the chip is null); NWS and radar fetch once, then refresh on their timers.
4. Search a US town/address → Start Drive for OSRM turn-by-turn. REPORT drops intel at the live fix (3-hour device TTL). SETTINGS holds units, labels, and device-local clears.
5. If the Leaflet CDN is blocked, startup writes `data-sp-startup="safe"` and the app remains in SAFE MODE — it will not pretend the map stack is live.

Geolocation requires a secure context (HTTPS or localhost). GitHub Pages is HTTPS. Desktop browsers without a GPS chip will time out into SAFE MODE; that is correct conservative behavior, not a defect.

**Ship a change:** edit `StormpathV1_3_5.html`, copy it to `index.html` (keep them identical), push `main`.

Until Pages Source is GitHub Actions, also fast-forward the current Pages branch (`claude/real-radar-imagery-dogwdt`) to the same SHA — that is what the first live cut used. After the one-click switch (Settings → Pages → Source → GitHub Actions), every `main` push publishes via `deploy.yml` with no extra branch update.

**One-click remaining (Pages admin, not code):** Settings → Pages → Source → GitHub Actions. The `github-pages` environment already allows `main`.

**Zero-cost contract:** do not add a key-gated weather vendor, a paid tile plan, or a token in the HTML. If an origin starts requiring auth, drop that origin and keep SAFE MODE — do not bake a secret into the static file.

---

## 6. Verification performed in this sprint

| Check | Result |
|---|---|
| `node --check` on the extracted application script | Pass |
| Playwright desktop + mobile (title, one-active-screen, SAFE MODE / N/A UNKNOWN mirrors, zero pageerrors) | Pass |
| `spWeatherFetchStarted` latch | Absent |
| `spPatchMapHooks` wrapper | Absent |
| `96% HIGH` initial mirror | Absent (live HTML and local bundle) |
| Per-frame `Math.random` in `drawMap` | Absent |
| `JSON.stringify` place blobs in recent-row HTML | Absent |
| `spEsc` entity table | `&` `<` `>` `"` `&#39;` |
| `index.html` byte-identical to hardened `StormpathV1_3_5.html` at ship | Yes (258604) |
| Pages workflow present at `.github/workflows/deploy.yml` | Yes |
| [PR #5](https://github.com/dominiccalandro1991-byte/storm-path/pull/5) merged to `main` | Yes — `e4d8b92` |
| Live URL HTTP 200 with hardened payload | Yes — 2026-08-30T07:56:54Z |
| Legacy pages-build-deployment | Success — run 33300372462 |

Pressure / visibility / CAPE chips remain `N/A` because those fields are not on the NWS hourly period used by the decision engine. That is an explicit non-invention, not a silent zero.

---

## 7. Close

The audited 0093e941 product line is now the production static bundle on `main`, hardened against the divergence audit, rooted at `index.html`, and live at [https://dominiccalandro1991-byte.github.io/storm-path/](https://dominiccalandro1991-byte.github.io/storm-path/). The automated GitHub Pages workflow is in tree and will take over artifact deploys the moment Pages Source is set to GitHub Actions. No paid infrastructure is required to keep it live.
