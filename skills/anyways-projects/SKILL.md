---
name: anyways-projects
description: >-
  Use when the user wants to create, list, inspect, or copy ANYWAYS projects,
  or asks about the relationship between organizations, projects, scenarios,
  and networks. Also use when they mention "ANYWAYS project area", project
  GeoJSON, baseline vs comparison scenario, or want to open a project in
  the ANYWAYS app. Requires the ANYWAYS MCP server (see the anyways-mcp
  skill to add it).
---

# ANYWAYS Projects, Scenarios, Networks

A **project** is the top-level container for a study in ANYWAYS. It belongs to an **organization** and covers a geographic **area** (a GeoJSON polygon). Every project ships with two **scenarios** (a baseline and a comparison) and two **networks** (one per scenario). Datasets are attached to scenarios to feed them with trips and locations.

```
Organization
  └── Project (area: GeoJSON polygon)
        ├── Scenario 0  ── Network 0  ── linked Datasets
        └── Scenario 1  ── Network 1  ── linked Datasets
```

## When to use this skill

- Creating a new project for a study area.
- Listing or inspecting existing projects in an organization.
- Copying a project to fork analysis work.
- Understanding which scenario / network / dataset IDs to use in follow-on calls.
- Generating a link to open a project in the ANYWAYS app.

For dataset creation and editing, switch to the `anyways-datasets` skill.

## Key tools

| Tool | Use |
| --- | --- |
| `search_organizations` | Find an organization by name |
| `list_organizations` | List all orgs the user can access |
| `list_projects` | List projects in an organization |
| `get_project` | Full project detail incl. scenarios, networks, datasets |
| `create_project` | Create a new project with a GeoJSON area |
| `copy_project` | Duplicate an existing project |
| `get_snapshot_commit` | Resolve the commit a project network is pinned to |

`create_project` requires:

- `name` — project name.
- `organizationId` — GUID, from `search_organizations` or `list_organizations`.
- `areaGeoJson` — GeoJSON `Polygon` covering the area of interest. Coordinates are `[lon, lat]` and the ring must close (first = last point).

It returns a new project with two scenarios and two networks already initialised.

## Quick reference

```
GeoJSON polygon shape:
  {"type":"Polygon","coordinates":[[[lon,lat],[lon,lat],...,[lon,lat]]]}

Coord order is [lon, lat]   ── easy to swap by accident.
Ring must close             ── first point must equal last point.
Area should cover the study ── too small clips the routing, too big slows things down.
```

## Typical workflow: create a project

1. **Find the organization.**

   ```
   search_organizations(query="ANYWAYS")
   → [{ name, id, _type:"organization" }, ...]
   ```

2. **Decide the area.** Either:
   - Ask the user for explicit coordinates or a bounding box.
   - Geocode a place name and build a box around it.

   A ~1 km box around (lon, lat) is roughly `±0.0045°` latitude and `±0.007°` longitude (varies with latitude — wider at the equator, narrower towards the poles).

3. **Create the project.**

   ```
   create_project(
     name="Geel Office Test",
     organizationId="<org-guid>",
     areaGeoJson='{"type":"Polygon","coordinates":[[
       [4.984,51.160],[4.994,51.160],[4.994,51.170],[4.984,51.170],[4.984,51.160]
     ]]}'
   )
   → "Project '...' created with ID <project-guid>, 2 networks, 2 scenarios."
   ```

4. **Resolve the scenario IDs** for any follow-up dataset work.

   ```
   get_project(projectId="<project-guid>")
   → returns the project plus its scenarios, networks, datasets
   ```

   The baseline scenario is usually `Scenario 0`. Capture both scenario IDs from the returned list.

5. **Link datasets to scenarios** (see `anyways-datasets`).

## Typical workflow: inspect a project

```
get_project(projectId="<guid>")
```

Returns the project plus every related entity in one call:

- `scenarios[]` — each with its `network` reference and linked `datasets[]`.
- `networks[]` — pinned to a specific OSM `branch`.
- `datasets[]` — all datasets in the project (a dataset belongs to a project; linking to a scenario is a separate step).
- `routeplanners[]`, `congestions[]` — when present.

Use this as the entry point whenever you have a project ID and need to find related entities.

## Opening a project in the ANYWAYS app

After creating or finding a project, build a URL the user can open:

```
https://www.anyways.eu/app/project/<project-id>
```

For a specific scenario:

```
https://www.anyways.eu/app/scenario/<scenario-id>
```

## Notes and gotchas

- **GeoJSON coordinate order.** GeoJSON uses `[longitude, latitude]`. A common bug is swapping the two — points will land in the wrong hemisphere.
- **Closed rings.** A GeoJSON polygon ring must close: the first and last coordinate pairs must be identical. Forgetting this is a common cause of `create_project` validation errors.
- **Area sizing.** The project area determines what road network is loaded for routing. Too small and routes near the edge get clipped; too large and the project becomes slow to work with. Aim for the smallest box that comfortably covers the study area and a small buffer.
- **"Baseline vs comparison"** is convention only. `Scenario 0` is typically baseline (no edits) and `Scenario 1` is the comparison (edited network, added datasets). Nothing prevents using them differently.
- **Always ask the user for project name and area before calling `create_project`.** The MCP tool's own description requires this; do not guess either.
