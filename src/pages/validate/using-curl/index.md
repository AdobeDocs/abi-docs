---
title: Using curl - Brand Intelligence Validate
description: Submit your first asset validation invocation using the Adobe Brand Intelligence API.
---

# Using curl

This guide walks you through the complete validation flow - from getting an access token to reading per-asset results - using `curl`.

## Prerequisites

- Your organization's **brand profile** is configured. This is set up during Adobe Brand Intelligence onboarding - contact your Adobe representative if this has not been completed.
- OAuth Server-to-Server credentials set up in Adobe Developer Console (see [Authentication](../authentication/index.md)).
- Your **Client ID** and **Client Secret** available in a secure terminal session.
- One or more asset files ready to upload. The REST validation endpoint does not accept public asset URLs.


## Validation flow

The Brand Intelligence API is asynchronous. You submit an invocation, receive an `invocationId`, poll until the invocation completes, then fetch per-asset results.

All REST API requests use `https://brand-intelligence.adobe.com`:

```text
Application → Adobe IMS: POST /ims/token/v3 → access_token
For each asset:
  Application → Brand Intelligence: POST /storage/temp → uploadId, uploadUrl
  Application → Storage: PUT uploadUrl (asset bytes)
Application → Brand Intelligence: POST /skills/validate → invocationId
Application → Brand Intelligence: GET /skills/validate/{invocation_id} (poll)
Application → Brand Intelligence: GET /skills/validate/{invocation_id}/items
Application → Brand Intelligence: GET /skills/validate/{invocation_id}/items/{item_id}
```

## Step 1 - Get an access token

Export your credentials as environment variables:

```bash
export CLIENT_ID='<your_client_id>'
export CLIENT_SECRET='<your_client_secret>'
```

Request a token from Adobe IMS:

```bash
curl --location 'https://ims-na1.adobelogin.com/ims/token/v3' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" \
  --data-urlencode 'scope=openid,AdobeID'
```

Export the token for use in subsequent requests:

```bash
export ACCESS_TOKEN='<access_token_from_response>'
```

Tokens are valid for 24 hours (`expires_in: 86399`). Refresh before expiry to avoid interruption.


## Step 2 - Discover your tenant and campaigns

Tenant and permissions are resolved from your access token. To see the tenant available to you:

```bash
curl --request GET \
  --url 'https://brand-intelligence.adobe.com/index' \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

The response includes `tenantId` and `name`. You do not pass `tenantId` in subsequent request bodies.

To list validation campaigns:

```bash
curl --request GET \
  --url 'https://brand-intelligence.adobe.com/campaigns?skill=validate' \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

Each campaign in `items` includes an `id` and `name`. To select a campaign for validation, use its `id` as the optional `campaignId` in Step 4. If `page.next` is present and non-null, pass it as the `start` query parameter to fetch the next page.


## Step 3 - Upload your assets

Brand Intelligence validates uploaded assets. Upload each asset first and reference it by the returned `uploadId` when you submit the invocation.

For each asset, request an upload slot. This request has no body:

```bash
curl --request POST \
  --url 'https://brand-intelligence.adobe.com/storage/temp' \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

**Response** (`201 Created`):

```json
{
  "uploadId": "c3f1a2b0-0000-0000-0000-000000000001",
  "uploadUrl": "https://example.blob.core.windows.net/uploads/banner-01.png?sig=example",
  "uploadUrlExpiresAt": "2026-08-31T13:00:00Z"
}
```

Save the `uploadId`, then `PUT` the asset bytes directly to `uploadUrl`. For an Azure Blob SAS URL, include the `x-ms-blob-type` header. The signed URL authorizes the upload; do not send your IMS token to storage:

```bash
curl --request PUT \
  --upload-file banner-01.png \
  --header "x-ms-blob-type: BlockBlob" \
  --url "<uploadUrl_from_response>"
```

`uploadUrl` expires one hour after it's issued - upload before `uploadUrlExpiresAt`. Repeat this step once per asset. Save each `uploadId`:

```bash
export UPLOAD_ID_1=c3f1a2b0-0000-0000-0000-000000000001
export UPLOAD_ID_2=c3f1a2b0-0000-0000-0000-000000000002
```

## Step 4 - Submit a validation invocation

Submit one or more uploaded assets for brand validation. Each item requires an `itemSource`, a `sourceRef`, a `mediaType`, and an `itemName`. Set `itemSource` to `upload` and `sourceRef` to the `uploadId` from Step 3.

```bash
curl --request POST \
  --url 'https://brand-intelligence.adobe.com/skills/validate' \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "displayName": "Spring Campaign - Banner Set",
    "items": [
      {
        "itemSource": "upload",
        "sourceRef": "'"$UPLOAD_ID_1"'",
        "mediaType": "image/png",
        "itemName": "banner-01"
      },
      {
        "itemSource": "upload",
        "sourceRef": "'"$UPLOAD_ID_2"'",
        "mediaType": "image/png",
        "itemName": "banner-02"
      }
    ]
  }'
```

**Response** (`202 Accepted`):

```json
{
  "invocationId": "9b9d00c5-8659-4766-8430-ed0a1c9bd87d",
  "skill": "validate",
  "tenantId": "123e4567-e89b-12d3-a456-426614174003",
  "campaignId": "123e4567-e89b-12d3-a456-426614174004",
  "status": "queued",
  "itemCount": 2,
  "successCount": 0,
  "failureCount": 0
}
```

Save the `invocationId`:

```bash
export INVOCATION_ID=9b9d00c5-8659-4766-8430-ed0a1c9bd87d
```

<InlineAlert variant="info" slots="text"/>

`campaignId` is optional; add a campaign ID from Step 2 to select a campaign. You can also supply `validationContext` at the invocation or item level, for example `"validationContext": {"dimensions": {"market": ["us"]}}`. The response includes the resolved tenant and campaign IDs.


## Step 5 - Poll invocation status

Call the status endpoint every 5–10 seconds until the invocation reaches a terminal state.

```bash
curl --request GET \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

**Response while running** (`200 OK`):

```json
{
  "invocationId": "9b9d00c5-8659-4766-8430-ed0a1c9bd87d",
  "skill": "validate",
  "tenantId": "123e4567-e89b-12d3-a456-426614174003",
  "campaignId": "123e4567-e89b-12d3-a456-426614174004",
  "status": "running",
  "itemCount": 2,
  "successCount": 1,
  "failureCount": 0
}
```

Keep polling until `status` is one of: `completed`, `failed`, or `cancelled`. Under `completed`, check `successCount` / `failureCount` - some items may still have failed even though the invocation as a whole finished.


## Step 6 - Fetch per-asset results

Once the invocation completes, list its items for a summary view.

```bash
curl --request GET \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items" \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

**Response** (`200 OK`):

```json
{
  "itemsCount": 2,
  "itemsWithViolations": 1,
  "itemsWithoutViolations": 1,
  "itemsFailed": 0,
  "items": [
    {
      "itemId": "a1b2c3d4-0000-0000-0000-000000000001",
      "status": "completed",
      "mediaType": "image/png",
      "itemName": "banner-01",
      "thumbnailUrl": "https://example.com/signed/banner-01-thumb.png",
      "violationCount": 0
    },
    {
      "itemId": "a1b2c3d4-0000-0000-0000-000000000002",
      "status": "completed",
      "mediaType": "image/png",
      "itemName": "banner-02",
      "thumbnailUrl": "https://example.com/signed/banner-02-thumb.png",
      "violationCount": 1
    }
  ]
}
```

Use the returned `itemId` values for subsequent item requests; these are distinct from the temporary `uploadId` references used at submission. For completed items, `violationCount: 0` means no violations were reported. Check item status as well: failed, cancelled, or rejected items are not successful validations. For items with violations, fetch the full detail to see what was flagged:

```bash
curl --request GET \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/a1b2c3d4-0000-0000-0000-000000000002" \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

**Response** (`200 OK`):

```json
{
  "itemId": "a1b2c3d4-0000-0000-0000-000000000002",
  "status": "completed",
  "mediaType": "image/png",
  "itemName": "banner-02",
  "thumbnailUrl": "https://example.com/signed/banner-02-thumb.png",
  "renditionUrl": "https://example.com/signed/banner-02.png",
  "result": {
    "summaryText": "The asset violates 1 brand guideline.",
    "violations": [
      {
        "violationId": "123e4567-e89b-12d3-a456-426614174002",
        "violationSource": "system",
        "violationSummary": "Color #FF0000 is not in the approved palette.",
        "assetAttributions": [
          {
            "attributionId": "aa-0001",
            "documentId": "banner-02.png",
            "metadata": {},
            "regions": [
              {
                "regionType": "boundingBox",
                "boundingBox": { "x": 0.1, "y": 0.05, "width": 0.3, "height": 0.15 }
              }
            ]
          }
        ],
        "corpusAttributions": [
          {
            "attributionId": "ca-0001",
            "documentId": "brand-guidelines-color.pdf",
            "documentUrl": "https://example.com/signed/brand-guidelines-color.pdf",
            "chunkUrl": "https://example.com/signed/brand-guidelines-color-p4.pdf",
            "metadata": { "page": 4 },
            "regions": []
          }
        ],
        "recordVersion": 1
      }
    ],
    "passes": [],
    "rejections": []
  }
}
```

The response includes additional fields per item and per violation - see the [API Reference](../api/rest/index.md) for the full schema.


## What's next

- Explore all available endpoints in the [API Reference](../api/rest/index.md).
- Learn how to [attach reviewer feedback](../review-feedback/index.md) to flagged assets.
- Try the API interactively with [Postman](../using-postman/index.md).
