# mb-parcel-data

Bulk generated data shards for the [Manitoba parcel search](https://github.com/jayschellenberg/manitoba-opendata-parcelsearch)
web app, served through jsDelivr pinned to an immutable commit SHA
(never `@main` — branch HEADs lag on the CDN).

| Path | Contents | Built by |
|---|---|---|
| `rollentry-snapshot/` | Per-municipality Roll Entry GeoJSON shards + `_index.json` manifest — the app's fallback when the live provincial FeatureServer is mid-rebuild | `r/build_rollentry_snapshot.R` in the app repo |

Source data: Manitoba Open Data (Manitoba geoPortal), redistributed
with provenance recorded in the app's exports.

**Contract:** the app pins one commit SHA. History here exists only to
mint immutable SHAs — it may be periodically squashed to keep the repo
small; only the currently-pinned SHA must remain reachable, so always
repoint the app before pruning.
