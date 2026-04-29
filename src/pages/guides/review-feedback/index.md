---
title: Review Feedback - Brand Intelligence
description: Attach and manage reviewer feedback on validation results using the Adobe Brand Intelligence API.
---

# Review Feedback

After a validation job completes, reviewers can attach structured feedback to each item's validation findings. This guide covers listing, creating, and updating comments on a completed bulk validation item.

## Overview

Each validation finding (`BrandValidationError`) on a completed item can receive reviewer feedback in the form of a **comment**. Comments are either:

- `pipeline` — generated automatically by the validation pipeline. These are read-only; you cannot create them via the API.
- `user` — created by your application on behalf of a human reviewer. These can be created and updated.

Comments sit at the **item** level under the `review-and-approve` resource hierarchy:

```
GET    /api/v1/review-and-approve/{jobId}/items/{itemId}/comments
POST   /api/v1/review-and-approve/{jobId}/items/{itemId}/comments
PATCH  /api/v1/review-and-approve/{jobId}/items/{itemId}/comments/{commentId}
```

<InlineAlert variant="info" slots="text"/>

`itemId` is the UUID returned in the `itemId` field of the items response — not `clientItemId`. Comments can only be added to items with status `SUCCEEDED`.


## List comments

Retrieve all validation comments for an item, including both pipeline-generated and user-created entries.

```bash
curl --request GET \
  --url "https://abi.adobe.io/api/v1/review-and-approve/$JOB_ID/items/$ITEM_ID/comments" \
  --header "Authorization: Bearer $ACCESS_TOKEN"
```

Query parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Comments per page. Range 1–100, default 20. |
| `cursor` | string | Opaque cursor returned by the previous page. Omit for the first page. |

**Response** (`200 OK`):

```json
{
  "errors": [
    {
      "errorId": "e1a2b3c4-0000-0000-0000-000000000001",
      "errorSummary": "Logo clear space violation detected.",
      "errorSource": "pipeline",
      "accepted": null,
      "rejected": null
    },
    {
      "errorId": "a9b8c7d6-0000-0000-0000-000000000002",
      "errorSummary": "Logo clear space is insufficient on mobile breakpoint.",
      "errorSource": "user",
      "userName": "Jane Smith",
      "accepted": null,
      "rejected": null
    }
  ],
  "nextCursor": null
}
```

When `nextCursor` is non-null, pass it as the `cursor` parameter to fetch the next page.


## Add a comment

Create a user-authored comment on a completed item.

```bash
curl --request POST \
  --url "https://abi.adobe.io/api/v1/review-and-approve/$JOB_ID/items/$ITEM_ID/comments" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "errorSummary": "Logo clear space is insufficient on mobile breakpoint.",
    "userName": "Jane Smith"
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `errorSummary` | Yes | Description of the finding (1–1024 characters). |
| `userName` | No | Display name of the reviewer creating this comment. |

**Response** (`201 Created`): the created comment object with `errorSource: user`.


## Update a comment

Apply a feedback action to an existing comment using `PATCH`. Three actions are available.

### Accept a comment

Mark a finding as reviewed and accepted:

```bash
curl --request PATCH \
  --url "https://abi.adobe.io/api/v1/review-and-approve/$JOB_ID/items/$ITEM_ID/comments/$COMMENT_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "setAccepted",
    "accepted": true
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `accepted` | Yes | `true` to accept, `false` to undo acceptance. |

### Reject a comment

Dispute a finding with an optional predefined reason:

```bash
curl --request PATCH \
  --url "https://abi.adobe.io/api/v1/review-and-approve/$JOB_ID/items/$ITEM_ID/comments/$COMMENT_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "setRejected",
    "rejected": true,
    "reason": "incorrectAssessment",
    "details": "The logo spacing meets the updated brand guidelines v2.1."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `rejected` | No | `true` to reject, `false` to undo rejection. Defaults to `true`. |
| `reason` | No | One of `incorrectAssessment`, `notApplicable`, `outOfScope`. |
| `details` | No | Free-text elaboration on the rejection reason. |

### Edit a comment

Update the summary text of a user-created comment, and optionally remove specific attribution rows:

```bash
curl --request PATCH \
  --url "https://abi.adobe.io/api/v1/review-and-approve/$JOB_ID/items/$ITEM_ID/comments/$COMMENT_ID" \
  --header "Authorization: Bearer $ACCESS_TOKEN" \
  --header 'Content-Type: application/json' \
  --data '{
    "action": "editContent",
    "errorSummary": "Updated finding description."
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `errorSummary` | No | Replacement summary text (max 1024 characters). |
| `removeAssetAttributionIds` | No | Array of asset attribution UUIDs to remove. |
| `removeCorpusAttributionIds` | No | Array of corpus attribution UUIDs to remove. |


## What's next

- See the full schema for comments in the [API Reference](../../api/index.md).
- Return to [Core Concepts](../core-concepts/index.md) for an overview of the Jobs → Items → Comments hierarchy.
