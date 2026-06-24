# agents-skills

Agent skills for working with the [ANYWAYS](https://www.anyways.eu) platform. These skills give any AI coding agent the context it needs to drive the ANYWAYS MCP server correctly — including how to add the server, the project/scenario/network data model, and the available dataset types.

The skills are plain markdown files that work with any agent that can read documentation. First-class installation is provided for Claude Code via its plugin system.

## Skills

| Skill | Purpose |
|-------|---------|
| `anyways-mcp` | Add and authenticate the ANYWAYS MCP server in your agent |
| `anyways-projects` | Project / scenario / network data model and common workflows |
| `anyways-datasets` | The three dataset configurations (manual / counters / locations) and when to use each |
| `anyways-traffic-counts` | Multi-provider traffic-counter discovery, AADT/peaks, hourly counts, export bundles |

### anyways-mcp

Use when adding the ANYWAYS MCP server to Claude Code, Claude Desktop, Cursor, or any other MCP-aware client, or when troubleshooting the OAuth sign-in flow.

### anyways-projects

Use when working with ANYWAYS projects — creating a project with a geographic area, listing or inspecting projects, copying a project, or understanding how scenarios and networks relate to a project.

### anyways-datasets

Use when creating or editing datasets in a project. Covers the three dataset types:

- **manual** — you draw origin/destination trips on the map.
- **counters** — counter stations with origin/destination counts at fixed points.
- **locations** — a single Location of Interest with random surrounding points and trips towards or away from it.

### anyways-traffic-counts

Use when working with real-world traffic-counter data from one of the integrated providers (AWV, NDW, BASt, UK National Highways, TII, Trafikverket, Vejdirektoratet). Covers counter discovery by bbox, fetching aggregated AADT / AM-PM peaks, pulling raw hourly counts, and the multi-phase export pipeline a Python script would drive against the MCP server.

## Installation

See [install.md](install.md) for setup instructions covering Claude Code, Gemini CLI, Codex, and other agents.

## Prerequisites

- An ANYWAYS account with access to the organizations and projects you want to work with — sign up at [www.anyways.eu](https://www.anyways.eu).
- An MCP client that supports remote HTTP servers with OAuth (Claude Code, Claude Desktop, Cursor, Gemini CLI, etc.).

The MCP server runs at `https://api.anyways.eu/mcp/` and uses OAuth against `https://www.anyways.eu/account/`. Your MCP client handles the sign-in flow automatically.

---

## About ANYWAYS

<a href="https://www.anyways.eu"><img src="https://www.anyways.eu/anyways-logo-128.png" alt="ANYWAYS logo" width="128"></a>

ANYWAYS BV builds tools for transport planning and scenario analysis based on open data. Learn more at [www.anyways.eu](https://www.anyways.eu) and in the [documentation](https://docs.anyways.eu).

## License

[MIT](LICENSE.txt).
