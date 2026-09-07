# mb-parcel-data

Bulk generated data shards for the [Manitoba parcel search](https://github.com/jayschellenberg/manitoba-opendata-parcelsearch)
web app. Every file here is fetched by the app at an **immutable commit SHA**
pinned in `web/src/arcgis.js` (`MB_PARCEL_DATA_REVISION`); nothing reads `main`.

## Delivery

The app fetches same-origin `/gh-data/mb-parcel-data/<sha>/<path>`. On Vercel
the `api/gh-data.js` edge function proxies that to `raw.githubusercontent.com`
with Vercel's edge cache in front — immutable per URL, so a repin never needs
a purge, and GitHub only ever sees Vercel egress rather than client IPs. In
`npm run dev` the same path is proxied straight to raw by `vite.config.js`.

This repo served from jsDelivr until 2026-08-17, when it outgrew jsDelivr's
50 MB package limit: cached files kept working while every cold file failed,
so land cover and water quietly returned null for any municipality nobody had
fetched before. Direct raw fetches replaced it, then tripped raw's per-IP rate
limit on the first live check, which is why the proxy exists.

Publishing a rebuilt family is one command in the app repo —
`update-cdn-pin.ps1` commits and pushes this repo and rewrites the pin — then
commit the pin. Until that runs, a new family's column stays blank in the app
rather than saying "None", which is the correct rendering of not knowing.

## Families

All per-municipality families are keyed by the parcel's `Roll_No_Txt` (roll to
three decimals) inside a shard named for `Muni_Name_With_Typ` (e.g.
`PINEY (RM)` → `PINEY_RM.json`), with an `_index.json` manifest mapping the
municipality name to `{ file, count }` plus a `_meta` block carrying the
build's vintage and sources, which the app's Data Status dialog reads.

| Path | Contents | Built by |
|---|---|---|
| `rollentry-snapshot/` | Per-municipality Roll Entry **GeoJSON** FeatureCollections carrying the ten fields the app consumes — the fallback when the live provincial FeatureServer is mid-rebuild. Manifest is nested (`munis: { <name>: { file } }`) and records the snapshot date and source `RollEntry_<date>.gpkg` | `r/build_rollentry_snapshot.R` in the app repo |
| `assessment/` | Per-municipality assessment / tax-history rows (`{ version, muni_no, fields[8], rows[] }`, 186 shards, ~438k rows) from `tax_history.parquet`; manifest lists `{ muni_no, file, row_count }`. Powers value-based filters such as "Vacant land only" | `r/build_assessment_index.R` in the app repo |
| `landcover/` | Five farmland cover fractions per parcel — `cult`, `past`, `bush`, `wet`, `other` (0–1, summing to ~1) — collapsed from the 12 classes of NRCan's **2020 Land Cover of Canada** raster (`LCR_RCT_2020`, 30 m), extracted per parcel by the mao-assembly pipeline. Only parcels over 10 acres (`ACRES_THRESHOLD`, kept in sync with `LAND_COVER_MIN_ACRES` in the app). Not an assessor product | `r/build_landcover.R` in the app repo, bridging the mao-assembly Parquet |
| `landcover-tiles/` | Static XYZ **lossless WebP** raster pyramid of the same 2020 land-cover raster, zooms 6–12 plus `manifest.json` (~138 MB). MapLibre reads it as a plain raster source for the Land Cover overlay's Detailed mode | `r/build_landcover_tiles.R` in the app repo (GDAL / `gdal2tiles.py`) |
| `masc/` | Per-municipality flat arrays of **MASC quarter-section soil ratings** — `{ q, s, t, r, d, rating, ratings, ra, lat, lon }` — for the MASC overlay. Source is the MASC-SCRAPE run named in `_meta.run` | `r/build_masc_shards.R` in the app repo |
| `parcel-masc/` | Per-parcel dominant MASC rating by area overlap — `{ rating, ratings, ra, q, s, t, r, d, source, label }` — so the grid's MASC Rating / Risk Area fill without an overlay load. Only parcels overlapping at least one rated quarter or river lot | `r/build_parcel_masc.R` in the app repo |
| `water/` | Water-influence stamp per parcel — `{ i, c, t, b, d }`: influenced yes/no, class (Waterfront / Direct / Reserve / Near), water-body type and name, distance in feet — from mao-assembly's waterfront detection. Only parcels with a non-"None" classification ship; ~370k of 437k parcels have no water within 50 m and are omitted | `r/build_water.R` in the app repo, bridging the mao-assembly Parquet |
| `flood/` | Flood-zone membership per parcel — `{ z: { <zone code>: <% of parcel> } }` across up to nine zones (statutory Designated Flood Areas, the 1-in-200 extent, observed 1997/2009/2011 extents, Winnipeg waterway corridors), joined at **full resolution** from MBFloodMapping. Only parcels intersecting at least one zone ship. `_meta` carries each layer's fetch date; two statutory layers are frozen at 2022-02-09 upstream | `r/build_flood.R` in the app repo |
| `landfacts/` | Open-data land facts for every parcel of **20 acres or more with a MASC rating** (173,697 rolls, 147 municipalities): crop history 2009–2025 from the AAFC Annual Crop Inventory (`cp` crop % and `dom` dominant class per year, `null` = year not observed, never 0), relief and mean slope from NRCan MRDEM-30 (`rel`, `slp`, `z`), mapped wetland share and classes from the Canadian Wetland Inventory v3A at 10 m (`wet`, `wc`), permanent and intermittent open-water shares from JRC Global Surface Water 1984–2021 (`gsw`, `gsi`). ~215 bytes per parcel; `_meta` carries the year range, thresholds and sources. In the app this family feeds the **Land Facts** column, popup and CSV, and the **Crop History** map overlay, which has two views derived from the same `cp`/`dom` series: **Years Cropped** (share of observed years with crop ≥ 50%, gold ramp) and **Land Use** (cover group of the last observed year: annual crop, grass/pasture, trees/shrub, water/wetland, barren/built) | `r/build_landfacts.R` in the app repo (`npm run landfacts:shards`); operating notes in the app's MAINTENANCE.md §6d |
| `mf-newbuild/` | **Multi-family new construction** on rolls with **3 or more dwelling units**, excluding any roll carrying a farm class (101 municipalities, 607 rolls, 686 events, 170 KB). Dated from 20 years of assessed BUILDING VALUE rather than building permits — outside Winnipeg there is no province-wide permit feed, and MAO publishes dwelling units only as a current scalar with no history. Manitoba freezes assessed values between biennial reassessments, so a within-biennium jump is almost pure physical change: across 2008–2027, 99.8% of Residential 2 rolls move across a reassessment boundary but only 7.9% move within a biennium. Cross-boundary jumps are normalised against the median revaluation factor for that municipality **and** dominant class and carry lower confidence. Each roll holds `du` (current units), `ad`, `cl`, `p` (primary event year) and `e[]` — every event as `{y, k, b, bp, c}`, where `k` is `appeared` / `expanded` / `new_roll` and `c` is `high` / `med` / `low`; `sdu` adds at-sale unit counts from the sales PDF archive where the roll sold. Detection runs at the ROLL level across all classes — requiring Residential 2 in both years finds 2 events province-wide, because a new block arrives as a new roll or reclassifies into R2 the same year the building appears. Event years are ASSESSMENT years and trail completion by about a year. The farm exclusion is load-bearing: 1,222 of the 1,841 rolls with 5+ units are Hutterite colonies at 20–35 units and $20–38M of buildings. In the app this family feeds the **New MF** column, popup and CSV, and the **New Multi-Family** map overlay (**Year** red recency ramp / **Units** blue size ramp) | `r/build_mf_newbuild.R` in the app repo (`npm run mf:shards`) |

Standalone files: `river-lots.json` (river-lot polygons) and `masc-riverlots.json`
(the same lots with their MASC ratings), read directly by the app.
`section-grid.json` (40 MB) is also tracked here but **not read** — the app
ships it from a GitHub Release through its `/api/section-grid` edge function.

Source data: Manitoba Open Data (Manitoba geoPortal), MASC, NRCan, AAFC, DUC,
JRC and MBFloodMapping, redistributed with provenance recorded in each
family's `_meta` and in the app's exports.

**Contract:** the app pins one commit SHA. History here exists only to
mint immutable SHAs — it may be periodically squashed to keep the repo
small; only the currently-pinned SHA must remain reachable, so always
repoint the app before pruning.
