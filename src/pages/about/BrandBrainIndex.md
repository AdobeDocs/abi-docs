# Adobe Brand Intelligence

Adobe Brand Intelligence System(ABIS) is a standalone offering that enables

* enterprises to capture both explicit and implicit brand guidelines
* validate how the brand shows up across channels
* and instruct automated assembly using creative apps
* predict engagement before activation

## Base URLs

| Environment | URL                          |
|-------------|------------------------------|
| Dev         | `https://abi-dev.adobe.io`   |
| Stage       | `https://abi-stage.adobe.io` |
| Prod  [index.md](index.md)      | `https://abi.adobe.io`       |

## Authentication

API key, IMS token.

* **FDE Interface**: protected by Service Token
* **Skill Interface**: protected by IMS User Token

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
