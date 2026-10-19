---
title: Review Feedback - Brand Intelligence
description: Attach and manage reviewer feedback on validation results using the Adobe Brand Intelligence API.
---

# Review Feedback

After a validation invocation completes, reviewers can attach structured feedback to each item's validation findings. This guide covers listing, creating, and updating **violations** on a completed item.

## Overview

Each validation finding (`ValidateItemViolation`) on a completed item is either:

- `system` - generated automatically by the validation pipeline. You cannot create these via the API, but you can review them.
- `user` - created by your application on behalf of a human reviewer. These can be created and edited.

The REST API base URL is `https://brand-intelligence.adobe.com`. Violations sit at the **item** level under the `skills/validate` resource hierarchy:

```
GET    /skills/validate/{invocation_id}/items/{item_id} → item detail incl. violations
POST   /skills/validate/{invocation_id}/items/{item_id}/violations
PATCH  /skills/validate/{invocation_id}/items/{item_id}/violations/{violation_id}
```

<InlineAlert variant="info" slots="text"/>

`item_id` is the UUID returned in the `itemId` field of the items response. Violations can only be added to items with status `completed`.


## List violations

Violations aren't listed on their own endpoint - fetch the item detail, which includes all violations (both `system` and `user`) under `result.violations`.

```bash
curl --request GET \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID" \
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
        "violationSummary": "Logo clear space violation detected.",
        "assetAttributions": [],
        "corpusAttributions": [],
        "reviewStatus": null,
        "rejectionReason": null,
        "rejectionDetails": null,
        "userName": null,
        "recordVersion": 1
      }
    ],
    "passes": [],
    "rejections": []
  }
}
```

Each violation may also include `assetAttributions` and `corpusAttributions` linking back to the specific regions in the asset and the guideline chunks that informed the finding. `reviewStatus` is `null` until a reviewer accepts or rejects the violation. The result also includes `passes` and `rejections` lists with the same finding schema. See the [API Reference](../api/rest/index.md) for the full `ValidateItemViolation` schema.


## Add a violation

Create a user-authored violation on a completed item.

```bash
curl --request POST \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID/violations" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "violationSummary": "Logo clear space is insufficient on mobile breakpoint."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `violationSummary` | Yes | Description of the finding (1–2048 characters). |

Do not include `userName`; it is not part of the create request schema.

**Response** (`201 Created`): the created violation object with `violationSource: user` and a `recordVersion`.


## Update a violation

Apply a feedback action to an existing violation using `PATCH`. The `action` field selects one of four request shapes: `acceptViolation`, `rejectViolation`, `resetViolation`, or `editViolation`.

Every update requires an `If-Match` header containing the violation's current version as a quoted string. Read `recordVersion` from the item detail or the created violation:

```bash
export VIOLATION_ID=123e4567-e89b-12d3-a456-426614174002
export RECORD_VERSION=1
```

All update examples below use that version. Each successful update returns **`204 No Content`**, not a JSON body. Use `--include` to see the response headers, then update `RECORD_VERSION` to the version in `ETag` (for example, `ETag: "2"` means `export RECORD_VERSION=2`) before another update. If another reviewer has changed the finding, fetch the current item detail and reconcile the change before retrying with its version.

### Accept a violation

```bash
curl --include --request PATCH \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header "If-Match: \"$RECORD_VERSION\"" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "acceptViolation"
  }'
```

### Reject a violation

```bash
curl --include --request PATCH \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header "If-Match: \"$RECORD_VERSION\"" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "rejectViolation",
    "rejectionReason": "incorrectAssessment",
    "rejectionDetails": "The logo spacing meets the updated brand guidelines v2.1."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `rejectionReason` | No | One of `incorrectAssessment`, `notApplicable`. Only meaningful when rejecting. |
| `rejectionDetails` | Yes | Free-text elaboration on the rejection (1–2048 characters). |

### Reset a violation

Clear a violation's feedback back to pending (un-accept / un-reject):

```bash
curl --include --request PATCH \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header "If-Match: \"$RECORD_VERSION\"" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "resetViolation"
  }'
```

### Edit a violation

Update the summary text of a violation:

```bash
curl --include --request PATCH \
  --url "https://brand-intelligence.adobe.com/skills/validate/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header "If-Match: \"$RECORD_VERSION\"" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "editViolation",
    "violationSummary": "Updated finding description."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `violationSummary` | Yes | Replacement summary text (1–2048 characters). |


## What's next

- See the full schema for violations in the [API Reference](../api/rest/index.md).
- Return to [Core Concepts](../core-concepts/index.md) for an overview of the Invocation → Item → Violation hierarchy.
