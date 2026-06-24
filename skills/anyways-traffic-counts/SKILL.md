---
name: anyways-traffic-counts
description: >-
  Use when the user wants to discover, inspect, or export traffic-counter
  data through the ANYWAYS MCP server — counts from AWV (Flanders), NDW
  (Netherlands), BASt (Germany), UK National Highways, TII (Ireland),
  Trafikverket (Sweden), or Vejdirektoratet (Denmark). Also use when they
  mention AADT, AM/PM peak, hourly traffic counts, counter discovery by
  bbox or polygon, a SUUNTA-style export bundle, or want to write a Python
  script that produces CSVs of counter data. Requires the ANYWAYS MCP
  server (see the anyways-mcp skill).
---

# ANYWAYS Traffic Counts

The ANYWAYS MCP server exposes a read-only surface over the **traffic-counts-api**: a multi-provider catalogue of road traffic counters with hourly counts and pre-computed annual aggregates. Each provider models several real-world counter stations; each station has one or more sub-counters (lanes / directions). Hourly counts and aggregations live at the sub-counter level.

```
Provider (awv, ndw, bast, uk, tii, trafikverket, denmark)
  └── Counter (UUID GlobalId — the public identifier)
        └── ProviderCounter (catalogue grouping; one per Counter)
              └── ProviderSubCounter[] (lane / direction; hourly data lives here)
```

## When to use this skill

- "Show me traffic counters near …" — discover by bbox or polygon.
- "What's the AADT / morning peak at counter X for 2025?" — aggregated stats.
- "Export a year of hourly counts for every counter in this polygon to CSV" — bulk export workflow.
- Building a Python or shell script against the MCP server that pulls counter data.

For the underlying account / sign-in flow, use the `anyways-mcp` skill. The traffic-counts tools are public reads — anyone signed in to the MCP server can call them; no special role needed.

## Provider keys

| Key | Name | Country | Granularity |
| --- | --- | --- | --- |
| `awv` | AWV (Vlaams Verkeerscentrum) | Belgium (Flanders) | minute |
| `ndw` | NDW | Netherlands | minute |
| `bast` | BASt | Germany | hour |
| `uk` | National Highways (WebTRIS) | United Kingdom | hour |
| `tii` | Transport Infrastructure Ireland | Ireland | hour |
| `trafikverket` | Trafikverket | Sweden | minute |
| `denmark` | Vejdirektoratet | Denmark | annual only |

`denmark` only carries pre-computed annual AADT — hourly data is not available there.

## Vehicle types

| Int | String | Notes |
| --- | --- | --- |
| 1 | Car | Passenger car |
| 2 | Bicycle | Bicycle / e-bike |
| 3 | Truck | Heavy goods vehicle |
| 4 | Pedestrian | Pedestrian |
| 5 | AnyVehicle | Undivided total used by sources that don't break out by class (some NDW sites). Prefer per-class when available. |

Pass the integer to `vehicleType` parameters; responses serialise the string.

## Key tools

| Tool | Use |
| --- | --- |
| `list_traffic_count_providers` | List all providers with attribution and counter counts |
| `find_traffic_counters` | Counters whose snapped road segment intersects a bbox; hard-cap 5000 |
| `get_traffic_counter` | Full counter detail: geometry, sub-counters, provider metadata |
| `get_traffic_counter_aggregated` | AADT + AM/PM peak per (sub-counter, year, vehicle type) |
| `get_traffic_counter_hourly` | Raw hourly counts (UTC), up to 366-day window, summed or per-sub-counter |

## Discovery workflow

```
1. find_traffic_counters(minLon, minLat, maxLon, maxLat, provider="awv")
   → { bbox, count, truncated, counters: [{ globalId, provider, lon, lat, name, ... }] }
2. If truncated=true, split the bbox into quadrants and recurse.
3. (Optional) Point-in-polygon filter the (lon, lat) of each counter against
   a polygon to narrow further — done client-side, not in the API.
4. get_traffic_counter(globalId)  for each candidate
   → snapped Line geometry + subCounters[] (the join key for hourly data).
```

`find_traffic_counters` excludes counters that haven't been snapped to a road yet (`Line IS NULL`). If you already have a globalId, `get_traffic_counter` returns it with the centroid even when the snap loop is still pending.

## Aggregated data

```
get_traffic_counter_aggregated(globalId, year?, vehicleType?)
→ data: [{
    year, vehicleType, subCounterKey,
    aadt, amPeak, pmPeak,
    weekdayCount, totalDayCount, updatedAt
  }, ...]
```

- **AADT** is the annual mean over **all days with data** (FHWA / TMG convention — not weekday-only).
- **AM peak / PM peak** are the 95th percentile of per-day totals in **local hour 7 / 16** (provider's time zone), over **weekday-non-holiday** days only.
- `aadt`, `amPeak`, `pmPeak` are `null` until there are ≥20 qualifying weekday-non-holiday days. Recently-added counters and the current/previous day are always excluded (2-day completion lag).
- Counters without an assigned region are skipped (no holiday calendar = no peak filtering).

## Hourly data

```
get_traffic_counter_hourly(globalId, from, to, vehicleType?, perSubCounter?)
→ hourly: [{ day, hour, vehicleType, count, samples, subCounterKey? }, ...]
```

Critical caveats:

- **UTC throughout.** `day` is a UTC date; `hour` is the UTC hour `0..23`. The response includes `"timeZone": "UTC"` as a reminder.
- **Missing rows = sensor offline that hour.** Do **not** treat them as zeros. The `samples` column is the data-quality signal — max 60 for a sub-counter under minute-native providers (one slot per minute); a half-broken sensor reports `samples=30`.
- **Per-sub-counter mode.** Default `perSubCounter=false` sums across sub-counters. Set `perSubCounter=true` for per-lane / per-direction rows — each row then includes `subCounterKey` which joins to `subCounters[].key` from `get_traffic_counter`.
- **366-day cap.** `to - from` must be ≤ 366 days. Longer windows return HTTP 400.

## Export bundle workflow

For a SUUNTA-style export — *all counters in a polygon, full year of hourly counts, CSV bundle* — drive the MCP server from a Python script (or any HTTP/MCP client). The LLM should not orchestrate this; one year × ~100 counters is tens of MB of JSON that won't fit in an LLM context.

```
Inputs: polygon (GeoJSON), date range, provider key.

Phase 1 — Discovery
  bbox = bbox-of(polygon)
  counters = find_traffic_counters(*bbox, provider=...).counters
  shortlist = point_in_polygon_filter(counters, polygon)
  → write awv-YYYY-counters.geojson

Phase 2 — Catalogue (one row per sub-counter)
  for each counter in shortlist (parallel ~8):
      detail = get_traffic_counter(counter.globalId)
      flatten subCounters[] → CSV row with counter + sub_counter metadata
  → write awv-YYYY-subcounters.csv

Phase 3 — Hourly (the bulk)
  for each counter in shortlist (parallel ~8):
      rows = get_traffic_counter_hourly(globalId, from, to, perSubCounter=true)
      pivot vehicleType across columns:
        car_count, car_samples, truck_count, truck_samples, any_count, any_samples, ...
      append to awv-YYYY-hourly.csv (stream — don't build a giant list)
  → write awv-YYYY-hourly.csv
```

Output bundle layout (mirrors prior client expectations):

```
2026-XX-XX-{study}-export/
├── polygon.geojson
├── awv-2025-counters.geojson
├── awv-2025-subcounters.csv
└── awv-2025-hourly.csv
```

**Operational guardrails:**

- Retry HTTP 5xx and timeouts up to 3 times with exponential backoff.
- Never fold missing hours into zeros — leave them blank in the CSV. `samples` is the data-quality column the client uses.
- Sanity check: pick one counter, call `get_traffic_counter_hourly` without `perSubCounter`, and confirm your per-sub-counter rows sum to the counter-level totals per `(day, hour, vehicleType)`.

For AWV around Antwerp, full year: phase 1 = 1 call, phases 2 and 3 ≈ 50–150 calls each at 8-way parallel, total wall-clock well under an hour, total disk ~100–200 MB.

## Attribution

Each provider response includes an `attribution` string. **Surface it to end users** whenever you display data from that provider — it's a licence condition for several of the sources (Vlaams Verkeerscentrum, BASt, etc.). `portalUrl` is the canonical public landing page for each provider.

## Notes and gotchas

- **Coord order.** `find_traffic_counters` takes `(minLon, minLat, maxLon, maxLat)` — longitude first, matching GeoJSON. Latitude first in human conversation is a frequent swap bug.
- **`truncated=true`** in `find_traffic_counters` means the result was clipped at 5000. Narrow the bbox or split by provider — don't assume the visible counters are everything in scope.
- **AADT vs AAWT.** Our AADT is the all-days annual mean (FHWA convention). If the user expects a weekday-only number (AAWT), that's a different metric — flag it.
- **Holiday filtering needs a region.** Counters whose `region` is null have `null` for `amPeak`/`pmPeak` even if they have plenty of data.
- **Sub-counter grouping.** Two-direction counters (e.g. BASt R1/R2) are modelled as **two separate Counters** sharing a point, not one Counter with two sub-counters. AWV per-lane breakdowns are modelled as **one Counter with several sub-counters**. The geometry difference matters when picking a join strategy.
- **Pagination doesn't exist.** Date range is capped at 366 days, bbox results at 5000 — there's no "next page" cursor. Split the input instead.
- **Tiles endpoint is not exposed via MCP.** The traffic-counts-api also serves binary MapLibre vector tiles, but they're not LLM-usable; the MCP server does not wrap them.
