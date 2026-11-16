---
name: anyways-projects
description: >-
  Use when the user wants to create, list or inspect ANYWAYS projects, or asks
  about the relationship between organizations, projects, scenarios and
  networks. Also use when they mention "ANYWAYS project area", a project
  GeoJSON polygon, baseline vs comparison scenario, copying a network, or
  want to open a project in the ANYWAYS app. Requires the ANYWAYS MCP server
  (see the anyways-mcp skill to add it).
---

# ANYWAYS Projects, Scenarios, Networks

A **project** is the container for a study. It belongs to an **organization** and covers an **area**. Inside it, a **network** is the routable graph — roads, paths and connections — and a **scenario** is one network plus the datasets drawn on it. A new project arrives with two of each, so it can be compared against itself.

```
Organization
  └── Project (area: GeoJSON polygon)
        ├── Network 0 ←── Scenario 0   (baseline)
        └── Network 1 ←── Scenario 1   (a copy of Network 0, to edit)
              └── linked Datasets
```

A network can be used by **any number of scenarios**. That is the fact everything else follows from: two scenarios on the same network share its edits, so editing the roads for one changes what the others say.

## When to use this skill

- Creating a project for a study area.
- Listing or inspecting projects.
- Working out which network or scenario ID to use next.
- Making another network to edit separately.
- Linking to a project in the ANYWAYS app.

For datasets, switch to `anyways-datasets`. To add the server, see `anyways-mcp`.

## Key tools

| Tool | Use |
| --- | --- |
| `whoami` | The signed-in user and the organizations they belong to |
| `list_projects` | Projects, newest first, paged with `offset`; optionally one organization |
| `get_project` | A project with its networks, scenarios and datasets |
| `create_project` | A project with two networks and two scenarios, as the app makes one |
| `copy_network` | Copy a network so a scenario can be edited independently |
| `create_network` | A network from untouched map data — rarely what you want |
| `create_scenario` | A scenario on a network you name |
| `get_scenario` | A scenario's network and the datasets on it |

There are no organization tools beyond `whoami`, and no project copying.

## Creating a project

`create_project(organizationId, name, area, description?)`

- `organizationId` — a GUID from `whoami`.
- `name` — ask the user; do not invent one.
- `area` — a GeoJSON `Polygon`. The app uses the map's visible rectangle, so a rectangle around the place being studied is normal.
- `description` — optional, and if given, **one short sentence**. It is shown next to the name in lists in the app, and the server refuses anything over 200 characters.

```
{"type":"Polygon","coordinates":[[[4.984,51.160],[4.994,51.160],
                                  [4.994,51.170],[4.984,51.170],[4.984,51.160]]]}
```

Coordinates are `[longitude, latitude]` — longitude first, which is easy to swap — and the ring must close, so the last pair repeats the first.

It returns the project with `Network 0`, `Network 1`, `Scenario 0` and `Scenario 1` already made and linked. Nothing else needs calling to have a usable project.

A ~1 km box around a point is roughly `±0.0045°` latitude and `±0.007°` longitude, varying with latitude.

### What the area does, and does not do

The area is **not** a boundary on the road network. A network covers everywhere regardless of it; the area is what the app draws the project on, and it bounds an ABM export taken without an explicit area. So a project area that is too small does not clip routing, and does not have to be enlarged "to avoid edge effects".

## Adding another network

A scenario is usually a what-if about the infrastructure, so the normal way to set one up is to **copy a network first and build the scenario on the copy**:

```
copy_network(projectId, networkId)      → the new network
create_scenario(projectId, name, networkId: <the copy>)
```

`copy_network` takes the roads as they stand now, so two copies taken minutes apart differ if someone edited in between.

Point a new scenario at an **existing** network instead only when the scenarios are meant to differ in their *data* rather than their roads — they will then share every edit.

`create_network` cuts from untouched map data and carries none of the project's accumulated edits, so it will disagree with the project's other networks wherever roads have been corrected. Use it only when starting clean is the point.

## Inspecting a project

```
get_project(projectId)
```

Returns the project with `networks[]`, `scenarios[]` and `datasets[]`. A dataset belongs to the project; linking it to a scenario is a separate step (`add_dataset_to_scenario`).

`list_projects` is newest-first and paged — pass `offset` for more, and `organizationId` to narrow it. There is no search by name.

## Opening in the ANYWAYS app

```
https://www.anyways.eu/app/project/<project-id>
https://www.anyways.eu/app/scenario/<scenario-id>
```

## Notes and gotchas

- **`[longitude, latitude]`.** GeoJSON order. A swap puts the area in the wrong place entirely, and the server rejects pairs out of range rather than accepting them.
- **Rings must close.** First and last coordinate pairs identical.
- **Network 0 is the one to copy.** It is created with the project and carries its edits. `get_project` lists networks by name, so it is normally the first called `Network 0`.
- **Networks are named, never guessed.** Every tool that reads or writes roads requires `networkId`. If the user has not chosen, use Network 0 and tell them that is what you used.
- **"Baseline vs comparison" is convention.** `Scenario 0` is typically untouched and `Scenario 1` is where a change goes. Nothing enforces it.
- **Ask for name and area before creating.** Neither is guessable, and a project is not something to create speculatively.
