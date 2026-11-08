# Vertical profile tool — project notes

Read this first. It replaces the earlier HANDOVER.md and covers both chats:
the original build (v1–v13, plus the RASP side-project) and the follow-up
(v14–v16: best_match fix, Windy research, route plotting, mobile).

Personal planning tool for mountaineering in the Western Cape. Time–height
cross-sections from ground to ~2500 m for peaks like Table Mountain (1086 m),
Du Toit's Peak (1911 m, −33.753698, 19.189022) and Matroosberg (2249 m).

`index.html` is v16 — one self-contained page, plain JS, no build step,
Leaflet from CDN for the map picker. Hosted on GitHub Pages. Data from
Open-Meteo. Temperatures in °C throughout.

---

## Open items

1. **Check v16 on a real phone.** Mobile layout (controls collapse below
   760 px) and tap-to-probe readout were never tested on a device.
2. **Add the route caveat to the note text.** The route line is the
   free-atmosphere column at that height, not conditions on the ridge. A line
   invites more trust than a fill does — say so near the route summary.
3. **Favourites have no delete or rename.** Only route today is
   `localStorage.removeItem('vp_favs')` in the console. Add a remove control.
4. **Shared locations file** — agreed design, not built: `locations.json` in
   the Pages folder, read at startup into a "Shared" optgroup (store
   elevation in the file); personal saves stay in localStorage under
   "My saved"; an export button dumps personal saves as JSON to commit into
   the shared file. Pages is read-only — never put a GitHub token in the page.
5. **ECMWF default trap.** The model dropdown shows levels-in-band once a
   model has been loaded. On Open-Meteo ECMWF has only 3 levels below 2000 m,
   so don't make it the default. ICON/GFS are better *here* — licensing, not
   skill.
6. **3-hourly resample test** before committing to a Windy port: resample the
   Open-Meteo grid to 3-hourly and see whether the contours are still usable.
7. **RASP — discussion only, no code yet.** See the RASP section. Preferred
   use if ever built: boundary-layer top as five markers a day drawn on top of
   the panels, never merged into the gridded fields.
8. Rain along the route is hourly rate sampled every 10 min and scaled — an
   estimate, labelled "about".

---

## Decisions — don't relitigate

- **Open-Meteo** is the data source for the hosted page (free, no key, CORS
  open). Windy Point Forecast API (€990/yr, no ECMWF), Windy Premium (display
  only, no API) and meteoblue (profile data from €2400/yr) were rejected.
- **WindGuru colours and line conventions**, taken from its client. Wind
  stops (knots, held in m/s): 0 & 5 white, 8.9 rgb(103,247,241),
  13.5 rgb(0,255,0), 18.8 rgb(255,240,0), 24.7 rgb(255,50,44),
  31.7 rgb(255,10,200), 38 rgb(255,0,255), 45 rgb(150,50,255),
  60 rgb(60,60,255), 70 rgb(0,0,255). Temperature: −25 rgb(80,255,220),
  −15 rgb(171,190,255), 0 white, 10 rgb(255,255,100), 20 rgb(255,170,0),
  30 rgb(255,50,50), 35 rgb(255,0,110), 40 rgb(255,0,160), 50 rgb(255,80,220).
  Gridlines dashed 2,2 black at 10% / 30%; day boundaries solid 55%. Night
  `#000088` at 5%. Day bands `#d5d5d5` / `#f4f4f4`. Open Sans; labels black
  at varying alpha, never grey.
- **Contours**: isotherms every 1 °C, isotachs every 5 km/h.
- **Cloud shading from Meteoblue**: banded (`#d2d2d2 #b1b1b1 #929292 #737373
  #565656`, nothing below 10%) or smooth (same colours, band colours at range
  midpoints). Cloud field box-blurred; wind and temperature are not.
- **Rain like Windy**: `rgb(0,64,255)` bars in the bottom fifth of the cloud
  panel, **fixed 8 mm/h ceiling** (auto-scaling made 0.1 mm look like a storm).
- **Wind arrows**: WindGuru path
  `M50,0 L30,30 L46,27 L43,90 L57,90 L54,27 L70,30 Z`, rotated direction + 180,
  alpha fades below ~4 km/h.
- **Apparent temperature: Steadman**, not JAG/TI (undefined above 10 °C and in
  light wind). Shade and sun variants; sun term `0.70Q/(ws+10)`, Q reduced by
  cloud **above the cursor only**.
- **Cloud base** — Open-Meteo `cloud_base` is always null in South Africa, so
  it's derived as the lowest level at ≥ `CLOUD_BASE_PCT` cloud, now **40**.
- Mid cloud is back in the strip under high cloud (v15): on a 1500–2000 m
  axis it's usually off the top of the panel, not redundant.
- Default axis top is 2000 m; days selectable 2–7.

---

## Code map (v16)

- **The seam**: `fetchGrid()` returns one flat object (`times, nLevels,
  levels, heights, wind, temp, dir, cloud, rh, precip, rain, snow, prob,
  cloudBase, cloudMid, cloudHigh, toa, elevation, terrainMin, cellLat,
  cellLon, n, nights, …`). Everything downstream reaches it only through
  `columnAt()` and `atHeight()`. A new data source = a second producer of
  this object.
- **Zoom** goes through `tAtPx` / `xAtT` / `hAtPy` / `yAtH`. New drawing must
  route through these — no special cases.
- Height gridlines and axis labels share `heightSteps()` (they drifted once).
- `LEVELS` requests 1000–200 hPa; levels whose `geopotential_height` is all
  null are dropped (`kept`), so ECMWF's missing levels don't leave holes.
- `identifyBestMatch()` — **verified**: multi-model responses are one object
  with suffixed keys (`temperature_850hPa_best_match`); exact equality works;
  globals are tested before seamless variants (identical over SA). Over Cape
  Town `best_match` → `icon_global` at pressure levels.
- `annotateModelOption()` writes "N lvl ≤X m" into the model dropdown after a
  load. `labelPresetElevations()` adds elevations to preset places (one
  batched call).
- **Route**: `buildRoute()` → four `{t,h}` points (trailhead, summit,
  summit + buffer, trailhead). Summit = point elevation; trailhead = manual
  field or `terrainMin`. `routeLine()` draws on all three panels inside the
  clip (white halo, dark line). `sampleRoute()` walks every 10 min through
  `columnAt`/`atHeight` and Steadman. `drawRouteChart()` adds a block below
  the panels (cloud ribbon, temp + feels on the left axis, wind on the
  right); `renderRouteSummary()` writes the text line. Route inputs redraw,
  never refetch.
- **Readout**: `readoutAt(x,y)`. Mouse = hover. Touch = a tap places a sticky
  crosshair (`probe`) on the overlay canvas; touch-drag still zooms.
- **Mobile**: below 760 px `.controls` hide behind `#ctlToggle`.
- **Favourites**: localStorage key `vp_favs`, array of `{name,lat,lon,elev}`;
  name is the dedupe key; `memFavs` fallback when storage throws (e.g.
  `file://`).

### Verification that actually catches bugs
`node --check` only parses. Also **execute the script against a stubbed DOM**
(stub `document`, `window`, `localStorage`, `fetch`, `Path2D`,
`ResizeObserver`) — that's how a stale `DAYS` reference was finally caught.
For maths, export the pure functions and run them against a synthetic grid
(done for the route: 340 → 1911 m gave 17.8 → 7.6 °C on a 6.5 °C/km lapse).

---

## Verified facts — Open-Meteo

- Cells over Cape Town: best_match 7.8 km, UKMO 10.4, GFS 13, CMA 13.9,
  GEM 16.7, ICON / ECMWF / ARPEGE 27.8.
- **Pressure-level fields are constant across a whole cell**; only surface
  variables get elevation downscaling (Table Mountain summit vs 1 km north:
  2 m temp 7.8 vs 11.8 °C, 850 hPa 16.1 vs 16.1 °C). The status line reports
  the cell so an unchanged chart doesn't look like a failed refresh.
- **ECMWF open data is a level subset**: 1000, 925, 850, 700, 600, 500… —
  975/950/900/800/750 are null. Three levels in 0–2000 m. Open-Meteo's 9 km
  IFS HRES has no pressure levels; those come from the 0.25° product.
- **ECMWF is natively 3-hourly** on Open-Meteo (hourly values interpolated);
  ICON and GFS are natively hourly.
- All major models run 4×/day. Lag after 12Z ≈ ICON 3.8 h, ARPEGE 4.0,
  GFS 5.8, UKMO 7.5, ECMWF 7.8. Some Open-Meteo models are stale (BOM ACCESS
  from 2025; ecmwf_ifs04 / aifs025 Feb 2025) — check
  `/data/{model}/static/meta.json` → `last_run_initialisation_time`.
- Multi-location requests take comma-separated coords; 9 points cost 282 KB /
  191 ms vs 31 KB / 696 ms for one — latency dominates. Elevation API batches
  the same way. Response lat/lon = cell centre.
- Cell blending (9 probes, inverse-square, ramped 925→850 hPa; wind blended
  as u/v; surface fields stay nearest) changes ~0.005 °C/km — it removes
  cell-boundary steps, adds no information.

## Verified facts — Windy (for a possible plugin)

- Plugins: TypeScript + Svelte. **Fork** `windycom/windy-plugin-template`
  (publishing runs via a GitHub Action holding `WINDY_API_KEY`). `npm start`
  → accept the cert at `https://localhost:9999/plugin.js` → open
  `windy.com/developer-mode`. Windows: `mkdir dist` then `npm run build:win`.
  `private: true` gives an unlisted URL with no review. Never call
  `setMetric`; read `@windy/metrics` and convert locally.
- **Layout**: `desktopUI` is only `rhpane` | `embedded` — no desktop
  fullscreen. `desktopWidth` (px, default 400) is configurable and the live
  CSS has no max-width; `.plugin__content` scrolls vertically and clips
  horizontally. rhpane closes when the menu opens. Spike this first.
- Endpoint `node.windy.com/forecast/point/{model}/v3.0/{lat}/{lon}
  ?days=…&include=header,sounding`. `gh-*` metres AMSL (`gh-surface` always
  null), `temp-*` Kelvin, `wind-*` m/s resolved, `windDir-*` degrees FROM,
  plus `cloud-*`, `rh-*`, `dewPoint-*`, and **`cloudBase` populated** (ECMWF).
- **Always 3-hourly** — `step=1` is ignored for every model and region tested
  (incl. Windy's own meteogram request). Hourly displays are interpolated.
  Premium changes nothing. Sounding horizon ~5.3 days.
- **ECMWF on Windy has more levels**: 1000/950/925/900/850/800 — six below
  2500 m vs three on Open-Meteo. Windy is on a licensed feed.
- `header.elevation` is **unreliable** (245/275/916 m for identical requests)
  — use an external DEM. `header.modelElevation` is junk.
- Firing many point requests quickly returns inconsistent data — space them.
  Point forecasts are interpolated, so grid spacing can't be measured.
- Precipitation in the sounding/meteogram blocks: **not yet checked**.
- meteoblue on Windy (`mblue`) comes via a "Meteoblue Meteograms" plugin
  path, had a 2-day-old `refTime`, and its `meteogram` block has a 47-level
  cloud ladder but no temp/wind. meteoblue's own SA domains (NEMS-10 at
  10 km, NMM-18) were current — the staleness is Windy's copy. meteoblue runs
  twice daily by design.
- Terms forbid taking Windy met data off-platform.

---

## RASP (from the original chat — set aside, now under discussion)

Two WRF products on rasp.org.za:
- `rasp-hd-d0..d3/FCST/` — region SA_WCAPE. Grid `w2` = **1.3 km**, only at
  1100/1400/1700; parent `d2` covers 0500–1700.
- `beta/ZS2/FCST/` (days `ZS2`, `ZS2+1`…`ZS2+4`) — 300 × 216 grid,
  half-hourly 0600–1500, produced in sequence through the morning (~05:13Z
  to 11:50Z); ask too early and you get yesterday's.

Each chart has a numeric twin `{param}.curr.{HHMM}lst.{grid}.data`: ASCII,
self-describing header (`Model= RASP … Indexs= 1 nx 1 ny Proj= lambert dx dy
tl1 tl2 slon rlat rlon`, then `Param= … Unit= … Min= Max=`), then
whitespace-separated values. ~298 KB per file.

- **Geolocation proven** with sensible heat flux (≈0 over sea): reference
  point = **grid centre**, rows run **south first** (98% land/sea agreement;
  north-first gives 106 W/m² over "sea"). Inverse Lambert conformal code
  exists in the old `rasp-zs2.html`.
- **Times**: filenames say `lst`, headers say UTC — `1200lst` is 1200Z.
- **Fill** is −999999; test on sign, since `zsfclcl` legitimately reaches
  17018 ft. Filled pressure level = surface below terrain.
- **Units**: feet and knots.
- Parseable: `hbl`/`dbl` (BL top/depth), `blcwbase`, `zsfclcl`,
  `blcloudpct`, `sfcwind0/2spd` (2 = 30 m), `blwindspd`, `bltopwindspd`,
  `blwindshear`, `sfctemp`, `cape`, `wstar`, `sfcshf`,
  `press850wspd/wdir`, `press700wspd/wdir`. `press850` is **vertical
  velocity**. No temperature aloft and no soundings as data — **RASP cannot
  replace a pressure-level field**, only supplement the boundary layer.
  `zblcl` is an overcast diagnostic, not a cloud base.
- **No CORS** (an earlier test passed only because the Chrome extension
  bypasses CORS). The old chat built `rasp-proxy.js` (Node, port 8787, 6 h
  disk cache, path-traversal guard, `/rasp-index`). GitHub Pages can't run
  that, so a hosted RASP feature needs a proxy elsewhere or a scheduled job
  (e.g. a GitHub Action) mirroring files into the repo.
- Most useful single output: is the summit **inside or above** the boundary
  layer (`hbl` vs peak elevation)? Inside → gusty mixed layer,
  `blwindspd`; above → free atmosphere, `bltopwindspd`, possibly above a
  cloud deck.
