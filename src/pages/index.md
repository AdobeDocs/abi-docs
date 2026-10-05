---
title: Adobe Brand Intelligence APIs
description: Validate creative assets against brand guidelines and simulate how they will land with an audience using Adobe Brand Intelligence.
contributors:
  - https://github.com/Aeabreu-hub
---

<SuperHero slots="heading, text" background="rgb(194, 100, 29)"/>

# Adobe Brand Intelligence APIs

Validate creative assets against brand guidelines at scale, and simulate how they will land with your audience before they go live.

## Overview

Adobe Brand Intelligence is an AI-powered service that helps enterprises get creative assets right before they are published. It offers two capabilities:

- **Validate** checks creative assets - designs, images, documents, and layouts - against your organization's brand guidelines and campaign-specific rules. See [Validate](validate/index.md).
- **Simulate** tests how creative assets will land with a synthetic audience built from your organization's populations and segments. See [Simulate](simulate/index.md).

Use Brand Intelligence to:

- **Validate assets in bulk** - submit a batch of assets and receive structured pass/fail feedback per asset.
- **Manage review feedback** - attach structured violations to flagged assets and track reviewer acceptance or rejection. See [Review Feedback](validate/review-feedback/index.md).
- **Simulate audience response** - test one or more creatives against an audience segment and receive an executive summary of how each performed. See [Simulate Using MCP](simulate/using-mcp/index.md).

## Discover

<DiscoverBlock slots="heading, link, text"/>

### Validate

[Core Concepts](validate/core-concepts/index.md)

Understand the async invocation model, the Invocation → Items → Violations resource hierarchy, and how validation results are structured.

<DiscoverBlock slots="link, text"/>

[Authentication](validate/authentication/index.md)

Set up OAuth Server-to-Server credentials in Adobe Developer Console and generate your first access token.

<DiscoverBlock slots="link, text"/>

[Using curl](validate/using-curl/index.md)

Submit your first validation invocation and retrieve results with step-by-step `curl` examples.

<DiscoverBlock slots="heading, link, text"/>

### Simulate

[Simulate Core Concepts](simulate/core-concepts/index.md)

Understand workspaces, templates, audiences, asset types, and the simulation lifecycle.

<DiscoverBlock slots="link, text"/>

[Simulate Using MCP](simulate/using-mcp/index.md)

Connect your chat assistant to the Simulate MCP server and run your first simulation.

<DiscoverBlock slots="heading, link, text"/>

### API Reference

[Brand Intelligence API](validate/api/rest/index.md)

Full OpenAPI reference for the Validation endpoints.

<DiscoverBlock slots="link, text"/>

[Simulate MCP Tools](simulate/api/mcp/index.md)

Request and response schemas for the Simulate MCP tools.
