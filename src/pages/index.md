---
title: Overview - Analytics
description: This is the overview page of Analytics
contributors:
  - https://github.com/icaraps 
---

<Superhero slots="heading, text" background="rgb(194, 158, 29)"/>

# Adobe Brand Intelligence

Explore the API documentation for Adobe Brand Intelligence

## Overview

Adobe Brand Intelligence System(ABIS) is a standalone offering that enables Enterprises to capture both explicit and implicit brand guidelines. Use this service to:

* Validate how the brand shows up across channels.
* Instruct automated assembly using creative apps.
* Predict engagement before activation.

## Base URLs

| Environment | URL                          |
|-------------|------------------------------|
| Dev         | `https://abi-dev.adobe.io`   |
| Stage       | `https://abi-stage.adobe.io` |
| Prod  [index.md](index.md)      | `https://abi.adobe.io`       |

## Authentication

The service is authenticated using API key, IMS token.

* **FDE Interface**: Protected by Service Token.
* **Skill Interface**: Protected by IMS User Token.

## APIs

| API             | Description                         | Auth          |
|-----------------|-------------------------------------|---------------|
| FDE Inteface    | Manages tenants and configurations  | Service Token |
| Skill Interface | Provides Headless API to use Skills | User Token    |
| Book keeping    | Get context related to campaigns    | User Token    |

Each API has its own detailed reference in the sidebar.

## Common Headers

| Header          | Required | Description                |
|-----------------|----------|----------------------------|
| `Authorization` | Yes      | Bearer token               |
| `x-api-key`     | Yes      | Client API key             |
| `x-request-id`  | No       | Correlation ID for tracing |
