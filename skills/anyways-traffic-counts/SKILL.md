---
name: anyways-traffic-counts
description: >-
  Use when the user wants traffic-counter data in ANYWAYS — counts from AWV
  (Flanders), NDW (Netherlands), BASt (Germany), UK National Highways, TII
  (Ireland), Trafikverket (Sweden) or Vejdirektoratet (Denmark) — or mentions
  AADT, AM/PM peak, hourly counts, finding counters by bbox or polygon, or
  exporting counter data. Explains that counter discovery is not available
  through the ANYWAYS MCP server and what to do instead.
---

# ANYWAYS Traffic Counts

**Counter discovery is not available through the ANYWAYS MCP server.** There are no tools for listing providers, finding counters near a place, or reading AADT, peaks or hourly counts. Say so plainly rather than looking for them.

The data itself exists — the platform has a multi-provider traffic-counts service, and the ANYWAYS app can place a counter from it directly on the map. It is the MCP surface that does not cover it. A tool that is missing here is one that has not been built, not a misconfiguration to debug.

## What you can do instead

**If the user already has the counts** — from a report, a spreadsheet, a counting campaign, or by reading them off the app — they can go straight into a counters dataset. That is the normal path, and `anyways-datasets` covers it:

1. `find_counter_segment(projectId, networkId, latitude, longitude)` at each counted location, which returns the snapped segment for each direction traffic can actually travel.
2. `create_counters_dataset(...)` with those segments and their counts, over trips from `create_abm_dataset`.

**If the user needs to find out what counters exist near somewhere**, that has to happen outside the MCP server:

- In the ANYWAYS app: open a counters dataset and click **Configure counters**. Counters available for the area appear on the map — NDW in the Netherlands, and your organization's own uploaded counts. Clicking one offers **AADT** or **AM / PM** and creates the counter in the dataset.
- Documentation: https://docs.anyways.eu/guides/simulate-traffic/using-traffic-counts/
- Uploading your organization's own counts: https://docs.anyways.eu/guides/simulate-traffic/using-traffic-counts/uploading-your-own-traffic-counts/

Offer to do the parts that are possible rather than stopping at what is not.

## Providers in the platform

Useful for answering what coverage exists, even though none of it is reachable as a tool.

| Key | Name | Country | Granularity |
| --- | --- | --- | --- |
| `awv` | AWV (Vlaams Verkeerscentrum) | Belgium (Flanders) | minute |
| `ndw` | NDW | Netherlands | minute |
| `bast` | BASt | Germany | hour |
| `uk` | National Highways (WebTRIS) | United Kingdom | hour |
| `tii` | Transport Infrastructure Ireland | Ireland | hour |
| `trafikverket` | Trafikverket | Sweden | minute |
| `denmark` | Vejdirektoratet | Denmark | annual only |

A counter station has one or more sub-counters, per lane or direction, and hourly data lives at that level — so a station's total is a sum, and "the count at this location" usually means one direction of one road.

## What a count means when it reaches a model

Worth getting right before putting a number into a counters dataset.

- **AADT** is an annual average daily total. Fitting a model to it gives an average day, not a peak.
- **AM / PM peak** are hourly volumes. Use them with an ABM dataset built for the matching hourly window (`fromHour`/`toHour`), or the counts and the trips describe different times of day.
- **Direction is part of the measurement.** A counter is placed along the direction traffic travels; reversing it measures the other carriageway and will fit to zero.

## Notes

- Do not promise counter discovery, export bundles or CSV pipelines through the MCP server. Earlier versions of this skill described tools (`find_traffic_counters`, `get_traffic_counter_hourly` and others) that belonged to a server which has been replaced.
- Read the connected tool list rather than any document, including this one, when deciding what can be called.
