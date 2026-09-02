# mb-parcel-data

Bulk generated data shards for the [Manitoba parcel search](https://github.com/jayschellenberg/manitoba-opendata-parcelsearch)
web app, served through jsDelivr pinned to an immutable commit SHA
(never `@main` — branch HEADs lag on the CDN).

| Path | Contents | Built by |
|---|---|---|
| `rollentry-snapshot/` | Per-municipality Roll Entry GeoJSON shards + `_index.json` manifest — the app's fallback when the live provincial FeatureServer is mid-rebuild | `r/build_rollentry_snapshot.R` in the app repo |
| `landfacts/` | Per-municipality land facts for every parcel of 20 acres or more with a MASC rating (173,697 rolls, 147 municipalities): crop history 2009-2025 from the AAFC Annual Crop Inventory (dominant class and crop % per year, `null` = year not observed), relief and mean slope from NRCan MRDEM-30, mapped wetland share and classes from the Canadian Wetland Inventory v3A (10 m), permanent and intermittent open-water shares from JRC Global Surface Water 1984-2021. Keyed by `Roll_No_Txt`; `_index.json` carries the year range, thresholds and sources in `_meta` | `r/build_landfacts.R` in the app repo (`npm run landfacts:shards`); see the app's MAINTENANCE.md 6d |

Source data: Manitoba Open Data (Manitoba geoPortal), redistributed
with provenance recorded in the app's exports.

**Contract:** the app pins one commit SHA. History here exists only to
mint immutable SHAs — it may be periodically squashed to keep the repo
small; only the currently-pinned SHA must remain reachable, so always
repoint the app before pruning.
