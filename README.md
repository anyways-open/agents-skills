# agents-skills

Agent skills for working with the [ANYWAYS](https://www.anyways.eu) platform. These skills give any AI coding agent the context it needs to drive the ANYWAYS MCP server correctly — including how to add the server, the project/scenario/network data model, and the available dataset types.

The skills are plain markdown files that work with any agent that can read documentation. First-class installation is provided for Claude Code via its plugin system.

## Skills

| Skill | Purpose |
|-------|---------|
| `anyways-mcp` | Add and authenticate the ANYWAYS MCP server in your agent |
| `anyways-projects` | Project / scenario / network data model and common workflows |
| `anyways-datasets` | The dataset types (modelled routes / counters / locations) and when to use each |

### anyways-mcp

Use when adding the ANYWAYS MCP server to Claude Code, Claude Desktop, Cursor, or any other MCP-aware client, or when troubleshooting the OAuth sign-in flow.

### anyways-projects

Use when working with ANYWAYS projects — creating a project with a geographic area, listing or inspecting projects, copying a network, or understanding how scenarios and networks relate to a project.

### anyways-datasets

Use when creating datasets in a project. Covers the three dataset types:

- **modelled routes** — the agent-based model's own routes over an area, folded into origin/destination trips. The usual starting point.
- **counters** — observed traffic counts, acting on trips copied from another dataset, either calibrating its level or fitting volumes onto the routes that cross each counter.
- **locations** — a single Location of Interest with random surrounding points and trips towards or away from it.

## Installation

See [install.md](install.md) for setup instructions covering Claude Code, Gemini CLI, Codex, and other agents.

## Prerequisites

- An ANYWAYS account with access to the organizations and projects you want to work with — sign up at [www.anyways.eu](https://www.anyways.eu).
- An MCP client that supports remote HTTP servers with OAuth (Claude Code, Claude Desktop, Cursor, Gemini CLI, etc.).

The MCP server runs at `https://api.anyways.eu/impact/mcp` and uses OAuth against `https://www.anyways.eu/account`. The sign-in flow is handled by your MCP client — but it must be configured with the `anyways-mcp` client ID and callback port 7331, because the ANYWAYS identity server does not support dynamic client registration. See the `anyways-mcp` skill.

---

## About ANYWAYS

<a href="https://www.anyways.eu"><img src="https://www.anyways.eu/anyways-logo-128.png" alt="ANYWAYS logo" width="128"></a>

ANYWAYS BV builds tools for transport planning and scenario analysis based on open data. Learn more at [www.anyways.eu](https://www.anyways.eu) and in the [documentation](https://docs.anyways.eu).

## License

[MIT](LICENSE.txt).
