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
| `base_url` | `https://brand-intelligence.adobe.com` | Brand Intelligence API base URL |
| `ims_url` | `https://ims-na1.adobelogin.com` | Adobe IMS token endpoint base URL |
| `client_id` | *(your Client ID)* | From Adobe Developer Console |
| `client_secret` | *(your Client Secret)* | From Adobe Developer Console - mark as **secret** |
| `access_token` | *(leave blank)* | Populated automatically in Step 2 |
| `upload_id` | *(leave blank)* | Populated in Step 4 for each uploaded asset |
| `invocation_id` | *(leave blank)* | Populated as you work through invocations |
| `item_id` | *(leave blank)* | Returned by the results endpoint in Step 6 |

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

1. Open the [API Reference](../api/rest/index.md) page and use its **Download OpenAPI specification** link to save the spec file.
2. In Postman, select **Import** and choose the downloaded file.
3. Postman will generate a collection with all available endpoints pre-populated.
4. Set the collection's **Authorization** to **Bearer Token** and set the token value to `{{access_token}}`.


For imported requests, also check the generated `authorization` header: set its value to `Bearer {{access_token}}` or remove it so the collection authorization supplies the header. Do not leave a generated placeholder that overrides your token.


## Step 4 - Discover context and upload your asset

Tenant and permissions are resolved from your token. Send **GET** `{{base_url}}/index` with Bearer Token authorization to discover your tenant's ID and name. You do not send a `tenantId` in upload or validation bodies.

Send **GET** `{{base_url}}/campaigns?skill=validate` with the same authorization to list validation campaigns. Each entry in `items` includes an `id` you can use as the optional `campaignId` in Step 5. When `page.next` is present and non-null, pass it as the `start` query parameter to fetch the next page.

Brand Intelligence validates uploaded assets only. Upload each asset first and reference it by the returned `uploadId` when you submit the invocation.

**Request an upload slot.** Create a **POST** request:

- **URL:** `{{base_url}}/storage/temp`
- **Authorization:** Bearer Token → `{{access_token}}`
- **Body:** none

The response (`201 Created`) includes an `uploadId`, an `uploadUrl`, and `uploadUrlExpiresAt`. Copy the `uploadId` into your `upload_id` environment variable.

**Upload the bytes.** Create a **PUT** request:

- **URL:** the `uploadUrl` from the previous response
- **Authorization:** No Auth (the URL already carries a SAS token)
- **Headers:** `x-ms-blob-type: BlockBlob`
- **Body:** **binary**, then select your asset file

`uploadUrl` expires one hour after it's issued - upload promptly. Repeat this step once per asset.

<InlineAlert variant="info" slots="text"/>

The REST validation endpoint accepts only `itemSource: "upload"`. Public asset URLs and the old `blob` and `web` source values are not supported.


## Step 5 - Submit a validation invocation

Create a **POST** request:

- **URL:** `{{base_url}}/skills/validate`
- **Authorization:** Bearer Token → `{{access_token}}`
- **Body** (raw JSON):

```json
{
  "displayName": "My first validation invocation",
  "items": [
    {
      "itemSource": "upload",
      "sourceRef": "{{upload_id}}",
      "mediaType": "image/png",
      "itemName": "asset-01"
    }
  ]
}
```

Set `itemSource` to `upload` and `sourceRef` to the `uploadId` from Step 4. You can optionally include `campaignId` from the campaign list or a `validationContext` such as `{"dimensions": {"market": ["us"]}}` at the invocation or item level.

The response (`202 Accepted`) includes an `invocationId` and `skill: "validate"`. Copy the `invocationId` into your `invocation_id` environment variable.


## Step 6 - Poll and retrieve results

**Poll invocation status:**

- **GET** `{{base_url}}/skills/validate/{{invocation_id}}`
- **Authorization:** Bearer Token → `{{access_token}}`

Repeat until `status` is `completed`, `failed`, or `cancelled`.

**Fetch results:**

- **GET** `{{base_url}}/skills/validate/{{invocation_id}}/items`
- **Authorization:** Bearer Token → `{{access_token}}`

Copy an item's returned `itemId` into `item_id`, then fetch its full detail with **GET** `{{base_url}}/skills/validate/{{invocation_id}}/items/{{item_id}}` using the same authorization. The detail includes `result.violations`, `result.passes`, and `result.rejections`. The result `itemId` is not the temporary `uploadId`.

To list your previous validation invocations, use **GET** `{{base_url}}/skills/invocations?skill=validate`. The `skill` query parameter is required. Use `page.next` as `start` to fetch more pages until it is absent or `null`.

See [Using curl](../using-curl/index.md) for annotated example responses and [Review Feedback](../review-feedback/index.md) for versioned feedback updates.
