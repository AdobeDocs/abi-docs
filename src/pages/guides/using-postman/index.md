---
title: Using Postman - Brand Intelligence
description: Explore and test the Adobe Brand Intelligence API using Postman.
---

# Using Postman

Postman is a convenient way to explore the Brand Intelligence API without writing code. This guide walks through setting up an environment, generating a token, and running a validation invocation.

## Prerequisites

- [Postman](https://www.postman.com/downloads/) installed.
- OAuth Server-to-Server credentials from Adobe Developer Console (see [Authentication](../authentication/index.md)).


## Step 1 - Set up a Postman environment

Create a new environment in Postman with the following variables:

| Variable | Initial value | Description |
|----------|---------------|-------------|
| `base_url` | `https://abi-mw.adobe.io` | Brand Intelligence API base URL |
| `ims_url` | `https://ims-na1.adobelogin.com` | Adobe IMS token endpoint base URL |
| `client_id` | *(your Client ID)* | From Adobe Developer Console |
| `client_secret` | *(your Client Secret)* | From Adobe Developer Console - mark as **secret** |
| `access_token` | *(leave blank)* | Populated automatically in Step 2 |
| `tenant_id` | *(your Tenant ID)* | Required on every invocation |
| `invocation_id` | *(leave blank)* | Populated as you work through invocations |

<InlineAlert variant="warning" slots="text"/>

Mark `client_secret` as a **secret** variable in Postman so it is not synced to the cloud or shared with others.


## Step 2 - Generate an access token

Create a new **POST** request:

- **URL:** `{{ims_url}}/ims/token/v3`
- **Body** (x-www-form-urlencoded):

| Key | Value |
|-----|-------|
| `grant_type` | `client_credentials` |
| `client_id` | `{{client_id}}` |
| `client_secret` | `{{client_secret}}` |
| `scope` | `openid,AdobeID` |

To automatically save the token to your environment, add this script to the **Post-response** tab:

```javascript
const token = pm.response.json().access_token;
pm.environment.set("access_token", token);
```

Send the request. Postman will store the token in the `access_token` variable, ready for use in subsequent requests.

Tokens are valid for 24 hours. Re-run this request when your token expires.


## Step 3 - Import the OpenAPI spec (optional)

You can import the Brand Intelligence OpenAPI spec directly into Postman to auto-generate a request collection:

1. Open the [API Reference](../../api/index.md) page and use its **Download OpenAPI specification** link to save the spec file.
2. In Postman, select **Import** and choose the downloaded file.
3. Postman will generate a collection with all available endpoints pre-populated.
4. Set the collection's **Authorization** to **Bearer Token** and set the token value to `{{access_token}}`.


## Step 4 - Submit a validation invocation

Create a **POST** request:

- **URL:** `{{base_url}}/api/abi/skills/ra`
- **Authorization:** Bearer Token → `{{access_token}}`
- **Body** (raw JSON):

```json
{
  "tenantId": "{{tenant_id}}",
  "displayName": "My first validation invocation",
  "items": [
    {
      "itemSource": "blob",
      "sourceRef": "<itemId from uploading the asset>",
      "mediaType": "image/png",
      "itemName": "asset-01"
    }
  ]
}
```

<InlineAlert variant="info" slots="text"/>

`sourceRef` here is the `itemId` returned by `POST /api/abi/storage/temp` after you upload the asset - see [Quickstart: Step 2](../quickstart/index.md#step-2---upload-your-assets) for the full upload flow.

The response includes an `invocationId`. Copy it into your `invocation_id` environment variable.


## Step 5 - Poll and retrieve results

**Poll invocation status:**

- **GET** `{{base_url}}/api/abi/skills/ra/{{invocation_id}}`
- **Authorization:** Bearer Token → `{{access_token}}`

Repeat until `status` is `completed`, `failed`, or `cancelled`.

**Fetch results:**

- **GET** `{{base_url}}/api/abi/skills/ra/{{invocation_id}}/items`
- **Authorization:** Bearer Token → `{{access_token}}`

See the [Quickstart](../quickstart/index.md) for annotated example responses.
