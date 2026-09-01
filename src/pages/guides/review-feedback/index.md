---
title: Review Feedback - Brand Intelligence
description: Attach and manage reviewer feedback on validation results using the Adobe Brand Intelligence API.
---

# Review Feedback

After a validation invocation completes, reviewers can attach structured feedback to each item's validation findings. This guide covers listing, creating, and updating **violations** on a completed item.

## Overview

Each validation finding (`RaItemViolation`) on a completed item is either:

- `system` - generated automatically by the validation pipeline. You cannot create these via the API, but you can review them.
- `user` - created by your application on behalf of a human reviewer. These can be created and edited.

Violations sit at the **item** level under the `skills/ra` resource hierarchy:

```
GET    /api/abi/skills/ra/{si_id}/items/{item_id}          → item detail incl. violations
POST   /api/abi/skills/ra/{si_id}/items/{item_id}/violations
PATCH  /api/abi/skills/ra/{si_id}/items/{item_id}/violations/{violation_id}
```

<InlineAlert variant="info" slots="text"/>

`item_id` is the UUID returned in the `itemId` field of the items response. Violations can only be added to items with status `completed`.


## List violations

Violations aren't listed on their own endpoint - fetch the item detail, which includes all violations (both `system` and `user`) under `result.violations`.

```bash
curl --request GET \
  --url "https://abi-mw.adobe.io/api/abi/skills/ra/$INVOCATION_ID/items/$ITEM_ID" \
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
        "violationId": "v-0001",
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
    ]
  }
}
```

Each violation may also include `assetAttributions` and `corpusAttributions` linking back to the specific regions in the asset and the guideline chunks that informed the finding. `reviewStatus` is `null` until a reviewer accepts or rejects the violation. See the [API Reference](../../api/index.md) for the full `RaItemViolation` schema.


## Add a violation

Create a user-authored violation on a completed item.

```bash
curl --request POST \
  --url "https://abi-mw.adobe.io/api/abi/skills/ra/$INVOCATION_ID/items/$ITEM_ID/violations" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "violationSummary": "Logo clear space is insufficient on mobile breakpoint.",
    "userName": "Jane Smith"
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `violationSummary` | Yes | Description of the finding (1–2048 characters). |
| `userName` | Yes | Display name of the reviewer creating this violation. |

**Response** (`201 Created`): the created violation object with `violationSource: user`.


## Update a violation

Apply a feedback action to an existing violation using `PATCH`. The `action` field selects which of three shapes the rest of the body follows.

### Accept or reject a violation

```bash
curl --request PATCH \
  --url "https://abi-mw.adobe.io/api/abi/skills/ra/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "setReviewStatus",
    "reviewStatus": "rejected",
    "rejectionReason": "incorrectAssessment",
    "rejectionDetails": "The logo spacing meets the updated brand guidelines v2.1."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `reviewStatus` | Yes | `accepted` or `rejected`. |
| `rejectionReason` | No | One of `incorrectAssessment`, `notApplicable`. Only meaningful when rejecting. |
| `rejectionDetails` | No | Free-text elaboration on the rejection (1–2048 characters). |

### Reset a violation

Clear a violation's feedback back to pending (un-accept / un-reject):

```bash
curl --request PATCH \
  --url "https://abi-mw.adobe.io/api/abi/skills/ra/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "resetViolation"
  }'
```

### Edit a violation

Update the summary text of a violation:

```bash
curl --request PATCH \
  --url "https://abi-mw.adobe.io/api/abi/skills/ra/$INVOCATION_ID/items/$ITEM_ID/violations/$VIOLATION_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
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

- See the full schema for violations in the [API Reference](../../api/index.md).
- Return to [Core Concepts](../core-concepts/index.md) for an overview of the Invocation → Item → Violation hierarchy.
