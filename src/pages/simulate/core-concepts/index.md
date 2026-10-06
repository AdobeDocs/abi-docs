---
title: Core Concepts - Brand Intelligence Simulate
description: Key concepts for running audience simulations with Adobe Brand Intelligence.
---

# Core Concepts

## What Simulate does

Adobe Brand Intelligence Simulate predicts how creative assets will land with an audience. You supply one or more creatives - a headline, an email, an image, a video - and Brand Intelligence runs them past a synthetic audience drawn from your organization's populations and segments. When the run finishes, it returns an executive summary and detailed findings for each creative.

A simulation replaces a guess about which creative will resonate with an answer from the audience. When you integrate Simulate into an assistant, the simulation result is the answer: the assistant relays it and does not add its own opinion of the creatives.


## Resource model

```
Workspace
└── Simulation (one study: objective, template, audience, assets)
    └── Run (one execution of the simulation)
        └── Executive summary + findings
```

**Workspace** - a container for simulations. A user can belong to several workspaces, each either `personal` or `shared`. One is flagged `is_active`: the workspace the user currently has open in the Brand Intelligence web app. Every simulation belongs to exactly one workspace.

**Simulation** - one study testing a set of creatives against an audience. It records the objective, the template, the target population and segments, the sample size, and the assets.

**Run** - one execution of a simulation. A simulation's reported status follows its latest run when one exists.


## The four decisions

Creating a simulation means settling four decisions, in this order:

1. **Objective** - what the simulation should find out, for example "which of these two subject lines drives more opens".
2. **Assets** - the creatives to test.
3. **Template** - the study template to run.
4. **Audience and participants** - which segment or segments to target, and how many simulated respondents.

The objective and the assets come from the user. The template and audience options come from `get_simulation_options`, which lists what your organization has available.


## Templates

A template defines what a simulation measures. Each template returned by `get_simulation_options` carries:

- a `title` and a `description` of what it measures
- its `key_questions` and scored `dimensions`
- the design types it allows
- form defaults: `default_name`, `default_objective`, `default_design_type`, and `default_sample_size`

When the request omits a field, the template's default applies.

**Design type** follows from the number of assets. A comparative design (`sequential_monadic`) needs at least two assets; with only one asset, the server runs a `monadic` design instead.


## Audiences

Audiences come from your organization's **populations**. Each population contains **segments**, such as "Existing customers" or "Gen Z". A simulation targets one population and, optionally, one or more of its segments. An empty segment list targets the whole population.

**Sample size** is the number of simulated respondents. When omitted, the template's `default_sample_size` applies.


## Asset types

Each asset is a flat object tagged by `type`. The type follows what you are testing, not the format of the file you have.

| Type | Use for | Content |
|------|---------|---------|
| `text` | A message, headline, standalone subject line, tagline, product name, or messaging option. | Inline `content`. No upload, no URL. |
| `email` | Any email, including a subject line and preview text tested together. | Inline `subject` (required), optional `preview_text`, optional `body_image`. |
| `image`, `video`, `html`, `audio` | A media creative. | An `asset` whose bytes come from a URL or an upload. |

When you are testing emails and have an image of the email body, that image is the email's `body_image`, not a standalone `image` asset.

`label` is optional on every asset. Unlabeled assets are numbered "Option 1", "Option 2", and so on, by their position in the list.


## Asset sources

Only media assets and email body images carry bytes. Each declares a `source` telling Brand Intelligence where to read them from:

- **`url`** - the file is already reachable at a public HTTPS URL. Pass the URL directly; no upload step is needed.
- **`upload`** - the file is local. Stage it first with `open_asset_upload` (an inline upload panel) or `prepare_asset_upload` (a ready-to-run upload command), then reference the returned `upload_id`.

Supplying both a URL and an `upload_id`, or neither, is rejected. See [Using MCP](../using-mcp/index.md) for when to use each upload path.


## Simulation lifecycle

`create_simulation` returns one of two outcomes:

| Outcome | Meaning |
|---------|---------|
| `launched` | The simulation was created and a run started. The result includes the `simulation_id` and a `webview_url`. |
| `draft_incomplete` | The simulation was created, but a later setup step failed. `failed_step` names the step and `detail` explains it. Open the returned `webview_url` to finish or retry the draft in the Brand Intelligence web app. |

Simulations are asynchronous. After a launch, poll `get_simulation` until both of these are true:

- `status` is `resultsReady` or `archived`, and
- `summary_status` is `ready`.

`summary_status` tracks the executive summary separately from the run:

| `summary_status` | Meaning |
|------------------|---------|
| `not_ready` | No finished run yet. |
| `pending` | The run finished, but the summary is not available yet. Retry shortly. |
| `ready` | The executive summary is included in the response. |

A run can reach `resultsReady` while its summary is still `pending`, so keep polling until both conditions hold.


## Results

Once ready, `get_simulation` returns:

- **`executive_summary`** - a `summary` of how the creatives performed, plus `recommendations`.
- **`analysis`** - the detailed findings.
- **`inputs`** - what the simulation was configured with: objective, design type, participants, population, segments, dimensions, and assets.
- **`webview_url`** - a link to open the simulation in the Brand Intelligence web app.

Report results with their confidence. If the creatives perform within the margin of error, the honest answer is that there is no clear winner.


## Authentication

Every MCP request requires a Bearer token. MCP clients obtain one through an interactive OAuth sign-in; see [Using MCP](../using-mcp/index.md). The tenant is resolved server-side from the caller's Adobe organization, so no tool takes a tenant id.
