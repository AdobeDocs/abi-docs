---
title: MCP Integration - Brand Intelligence
description: Integrate Adobe Brand Intelligence Validate into your own chat assistant via its MCP server.
---

# MCP Integration

This guide is for teams integrating **Adobe Brand Intelligence (ABI) Validate** into their own chat assistant via its MCP (Model Context Protocol) server.

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
| `get_validation_options` | Discover the asset input shape (allowed `source` values, `media_type` guidance, the bulk `assets` list shape). | none | `source_types`, notes on media type / bulk shape |
| `open_validation_asset_upload` | Opens an inline drag-and-drop upload panel — **only relevant if your host renders MCP-Apps UI panels; see Section 4.** | none | `message`, `upload_session_id` |
| `get_validation_upload_status` | Poll target for the upload panel's completion. Call this yourself after `open_validation_asset_upload`, repeatedly, until `settled` is `true`. | `upload_session_id` | `settled: bool`, `assets: [{asset_id, media_type, asset_name}]` |
| `prepare_validation_asset_upload` | Headless upload path for a local file whose path you already have: mints a presigned upload target + a ready-to-run upload command. | `local_path`, `content_type` | `asset_id`, `upload_command` (run it yourself, confirm exit code 0) |
| `validate_asset` | Starts **one bulk invocation** covering one or more assets. | `assets: [{source: "web", media_type, value, asset_name} \| {source: "blob", asset_id, media_type, asset_name}, ...]`, optional `campaign_id` | `invocation_id`, `item_count` |
| `get_validation_status` | Cheap poll target for an in-flight invocation — overall status plus a per-item status summary, no violations yet. | `invocation_id` | `status`, `item_count`, `success_count`, `failure_count`, per-item `items[]` |
| `get_validation_result` | Full per-item report once the invocation is terminal — summary + every violation. Call once. | `invocation_id` | per-item `items[]`, each with `status`, `summary`, `violations[]`, `asset_url`/`asset_data_url` |

`upload_validation_asset_bytes` and `finalize_validation_upload_session` are
internal tools the upload panel itself calls — you won't call these directly
unless you're also implementing an MCP-Apps-compatible upload panel of your
own.

## 3. How to drive the tools correctly

This is the actual behavioral contract — adapt the wording to your own
system prompt as needed, but keep the substance:

- **Bulk by design.** `validate_asset` takes a **list** of assets and starts
  ONE invocation covering all of them. If the user gives you 3 creatives,
  that's one `validate_asset` call with 3 entries in `assets` — never three
  separate calls. Results come back per item, but the invocation itself is
  one thing to start and poll.
- **The validation result is the answer — never your own opinion.** Do not
  judge whether an asset is on-brand or compliant from your own read of it.
  If you haven't gotten a result from `get_validation_result`, you don't have
  an answer yet.
- **Never auto-fetch an asset.** Every asset comes only from what the user
  explicitly hands you (a URL, a presigned URL, or a local file). Don't go
  looking for one elsewhere — don't browse a filesystem, an inbox, or a
  connected drive on your own.
- **Get the asset(s).** If the user already gave a URL, use it directly. For
  a local file whose exact path you have, call
  `prepare_validation_asset_upload`. Otherwise (if your host supports the
  upload panel — see Section 4) open it immediately rather than asking a
  text question first.
- **Call `validate_asset` once** with every asset gathered so far — no
  per-asset calls, no scope question first. Omit `campaign_id` unless the
  user volunteered one; most tenants auto-resolve a default.
- **Poll for the result, silently.** After starting the invocation, call
  `get_validation_status(invocation_id)` again yourself, repeatedly, until
  `status` is `completed` or `failed` — don't narrate each check to the
  user. **"Poll" means literally calling the tool again** — this is easy to
  get wrong.
- **Read the result exactly once.** Once terminal, call
  `get_validation_result(invocation_id)` exactly once. That call ends
  polling for this invocation — don't schedule or make any further status or
  result calls for it afterward.
- **Report every item's result as given, not as interpreted.** For each
  item, share its `summary` and every entry in `violations` (each with
  `violation_summary`, `severity`, `guideline_section`, `focus_area`). An
  empty `violations` list means no issues found — say so plainly. Report
  per-item outcomes clearly when several assets were checked; don't collapse
  them into one blended verdict, and don't soften or add your own take on
  top of what the tool returned.

## 4. UI panels: confirm support before relying on `open_*` tools

`open_validation_asset_upload` opens an inline drag-and-drop panel via the
`io.modelcontextprotocol/ui` MCP extension. **Most custom-built agent
frameworks do not implement this extension.** If yours doesn't:

- Skip `open_validation_asset_upload` and `get_validation_upload_status`
  entirely.
- Route every local-file upload through `prepare_validation_asset_upload`
  instead — it returns a ready-to-run upload command with no UI dependency.

If you're unsure whether your framework supports it, assume it doesn't. A
host that calls `open_validation_asset_upload` without panel-rendering
support just gets back a text message pointing at the
`prepare_validation_asset_upload` fallback — safe, but redundant if you
route directly.

## 5. Reference

The [MCP Tools Reference](../../api/mcp/index.md) is a machine-readable OpenAPI
3.1 description of the 7 Validate tools' request/response schemas — useful
for validating your own integration's shapes against the real contract. Its
per-tool `POST /tools/<name>` paths are a documentation convention only, not
callable routes — live traffic goes over MCP JSON-RPC at `/mcp`, not REST.

Note: `upload_validation_asset_bytes` and `finalize_validation_upload_session`
are intentionally absent from the OpenAPI — they are internal tools called by
the MCP-Apps upload panel itself and are not part of the integrator-facing
surface.
