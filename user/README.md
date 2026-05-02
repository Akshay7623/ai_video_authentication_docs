# POST /api/user/generate

Executes the AI generation workflow for a previously initiated job. This is the **second and final step** after `/api/user/generate/initiate`.

The endpoint:
1. Verifies the generation record exists and belongs to the authenticated user.
2. Confirms the record is still in "initiated" state.
3. Merges any uploaded media URLs from the initiate step into the workflow inputs.
4. Executes the full AI workflow (Gemini / Wan 2.2 / RunPod).
5. Returns the result — or falls back to an FCM push notification if processing exceeds 30 seconds.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Authentication

| Type | Details |
|---|---|
| **Scheme** | Bearer token (custom Lambda JWT authorizer – `UserJwtAuth`) |
| **Header** | `Authorization: Bearer <access_token>` |
| **Token source** | Obtained from `/api/auth/email/verify-otp`, `/api/auth/social`, or `/api/auth/refresh-token` |

> The user's `email` is extracted from the verified JWT claims. A missing or invalid token returns **`401 Unauthorized`** before the function is invoked.

---

## Request

### Method & Path

```
POST /api/user/generate
```

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |
| `Content-Type` | ✅ Yes | `application/json` |

### Query Parameters

_None_

### Request Body

```json
{
  "generationId": "a41c5544-771a-460d-ad22-f1507632a645",
  "userInputs": {
    "inputImage-1777726635413": "user_assets/a41c5544-771a-460d-ad22-f1507632a645/inputImage-1777726635413_1777750182122_upload_1777750182122.jpg"
  }
}
```

#### Request Body Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `generationId` | `string` | ✅ Yes | The UUID returned by `/api/user/generate/initiate`. Identifies the generation job to execute. |
| `userInputs` | `object` | ❌ No | Fully dynamic key-value map of `nodeId → value`. Keys and values vary per workflow — see below. |

#### `userInputs` Structure

The `userInputs` object is **completely dynamic** — its keys are workflow node IDs and its values depend on the node type.

| Node ID pattern | Value type | Example value |
|---|---|---|
| `inputImage-*` | `string` — S3 key from the presigned upload | `"user_assets/a41c5544.../inputImage-177...jpg"` |
| `inputVideo-*` | `string` — S3 key from the presigned upload | `"user_assets/a41c5544.../inputVideo-177...mp4"` |
| `inputText-*` | `string` — free text entered by the user | `"A futuristic city at sunset"` |

> Pass `{}` or omit `userInputs` entirely if the workflow has no user inputs.

---

## Behavior

1. Reads `email` from the verified JWT claims.
2. Validates `generationId` is present in the body.
3. Fetches the generation record from `UserGenerationsTable` using `{ id: email, generationId }` — fails with `404` if not found.
4. Checks that `status === "initiated"` — fails with `400` if the job is already in another state.
5. Fetches the linked `SubTemplate` and `DeployedWorkflow` from DynamoDB in parallel.
6. Merges user text inputs with S3 media URLs and admin-fixed assets.
7. Marks the generation as "processing" in DynamoDB.
8. Executes the full AI workflow via `ExecuteWorkflow`.
9. Sets the final status to "complete" or "pending".
10. If execution exceeds 30 seconds, the result is sent via FCM push notification.
11. On any error, the function attempts to refund credits and mark as "failed".

---

## Responses

### ✅ 200 OK — Image Generation Complete (synchronous)

```json
{
  "success": true,
  "finalStatus": "complete",
  "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/.../result.png"
}
```

### ✅ 200 OK — Video Generation Started (asynchronous)

```json
{
  "success": true,
  "finalStatus": "pending"
}
```

### ❌ 400 Bad Request

```json
{ "error": "invalid JSON body" }
```

### ❌ 401 Unauthorized

```json
{ "error": "unauthorized" }
```

### ❌ 404 Not Found

```json
{ "error": "generation record not found or access denied" }
```

### ❌ 500 Internal Server Error

```json
{ "error": "<error message string>" }
```

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | Generation complete or async video started |
| `400` | Invalid body or job not in initiated state |
| `401` | Missing or invalid JWT token |
| `404` | Generation record not found |
| `500` | Workflow execution failed |

---

# GET /api/user/plans

Returns all **Active** subscription plans and in-app purchases (IAPs) available for a specific platform. Plans are sorted by `monthlyPrice` ascending; IAPs are sorted by `price` ascending.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Authentication

| Type | Details |
|---|---|
| **Scheme** | Bearer token (custom Lambda JWT authorizer – `UserJwtAuth`) |
| **Header** | `Authorization: Bearer <access_token>` |
| **Token source** | Obtained from `/api/auth/email/verify-otp`, `/api/auth/social`, or `/api/auth/refresh-token` |

---

## Request

### Method & Path

```
GET /api/user/plans?platform=android
```

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |

### Query Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `platform` | ❌ No | `android` | Target platform — `android` or `ios` |

---

## Behavior

1. Reads the `platform` query parameter (defaults to "android").
2. Scans the `PlansTable` filtering for items where `status = "Active"` AND platform matches.
3. Splits results into plans and IAPs.
4. Returns both groups along with the resolved platform string.

---

## Responses

### ✅ 200 OK — Success

```json
{
  "success": true,
  "platform": "android",
  "plans": [
    {
      "id": "plan_1777564651293",
      "name": "Ad Free",
      "category": "plan",
      "status": "Active",
      "monthlyPrice": 19.99,
      "yearlyPrice": 200,
      "features": ["No Ads", "100 Credit each month"]
    }
  ],
  "iaps": []
}
```

### ❌ 401 Unauthorized

```json
{ "message": "Unauthorized" }
```

### ❌ 500 Internal Server Error

```json
{ "success": false, "error": "Internal server error" }
```

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | Plans and IAPs returned successfully |
| `401` | Missing or invalid JWT token |
| `500` | Internal error |

---

# GET /api/user/sub-templates/{id}

Fetches all **example sub-templates** that belong to a specific deployed workflow template.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Request

### Method & Path

```
GET /api/user/sub-templates/{id}
```

### Path Parameters

| Parameter | Required | Description |
|---|---|---|
| `id` | ✅ Yes | The `workflowId` of the parent template |

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |

---

## Behavior

1. Extracts `workflowId` from the path parameter.
2. Queries the `SubTemplatesTable` filtering for items where `isExample = true`.
3. Returns projected fields: id, workflowId, userInputsSchema, title, outputType, url.

---

## Responses

### ✅ 200 OK — Success

```json
{
  "success": true,
  "subTemplates": [
    {
      "id": "2a9dce00-0554-42b9-a339-842a1676019b",
      "workflowId": "63e79239-8420-4088-97a4-c0e78cb3931f",
      "title": "Restore and enhance this photo",
      "outputType": "image",
      "url": "https://example.s3.amazonaws.com/...",
      "userInputsSchema": [
        {
          "nodeId": "inputImage-1777726635413",
          "inputType": "image",
          "feedsInto": ["geminiGen"]
        }
      ]
    }
  ]
}
```

### ❌ 401 Unauthorized

```json
{ "message": "Unauthorized" }
```

---

# GET /api/user/templates

Fetches all **Active** deployed workflow templates available for end-users.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Request

### Method & Path

```
GET /api/user/templates
```

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |

---

## Behavior

1. Reads the `template_order` configuration from `ConfigurationsTable`.
2. Scans `DeployedWorkflowsTable` for items where `status = "Active"`.
3. Returns projected fields: workflowId, name, description, category, thumbnail, schema.summary.

---

## Responses

### ✅ 200 OK — Success

```json
{
  "success": true,
  "templates": [
    {
      "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
      "name": "Text to image",
      "description": "",
      "category": "",
      "thumbnail": "",
      "schema": {
        "summary": {
          "outputType": "image",
          "outputImageCount": 1,
          "hasGemini": true,
          "userTextInputCount": 1
        }
      }
    }
  ],
  "templateOrder": ["8d0c436e-7225-4796-8ae1-a58c26b3e28d"]
}
```

---

# POST /api/user/generate/initiate

Initiates a new AI generation job for a user.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Request

### Method & Path

```
POST /api/user/generate/initiate
```

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |
| `Content-Type` | ✅ Yes | `application/json` |

### Request Body

```json
{
  "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
  "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
  "platform": "android",
  "fcmToken": "dAFe8N5GfAyVOKW1L_4jh9:APA91bErYXPEIFR..."
}
```

#### Request Body Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `workflowId` | `string` | ✅ Yes | UUID of the deployed workflow |
| `subTemplateId` | `string` | ✅ Yes | UUID of the sub-template |
| `platform` | `string` | ✅ Yes | `"android"` or `"ios"` |
| `fcmToken` | `string` | ❌ No | Firebase Cloud Messaging token |

---

## Behavior

1. Reads user email from JWT claims.
2. Validates workflowId, subTemplateId, and platform.
3. Fetches workflow and sub-template records.
4. Deducts credits from user's balance atomically.
5. Generates pre-signed S3 PUT URLs for media uploads (valid 1 hour).
6. Saves generation record with status "initiated".
7. Returns generationId and upload URLs.

---

## Responses

### ✅ 200 OK — Initiation Successful

```json
{
  "success": true,
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "uploadUrls": [
    {
      "nodeId": "inputImage-1776862028188",
      "uploadUrl": "https://admin-backend-users.s3.ap-south-1.amazonaws.com/...",
      "key": "user_assets/62a34ac6.../inputImage-1776862028188.jpg",
      "inputType": "image"
    }
  ]
}
```

### ❌ 403 Forbidden — Insufficient Credits

```json
{
  "success": false,
  "error": "insufficient credits"
}
```

---

# GET /api/user/generation-history

Returns the authenticated user's AI generation history for the **last 14 days**.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

---

## Request

### Method & Path

```
GET /api/user/generation-history
```

### Headers

| Header | Required | Value |
|---|---|---|
| `Authorization` | ✅ Yes | `Bearer <access_token>` |

---

## Behavior

1. Reads user email from JWT claims.
2. Queries `UserGenerationsTable` with partition key `id = email`.
3. Filters for records where `createdAt >= now - 14 days`.
4. Returns fields: id, generationId, createdAt, creditUsed, publicUrl, status, subTemplateId, workflowId.

---

## Responses

### ✅ 200 OK — Success

```json
{
  "success": true,
  "history": [
    {
      "generationId": "050b201e-bb13-4e14-9460-77eafd677c4e",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
      "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
      "status": "complete",
      "createdAt": 1777534739389,
      "creditUsed": 100,
      "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/.../result.mp4"
    }
  ]
}
```

#### Status Values

| Value | Description |
|---|---|
| "initiated" | Job created and credits deducted |
| "processing" | AI workflow running |
| "pending" | Async video generation in progress |
| "complete" | Generation finished successfully |
| "failed" | Generation failed; credits refunded |