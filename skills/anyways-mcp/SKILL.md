---
name: anyways-mcp
description: >-
  Use when the user wants to add or troubleshoot the ANYWAYS MCP server in
  Claude Code, Claude Desktop, Cursor, Gemini CLI, or any other MCP-aware
  client. Also use when they mention "ANYWAYS MCP", api.anyways.eu/mcp,
  ANYWAYS sign-in, or the OAuth flow against www.anyways.eu/account.
---

# ANYWAYS MCP — Setup and Authentication

The ANYWAYS MCP server exposes the ANYWAYS platform (organizations, projects, scenarios, networks, datasets, routing) as tools that an MCP-aware LLM client can call directly.

```
Server URL:  https://api.anyways.eu/mcp/
Transport:   HTTP (Streamable HTTP)
Auth:        OAuth 2.1, handled automatically by the MCP client
Identity:    https://www.anyways.eu/account/
```

## When to use this skill

- User wants to add the ANYWAYS MCP server to their agent.
- User signed in but no tools appear.
- User gets a 401 / 403 from an ANYWAYS MCP tool.
- User asks how the ANYWAYS auth flow works or how to switch accounts.

For *what to do* with the MCP once it is connected, use the `anyways-projects`, `anyways-datasets`, and `anyways-traffic-counts` skills.

## Prerequisites

- An ANYWAYS account with access to the organizations/projects you intend to use. Sign up or sign in at https://www.anyways.eu/account/.
- An MCP client that supports remote HTTP MCP servers with OAuth.

## Adding the server

The connection details are always the same; only the UI differs per client.

| Field | Value |
| --- | --- |
| Name | `anyways` (or any label) |
| Type / Transport | HTTP (Streamable HTTP) |
| URL | `https://api.anyways.eu/mcp/` |

### Claude Code (CLI)

```bash
claude mcp add --transport http anyways https://api.anyways.eu/mcp/
```

Then in a session run `/mcp`. On first use, Claude Code opens a browser to the ANYWAYS sign-in page. After authorising, the tools appear in that session.

### Claude Desktop

`Settings → Connectors → Add custom connector`. Set Name = `anyways`, URL = `https://api.anyways.eu/mcp/`. Restart Claude Desktop, then sign in when prompted.

### Cursor / Gemini CLI / other MCP clients

Add a remote MCP entry pointing at `https://api.anyways.eu/mcp/`. The client runs the OAuth flow on first connection.

## How authentication works

1. The client connects to `https://api.anyways.eu/mcp/`.
2. The server returns a 401 with an OAuth protected-resource metadata document ([RFC 9728](https://datatracker.ietf.org/doc/rfc9728/)) pointing at the ANYWAYS identity provider.
3. The client opens a browser to `https://www.anyways.eu/account/` for the user to sign in and consent.
4. The resulting access token is sent on every subsequent call. The MCP server forwards it as-is to the underlying ANYWAYS APIs, so the user's existing organization and project permissions apply.

Tokens are stored and refreshed by the MCP client, not by the server. To switch accounts, disconnect the server in the client and reconnect — the client will run the OAuth flow again.

## What tools become available

Once connected, the client sees ~30 tools across these groups:

- **Organizations / users** — `list_organizations`, `search_organizations`, `get_organization`, `list_organization_members`, `search_users`.
- **Projects** — `list_projects`, `get_project`, `create_project`, `copy_project`, `get_snapshot_commit`.
- **Datasets** — `create_dataset`, `get_dataset`, `update_dataset`, `delete_dataset`, `download_dataset`, `link_dataset_to_scenario`, plus type-specific creators (`create_locations_dataset`, `create_counter_dataset`), `add_dataset_locations` / `delete_dataset_locations`, `add_dataset_trips`, `add_counter_dataset_locations`, `generate_counter_segments`.
- **Routing utilities** — `list_profiles`, `snap_to_road`.
- **Traffic counts** — `list_traffic_count_providers`, `find_traffic_counters`, `get_traffic_counter`, `get_traffic_counter_aggregated`, `get_traffic_counter_hourly`. See the `anyways-traffic-counts` skill.

See the `anyways-projects`, `anyways-datasets`, and `anyways-traffic-counts` skills for how to use these together.

## Troubleshooting

**Client shows the server connected but no tools.**
The OAuth sign-in did not complete. Disconnect the server in the client and reconnect; complete the browser flow.

**403 from a tool.**
The signed-in user has no access to the targeted organization, project, or dataset. Verify access in the ANYWAYS app at https://www.anyways.eu/app/ first.

**401 after working for a while.**
Token expired and refresh failed. Disconnect and reconnect to re-run OAuth.

**Tool not found / stale tool list.**
The client cached an older tool list. Reconnect the server.

**Switching accounts.**
Disconnect the server in the MCP client, then reconnect — the OAuth flow will start fresh.

## Related links

- ANYWAYS app: https://www.anyways.eu/app/
- Documentation: https://docs.anyways.eu
- Contact / support: https://www.anyways.eu/contact
