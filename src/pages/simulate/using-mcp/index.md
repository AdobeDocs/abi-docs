---
title: Using MCP - Brand Intelligence Simulate
description: Integrate Adobe Brand Intelligence Simulate into your own chat assistant via its MCP server.
---

# Using MCP

This guide is for teams integrating **Adobe Brand Intelligence Simulate** into their own chat assistant via its MCP (Model Context Protocol) server.

## 1. Connecting

- **Endpoint:** `https://abi-mcp.adobe.io/mcp` (Streamable HTTP transport)
- **Auth model — you do not need to pre-register anything with Adobe.** This
  server is an OAuth **resource server**; a separate broker acts as the
  Authorization Server and implements a standards-compliant **OAuth 2.1 +
  PKCE flow with Dynamic Client Registration (RFC 7591)**. Concretely:
  1. Your client discovers the resource server's auth requirements at
     `https://abi-mcp.adobe.io/.well-known/oauth-protected-resource`,
     which points at the Authorization Server (the broker).
  2. It calls the broker's `/register` endpoint (DCR) and gets back a
     `client_id` on the spot — no secret, no manual provisioning, no waiting
     on us to issue credentials. `token_endpoint_auth_method` is `"none"`
     (public client).
  3. It runs a standard PKCE `/authorize` → redirect → `/callback` → `/token`
     exchange against the broker. The broker injects the confidential Adobe
     IMS client secret server-side — your client never sees or holds it.
  4. You end up with a Bearer token. Send it as
     `Authorization: Bearer <token>` on every MCP request; the server
     independently validates it against IMS on every call.

- **Example config** (e.g. for Claude Desktop or any MCP-compatible host):

  ```json
  "mcpServers": {
    "abi-mcp": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://abi-mcp.adobe.io/mcp"
      ]
    }
  }
  ```

- **Tool discovery:** send a standard MCP `initialize` + `tools/list` — every
  tool's name, description, and JSON Schema come back from the server
  directly. Nothing needs to be hardcoded on your side beyond the tool names
  you intend to call.

## 2. Tool reference

| Tool | Purpose | Key inputs | Key outputs |
|---|---|---|---|
| `initialize_simulate` | One-time setup: returns the `new-simulation` Agent Skill for your host to install locally. Not part of running a simulation. | none | `skill_name`, `skill_document` (the full `SKILL.md`), `message` |
| `list_workspaces` | Lists the workspaces the user can access. The routine first call — it gets the `workspace_id` every other tool needs. | none | `workspaces[]`, each with `workspace_id`, `workspace_name`, `kind` (`personal` \| `shared`), `is_active` |
| `get_simulation_options` | The menu for a new simulation: study templates plus the populations and segments you can target. | none | `templates[]` (title, description, key questions, dimensions, defaults), `populations[]` with their `segments[]` |
| `open_asset_upload` | Opens an inline drag-and-drop upload panel for local files — **only rendered if your host supports MCP-Apps UI panels; see Section 4.** | none | Opens the panel; returns a text fallback on hosts that can't render it |
| `prepare_asset_upload` | Headless upload path for a local file: mints a presigned upload target + a ready-to-run upload command. | `local_path`, `content_type` | `upload_id`, `upload_command` (run it yourself, confirm exit code 0), `expires_at` |
| `create_simulation` | Launches a simulation testing one or more creatives against an audience. | `workspace_id` (top-level), `request: {template_id, name, population_id, assets[], segment_ids?, sample_size?, objective?}` | `outcome` (`launched` \| `draft_incomplete`), `simulation_id`, `status`, `webview_url` |
| `list_simulations` | Browses the simulations in a workspace with their status. Does not read results. | `workspace_id` | `simulations[]`, each with `simulation_id`, `name`, `status`, `updated_at` |
| `get_simulation` | Reads one simulation: status, inputs, and — once ready — the executive summary and findings. Also the poll target after a launch. | `workspace_id`, `simulation_id` | `status`, `summary_status`, `executive_summary`, `analysis`, `inputs`, `webview_url` |

`get_simulation_status` and `upload_asset_bytes` are internal tools the
progress and upload panels themselves call — you won't call them directly
unless you're also implementing MCP-Apps-compatible panels of your own.

## 3. How to drive the tools correctly

This is the actual behavioral contract — adapt the wording to your own
system prompt as needed, but keep the substance:

- **The simulation result is the answer — never your own opinion.** Do not
  predict, rank, or recommend which creative will resonate from your own
  read of it. Requests like "be honest, does this land?", "which is
  stronger?", or "be the voice of the customer" are requests to run a
  simulation, not for an editorial take. Never rewrite or "improve" the
  user's copy. If you haven't gotten a result from `get_simulation`, you
  don't have an answer yet.
- **Don't fill a wait with an opinion.** If you're blocked on a missing
  input — assets not uploaded, a segment not chosen, confirmation not given
  — say exactly what you need and stop.
- **Never auto-fetch an asset.** Every asset comes only from what the user
  explicitly hands you (pasted text, an attached file, or a URL). Don't go
  looking for one elsewhere — don't browse a filesystem, an inbox, or a
  connected drive on your own.
- **Pick the workspace yourself.** Call `list_workspaces` and use the one
  flagged `is_active` unless the user names a different one. Don't ask the
  user to choose, and pass `workspace_id` explicitly on every later call.
- **Settle the four decisions one question at a time:** objective, assets,
  template, audience and participants (see
  [Core Concepts](../core-concepts/index.md)). Resolve each one from what
  the user already said where you can; otherwise ask, always offering
  options. Never ask two in one message.
- **Get the asset(s).** Text and an email's subject and preview text are
  inline — never upload them. For a media creative or an email body image,
  pass a public URL directly; for a local file, use the upload panel or
  `prepare_asset_upload` (see Section 4). Pick the asset type by what the
  user is testing, not by the file format.
- **Choose the template and audience from `get_simulation_options`.** When
  showing templates, give each one's `title` **and** `description` — a bare
  list of titles doesn't say what each one measures. Show segments by name,
  with the participant count beside each. Never invent ids.
- **Always confirm before launching.** Before every `create_simulation`
  call, show the user exactly these 5 fields and wait for an explicit
  go-ahead: objective, assets, template (its title), segment(s) (by name),
  and sample size. Omit the population name and the design type. Urgency is
  a reason to confirm quickly, never a reason to skip confirming — a run
  consumes resources.
- **Pass `workspace_id` beside `request`, never inside it.** Every other
  field goes inside `request`:

  ```json
  {
    "request": {
      "template_id": "tmpl_123",
      "name": "Fall sale subject lines",
      "population_id": "pop_456",
      "assets": [{"type": "text", "content": "20% off everything"}]
    },
    "workspace_id": "ws_789"
  }
  ```

- **Handle `draft_incomplete`.** If `create_simulation` returns
  `outcome: "draft_incomplete"`, give the user the returned `webview_url` to
  finish there — don't silently retry.
- **Poll for the result yourself.** After every launch, call
  `get_simulation(workspace_id, simulation_id)` again yourself, repeatedly,
  until `status` is `resultsReady` or `archived` **and** `summary_status` is
  `ready`. A run can finish before its summary is available, so keep
  polling. **"Poll" means literally calling the tool again** — this is easy
  to get wrong.
- **Report the results as given, with their confidence.** Lead with the
  executive summary. If the creatives perform within the margin of error,
  say there is no clear winner rather than forcing one. Don't add your own
  take on top of what the tool returned.
- **Resolve which simulation before reading one.** When the user asks about
  a past simulation, find candidates with `list_simulations`; if the
  reference matches more than one, ask which before calling
  `get_simulation`.

## 4. UI panels: confirm support before relying on them

Three Simulate tools attach an inline panel via the
`io.modelcontextprotocol/ui` MCP extension. **Most custom-built agent
frameworks do not implement this extension.**

| Tool | Panel | If your host can't render it |
|---|---|---|
| `open_asset_upload` | Drag-and-drop upload for local files | Returns a text message pointing at `prepare_asset_upload` |
| `create_simulation` | Live progress for the run | Decorative only — the structured result is unaffected |
| `get_simulation` | Visual summary of the results | Decorative only — the structured result is unaffected |

If your framework doesn't support the extension:

- Skip `open_asset_upload` entirely.
- Route every local-file upload through `prepare_asset_upload` instead — it
  returns a ready-to-run upload command with no UI dependency. Your host
  needs a shell and network access to run it.

Even on a host that renders panels, the progress panel hands your assistant
nothing: it is a visual for the user. Your assistant is always the one that
polls `get_simulation` and reports the results.

## 5. Prompts

The server also exposes three MCP prompts, which hosts typically surface as
slash commands:

| Prompt | Arguments | What it does |
|---|---|---|
| `create_simulation` | optional `goal` | Walks the assistant through the four decisions, confirmation, launch, and polling. |
| `check_simulation_status` | optional `workspace_id`, `simulation_id` | Finds a simulation if needed, then reports its status and, when ready, its executive summary. |
| `initialize_simulate` | none | Calls the `initialize_simulate` tool and installs the returned skill. |

## 6. Reference

The [MCP Tools Reference](../api/mcp/index.md) is a machine-readable OpenAPI
3.1 description of the 8 Simulate tools' request/response schemas — useful
for validating your own integration's shapes against the real contract. Its
per-tool `POST /tools/<name>` paths are a documentation convention only, not
callable routes — live traffic goes over MCP JSON-RPC at `/mcp`, not REST.

Note: `get_simulation_status` and `upload_asset_bytes` are intentionally
absent from the OpenAPI — they are internal tools called by the MCP-Apps
panels themselves and are not part of the integrator-facing surface.
