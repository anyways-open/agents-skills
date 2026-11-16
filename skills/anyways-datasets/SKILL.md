---
name: anyways-datasets
description: >-
  Use when the user wants to create, inspect or link ANYWAYS datasets, or asks
  about modelled routes from the agent-based model, traffic-count calibration,
  or Location of Interest (LOI) datasets. Also use when they mention
  origin/destination trips, donut rings around a site, fitting or calibrating
  against counts, AADT into a model, or routing profiles. Requires the ANYWAYS
  MCP server (see the anyways-mcp skill) and a project (see anyways-projects).
---

# ANYWAYS Datasets

A **dataset** is the demand put over a network: trips between origins and destinations, each carrying a count. A dataset belongs to a project and takes effect when it is **linked to a scenario**.

```
Project
  ├── Dataset  ── trips (origin → destination, profile, count)
  └── Scenario ── add_dataset_to_scenario ── Dataset
```

Three kinds can be made, and they are not alternatives so much as a sequence: the model gives you realistic trips, counts put them on the right level.

## When to use this skill

- Choosing how to represent demand for a study question.
- Building demand from the agent-based model.
- Calibrating or fitting against observed traffic counts.
- Modelling traffic to and from a single site.
- Linking a dataset to a scenario, or seeing what a scenario holds.

For projects, networks and scenarios, use `anyways-projects`.

## Choosing a dataset type

| Tool | What it represents | Use when |
| --- | --- | --- |
| `create_abm_dataset` | The agent-based model's own routes over an area, folded into OD trips | You want realistic demand without placing anything by hand. **The usual starting point.** |
| `create_counters_dataset` | Observed counts acting on trips copied from another dataset | You have measured counts and want the model on the right level |
| `create_locations_dataset` | Trips between one site and random points in rings around it | "Where does traffic to and from this site come from?" |

There is no hand-drawn dataset type, no dataset deletion, and no GeoJSON export.

## Modelled routes — `create_abm_dataset`

```
create_abm_dataset(projectId, name?, description?, fromHour?, toHour?, area?)
get_abm_dataset(datasetId)
```

Generates immediately: the dataset has trips when the call returns. `area` defaults to the project's own polygon — pass one explicitly when the catchment feeding the study is wider than the project area. `fromHour`/`toHour` filter to an hourly window.

Only car routes exist in the model today.

These are what the app calls **modelled routes**, and they are the usual source for a counters dataset.

## Traffic counts — `create_counters_dataset`

```
create_counters_dataset(projectId, networkId, sourceDatasetId?, name?,
                        description?, method?, profiles?, counters?)
add_counters(datasetId, counters)
get_counters_dataset(datasetId)
```

A counters dataset needs two things: **trips** to act on, and **counters** — counted volumes on particular road segments.

**The trips always come from another dataset**, named by `sourceDatasetId`. Origins and destinations cannot be placed directly on a counters dataset through the MCP server, so without a source it has nothing to fit the counts onto and finishes holding nothing. Use `create_abm_dataset` (usual) or `create_locations_dataset`, and wait for that source to be finished before copying from it.

### The two methods

| `method` | What the counts do | Use when |
| --- | --- | --- |
| `calibrate` | The trips already describe the *shape* of the traffic; the counts set the *level*. Every trip is scaled by one factor fitted to all counters at once, including trips no counter can see. | The trips come from the model, or an earlier dataset of your own |
| `fit` | The counters are the evidence of how much traffic exists. Each counter's volume is fitted onto the trips crossing it, and a trip no counter sees carries nothing. | The counts are all you have |

Defaults to `calibrate` with a source and `fit` without, matching the app, where these are called *"calibrate the source"* and *"fit counters to the routes"*. Either can be switched afterwards and the result is recomputed.

### Which network

`networkId` is required. Counts are observations of the real world, so use the network matching the roads **as they were when the counts were taken** — normally `Network 0`, the existing state. A network carrying planned changes would fit the counts to roads that were not there.

### Placing a counter

A counter is a short line **in the direction traffic travels**: reversing start and end measures the other carriageway. Each entry is `originLongitude`, `originLatitude`, `destinationLongitude`, `destinationLatitude`, `count`.

Do not guess the geometry. Use `find_counter_segment(projectId, networkId, latitude, longitude)`: given the spot where traffic was counted, it returns the snapped, driving-side segment for each direction the profile can actually travel — so a one-way street offers one direction, not two. Take the direction that matches the count and pass it as a counter.

To explore the network first — "is there a road called N104 near here", "what is around this point" — use `find_roads`, which returns matching roads as junction-to-junction stretches with geometry.

### It is not finished when it returns

`create_counters_dataset` returns as soon as the specification is written. A worker then copies the trips, routes them and fits the counters, which takes seconds to minutes. Poll `get_counters_dataset(datasetId)` until it reports done, and read the per-counter results: each says what the counter measured and the ratio of fitted traffic to it, where **1 is exact**.

## Trips around a site — `create_locations_dataset`

```
create_locations_dataset(projectId, latitude, longitude, rings,
                         origin?, profile?, name?, description?)
```

Places one Location of Interest and generates random points in concentric bands around it, one trip per point.

`rings` is a list of `{fromMetres, toMetres, trips}` — explicit distances, so bands may have gaps. `fromMetres: 0` starts at the site itself. Each trip carries a count of 1.

```
rings: [{fromMetres: 0, toMetres: 1000, trips: 50},
        {fromMetres: 1000, toMetres: 5000, trips: 30}]
```

`origin: true` (the default) makes the site the origin of every trip — traffic leaving it. `false` makes it the destination.

## Linking to a scenario

A dataset does nothing until it is on a scenario.

```
add_dataset_to_scenario(scenarioId, datasetId)
remove_dataset_from_scenario(scenarioId, datasetId)
get_scenario(scenarioId)
```

Both are idempotent and both refuse to link a dataset from a different project.

## Typical workflow: counts on modelled routes

This is the flow to reach for when someone has traffic counts and wants a calibrated model.

1. `get_project(projectId)` — note `Network 0` and the scenario to put this on.
2. `create_abm_dataset(projectId, ...)` — modelled routes. Trips exist on return.
3. `find_counter_segment(projectId, networkId, lat, lon)` for each counted location, taking the direction that matches the count.
4. `create_counters_dataset(projectId, networkId: <Network 0>, sourceDatasetId: <the ABM dataset>, counters: [...])` — `calibrate` by default.
5. Poll `get_counters_dataset` until done; check each counter's ratio against its count.
6. `add_dataset_to_scenario(scenarioId, datasetId)`.

## Notes and gotchas

- **A counters dataset with no source finishes empty.** Nothing refuses it, so the symptom is a dataset that completes holding nothing.
- **Counter direction matters.** A counter on the direction traffic cannot travel can never be crossed, so it fits to zero. `find_counter_segment` only offers directions the profile can travel.
- **Descriptions are one short sentence.** They appear next to the name in the app's lists; the server refuses anything over 200 characters. Put reasoning in your reply to the user, not on the record.
- **`profiles` defaults to `car.fast`, `car.short`, `car.classifications`** — what the app uses. More profiles means more possible paths between an origin and a destination.
- **Datasets cannot be deleted** through the MCP server, so do not create them speculatively while exploring.
- **Nothing reports how much of a dataset informed a calibration.** A factor fitted from a handful of counters is applied to every trip, and the response does not say how thin the evidence was — read the per-counter results and judge.
