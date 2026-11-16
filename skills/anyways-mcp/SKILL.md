---
name: anyways-mcp
description: >-
  Use when the user wants to add or troubleshoot the ANYWAYS MCP server in
  Claude Code, Claude Desktop, Cursor, Gemini CLI, or any other MCP-aware
  client. Also use when they mention "ANYWAYS MCP", api.anyways.eu/impact/mcp,
  ANYWAYS sign-in, or the OAuth flow against www.anyways.eu/account, or hit
  "does not support dynamic client registration" connecting to ANYWAYS.
---

# ANYWAYS MCP — Setup and Authentication

The ANYWAYS MCP server exposes the ANYWAYS platform — organizations, projects, networks, scenarios and datasets — as tools an MCP-aware client can call. It runs inside the ANYWAYS API itself, so a tool call carries the user's own permissions: anything the tools can see or change is something that user could see or change in the app.

```
Server URL:  https://api.anyways.eu/impact/mcp
Transport:   HTTP (Streamable HTTP)
Auth:        OAuth 2.1 + PKCE, pre-registered public client
Identity:    https://www.anyways.eu/account
Client ID:   anyways-mcp
Redirect:    http://localhost:7331/callback
```

> The ANYWAYS identity server does **not** support dynamic client registration (RFC 7591).
> Every client must be configured with the `anyways-mcp` client ID and callback port 7331.
> A client left to register itself will fail — see [Adding the server](#adding-the-server).

There is also a staging copy at `https://api.anyways.eu/impact/canary/mcp`, which runs ahead of production. Same client ID, same flow. Use it only when asked to try something not yet released.

## When to use this skill

- User wants to add the ANYWAYS MCP server to their agent.
- Connecting fails with a dynamic client registration error.
- User signed in but no tools appear.
- A tool returns 401 / 403, or refuses saying the server cannot reach something.
- User asks how the ANYWAYS auth flow works, or how to switch accounts.

For *what to do* once connected, use `anyways-projects` and `anyways-datasets`.

## Prerequisites

- An ANYWAYS account with access to the organizations and projects in question. Sign in at https://www.anyways.eu/account.
- An MCP client that supports remote HTTP MCP servers with OAuth **and** lets you set the OAuth client ID.

## Adding the server

| Field | Value |
| --- | --- |
| Name | `anyways` (or any label) |
| Type / Transport | HTTP (Streamable HTTP) |
| URL | `https://api.anyways.eu/impact/mcp` |
| OAuth client ID | `anyways-mcp` |
| Client type | Public — PKCE `S256`, **no client secret** |
| Redirect URI | `http://localhost:7331/callback` |
| Scopes to request | `openid impact offline_access` |

The server advertises `impact` as the scope it requires. Request `offline_access` as well so the client can refresh rather than making the user sign in again.

`http://localhost:7331/callback` is the only registered redirect URI, so the port is not optional.

### Claude Code (CLI)

```bash
claude mcp add --transport http anyways https://api.anyways.eu/impact/mcp \
  --client-id anyways-mcp \
  --callback-port 7331 \
  -s user
```

Then start a **new session** and run `/mcp`. On first use a browser opens to the ANYWAYS sign-in page; after authorising, the tools appear.

Both flags are required. Without `--client-id` the client tries to self-register and fails; without `--callback-port` it picks a port that is not registered.

Prefer `-s user` over `-s local`: a local registration is keyed to the exact directory it was run from, which is easy to get wrong when the session's working directory is a subdirectory of the project.

### Claude Desktop

`Settings → Connectors → Add custom connector`. Name `anyways`, URL `https://api.anyways.eu/impact/mcp`. Restart, then sign in when prompted. If it fails with a registration error, look for an OAuth client ID field and set `anyways-mcp`; if the dialog has none, use a client that allows a pre-registered client.

### Cursor / Gemini CLI / other clients

Add a remote MCP entry for the URL above and set the client ID, redirect URI and scopes from the table. A client that only supports dynamic registration cannot connect.

## How authentication works

1. The client POSTs to `https://api.anyways.eu/impact/mcp`.
2. The server answers **401** with `WWW-Authenticate: Bearer resource_metadata="https://api.anyways.eu/impact/.well-known/oauth-protected-resource/mcp"` ([RFC 9728](https://datatracker.ietf.org/doc/rfc9728/)).
3. That document names the authorization server: `https://www.anyways.eu/account`.
4. The client opens a browser there, using the `anyways-mcp` client ID and PKCE.
5. The access token is sent on every later call, and the server applies the user's own organization and project permissions.

Two facts about the identity provider, both measured, that explain most setup failures:

- Only `https://www.anyways.eu/account/.well-known/oauth-authorization-server` resolves — the path-insertion form 404s. Clients that *append* the discovery path work.
- The issuer it returns has **no trailing slash**. An authority configured with one fails twice: a double-slash 404 on discovery, and an issuer mismatch under RFC 8414 §3.3.

The metadata advertises no `registration_endpoint`, which is why a client must be told to use `anyways-mcp`. Everything else in the chain resolves normally — which is why the failure looks like a server misconfiguration when it is a missing client setting.

Tokens are held and refreshed by the client, not the server. To switch accounts, disconnect and reconnect.

## What tools become available

18 tools. Names and shapes are authoritative from the server — read the connected tool list rather than this table if the two disagree.

| Group | Tools |
| --- | --- |
| Context | `whoami` — the signed-in user and their organizations |
| Projects | `list_projects`, `get_project`, `create_project` |
| Networks | `copy_network`, `create_network`, `find_roads`, `find_counter_segment` |
| Scenarios | `create_scenario`, `get_scenario`, `add_dataset_to_scenario`, `remove_dataset_from_scenario` |
| Datasets | `create_abm_dataset`, `get_abm_dataset`, `create_locations_dataset`, `create_counters_dataset`, `add_counters`, `get_counters_dataset` |

Organizations come from `whoami`. There are no separate organization tools, no project copying, and no traffic-counter discovery — see `anyways-traffic-counts`.

## Troubleshooting

**"Incompatible auth server: does not support dynamic client registration".**
Added without an OAuth client ID, so the client tried to self-register. Re-add it:

```bash
claude mcp remove anyways -s user
claude mcp add --transport http anyways https://api.anyways.eu/impact/mcp \
  --client-id anyways-mcp --callback-port 7331 -s user
```

Nothing needs changing on the ANYWAYS side.

**Sign-in replaced by an ANYWAYS error page (`/account/home/error?errorId=…`).**
The callback port does not match the registered redirect URI; it must be exactly `http://localhost:7331/callback`. The page does not say which parameter was rejected, and a mistyped client ID looks identical — check both.

**Connected, but no tools.**
The OAuth flow did not complete. Disconnect, reconnect, and finish the browser flow.

**A new tool does not appear, or a tool rejects arguments that match its documented shape.**
Tool definitions are captured once, when the server attaches at session start, and are never re-fetched. A server deployed mid-session changes its *behaviour* immediately but not its *tool list* — so a renamed parameter or a brand-new tool is invisible until a **new session**. Start one before concluding anything is broken.

**A tool refuses saying the server cannot reach the publish API, the model, or the random-locations service.**
Not a client problem. Those tools depend on server-side configuration that varies by deployment, and they refuse with a message instead of failing obscurely. Report what it said; it names the setting.

**403 from a tool.**
The signed-in user has no access to that organization, project or dataset. Check in the app at https://www.anyways.eu/app first.

**401 after working for a while.**
The token expired and refresh failed. Disconnect and reconnect.

## Related links

- ANYWAYS app: https://www.anyways.eu/app
- Documentation: https://docs.anyways.eu
- Contact: https://www.anyways.eu/contact
