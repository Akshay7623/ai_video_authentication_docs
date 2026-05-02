# User API — Reference Documentation

> **Base URL** `https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com`
>
> All protected routes require a `Authorization: Bearer <access_token>` header. Tokens are issued by the auth endpoints below.

---

## Table of Contents

1. [User Profile](#1-user-profile)
   - [Get Profile](#11-get-profile)
2. [Templates](#2-templates)
   - [List Templates](#21-list-templates)
   - [List Sub-Templates](#22-list-sub-templates)
3. [Generation](#3-generation)
   - [Initiate Generation](#31-initiate-generation)
   - [Start Generation](#32-start-generation)
   - [Generation History](#33-generation-history)
4. [Plans & Purchases](#4-plans--purchases)
   - [Get Plans](#41-get-plans)
5. [Error Reference](#5-error-reference)
6. [Generation Lifecycle](#6-generation-lifecycle)

---

## Common Conventions

### Authentication

Protected routes use a **custom Lambda JWT authorizer**. The authorizer validates the Bearer token and injects the user's `email` into the request context. If the token is missing, expired, or invalid, the request is rejected before reaching the Lambda handler.

```
Authorization: Bearer <access_token>
```

### Response Envelope

All responses follow a consistent shape:

```json
// Success
{ "success": true, ...data }

// Error
{ "error": "human-readable message" }
// or
{ "success": false, "error": "human-readable message" }
```

### CORS

All endpoints allow `*` origins. `OPTIONS` preflight requests are handled automatically by API Gateway.

---

## 1. User Profile

### 1.1 Get Profile

Returns the authenticated user's profile. Sensitive fields (`passwordHash`, OTP fields, refresh token, etc.) are stripped before the response.

```
GET /api/user/profile
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |

**Responses**

```json
// 200
{
  "success": true,
  "profile": {
    "email": "user@example.com",
    "name": "Jane Doe",
    "credits": 250,
    "createdAt": 1714247549000,
    "status": "active"
  }
}

// 401 — Missing or invalid token
{ "message": "Unauthorized missing identity context.", "success": false }

// 403 — Account banned
{ "message": "Your account has been banned. Please contact support.", "success": false }

// 404 — User record not found
{ "message": "User not found.", "success": false }
```

---

## 2. Templates

Templates represent the AI workflows available to users. Sub-templates are pre-filled example runs for a given template.

---

### 2.1 List Templates

Returns all **Active** deployed workflow templates.

```
GET /api/user/templates
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |

**Responses**

```json
// 200
{
  "success": true,
  "templates": [
    {
      "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
      "name": "Photo Restoration",
      "description": "Restore and enhance old or damaged photos",
      "category": "image",
      "thumbnail": "https://cdn.example.com/thumbnails/photo-restore.jpg",
      "schema": {
        "summary": {
          "outputType": "image",
          "outputImageCount": 1,
          "hasGemini": true,
          "userTextInputCount": 0
        }
      }
    }
  ],
  "templateOrder": ["d59d2ad2-fb2d-4f32-9880-9833a712aefc"]
}
```

| Field           | Type       | Description                                                      |
|-----------------|------------|------------------------------------------------------------------|
| `workflowId`    | `string`   | Unique identifier — use this for all subsequent template calls   |
| `name`          | `string`   | Display name                                                     |
| `description`   | `string`   | Short description                                                |
| `category`      | `string`   | Content category (`"image"`, `"video"`, etc.)                    |
| `thumbnail`     | `string`   | URL of the preview image                                         |
| `schema.summary`| `object`   | Quick-look metadata about this workflow (see below)              |
| `templateOrder` | `string[]` | Ordered list of `workflowId`s for display ordering              |

**`schema.summary` fields**

| Field                | Type      | Description                                   |
|----------------------|-----------|-----------------------------------------------|
| `outputType`         | `string`  | `"image"` or `"video"`                        |
| `outputImageCount`   | `number`  | Number of images produced per run             |
| `hasGemini`          | `boolean` | Whether the workflow uses Gemini AI           |
| `userTextInputCount` | `number`  | Number of free-text inputs the user must fill |

---

### 2.2 List Sub-Templates

Returns all **example** sub-templates for a given workflow. Sub-templates represent pre-filled example runs and are used to show users what the workflow produces.

```
GET /api/user/sub-templates/{workflowId}
```

**Path Parameters**

| Parameter    | Required | Description                          |
|--------------|----------|--------------------------------------|
| `workflowId` | ✅        | The `workflowId` from List Templates |

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |

**Responses**

```json
// 200
{
  "success": true,
  "subTemplates": [
    {
      "id": "2a9dce00-0554-42b9-a339-842a1676019b",
      "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
      "title": "Restore this vintage portrait",
      "outputType": "image",
      "url": "https://cdn.example.com/outputs/example-result.jpg",
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

| Field              | Type       | Description                                              |
|--------------------|------------|----------------------------------------------------------|
| `id`               | `string`   | Sub-template ID — pass as `subTemplateId` when initiating |
| `workflowId`       | `string`   | Parent workflow ID                                       |
| `title`            | `string`   | Short description of this example                        |
| `outputType`       | `string`   | `"image"` or `"video"`                                   |
| `url`              | `string`   | URL of the example output (for preview)                  |
| `userInputsSchema` | `object[]` | Describes each input node the user must supply           |

**`userInputsSchema` item**

| Field       | Type       | Description                                                  |
|-------------|------------|--------------------------------------------------------------|
| `nodeId`    | `string`   | Node ID — used as the key in `userInputs` when generating   |
| `inputType` | `string`   | `"image"`, `"video"`, or `"text"`                            |
| `feedsInto` | `string[]` | Which processing nodes consume this input                    |

---

## 3. Generation

Generating content is a **two-step process**:

```
Step 1 → POST /api/user/generate/initiate
         • Deducts credits
         • Returns a generationId and pre-signed S3 upload URLs for media

Step 2 → Upload media directly to S3 (using the pre-signed URLs from Step 1)

Step 3 → POST /api/user/generate/start
         • Executes the AI workflow
         • Returns the result (image) or a pending status (video)
```

See [Generation Lifecycle](#6-generation-lifecycle) for a complete flow diagram.

---

### 3.1 Initiate Generation

Creates a generation job, deducts credits from the user's balance, and returns pre-signed S3 URLs for uploading input media.

```
POST /api/user/generate/initiate
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |
| `Content-Type`  | ✅        | `application/json`       |

**Request Body**

```json
{
  "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
  "subTemplateId": "2a9dce00-0554-42b9-a339-842a1676019b",
  "platform": "android",
  "fcmToken": "dAFe8N5G...",
  "mediaFiles": [
    {
      "nodeId": "inputImage-1777726635413",
      "fileName": "photo.jpg",
      "fileType": "image/jpeg"
    }
  ]
}
```

| Field          | Type       | Required | Description                                                                   |
|----------------|------------|----------|-------------------------------------------------------------------------------|
| `workflowId`   | `string`   | ✅        | From [List Templates](#21-list-templates)                                     |
| `subTemplateId`| `string`   | ✅        | From [List Sub-Templates](#22-list-sub-templates)                             |
| `platform`     | `string`   | ✅        | `"android"` or `"ios"` — used for push notification routing                  |
| `fcmToken`     | `string`   | ❌        | Firebase Cloud Messaging device token — required to receive push notifications |
| `mediaFiles`   | `object[]` | ❌        | Metadata for each media file to upload. If omitted, defaults are used.        |

**`mediaFiles` item**

| Field      | Type     | Required | Description                                  |
|------------|----------|----------|----------------------------------------------|
| `nodeId`   | `string` | ✅        | Must match a node ID from `userInputsSchema` |
| `fileName` | `string` | ❌        | Original filename (used for the S3 key)      |
| `fileType` | `string` | ❌        | MIME type, e.g. `"image/jpeg"`, `"video/mp4"` |

**Responses**

```json
// 200 — Initiated successfully
{
  "success": true,
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "uploadUrls": [
    {
      "nodeId": "inputImage-1777726635413",
      "uploadUrl": "https://user-bucket.s3.ap-south-1.amazonaws.com/user_assets/...?X-Amz-Signature=...",
      "key": "user_assets/62a34ac6.../inputImage-1777726635413_1714247549000_photo.jpg",
      "inputType": "image"
    }
  ]
}
```

```json
// 400 — Validation error
{ "error": "missing workflowId or subTemplateId" }
{ "error": "invalid or missing platform. Must be 'android' or 'ios'" }

// 401 — Unauthorized
{ "error": "unauthorized" }

// 403 — Insufficient credits
{ "success": false, "error": "insufficient credits" }

// 404 — Workflow or sub-template not found
{ "error": "workflow not found" }
{ "error": "sub-template not found" }

// 500 — Internal error
{ "error": "<message>" }
```

**After this call:**
1. Upload each media file to its `uploadUrl` using an HTTP `PUT` request with the matching `Content-Type` header. The pre-signed URL is valid for **1 hour**.
2. Call [Start Generation](#42-start-generation) with the returned `generationId`.

---

### 3.2 Start Generation

Executes the AI workflow for a previously initiated job. This is the final trigger — the workflow runs immediately and synchronously up to the 29-second API Gateway limit. Long-running video jobs switch to async and deliver results via FCM push notification.

```
POST /api/user/generate/start
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |
| `Content-Type`  | ✅        | `application/json`       |

**Request Body**

```json
{
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "userInputs": {
    "inputText-17777889": "A futuristic city at sunset",
    "inputImage-1777726635413": "user_assets/62a34ac6.../inputImage-1777726635413_photo.jpg"
  }
}
```

| Field          | Type     | Required | Description                                                               |
|----------------|----------|----------|---------------------------------------------------------------------------|
| `generationId` | `string` | ✅        | The ID returned by [Initiate Generation](#31-initiate-generation)         |
| `userInputs`   | `object` | ❌        | Key-value map of `nodeId → value`. Omit or pass `{}` if no user inputs.  |

**`userInputs` values by node type**

| Node ID pattern  | Value                                  | Example                                    |
|------------------|----------------------------------------|--------------------------------------------|
| `inputText-*`    | Free text string                       | `"A futuristic city at sunset"`            |
| `inputImage-*`   | S3 object key returned by initiate     | `"user_assets/62a34ac6.../photo.jpg"`      |
| `inputVideo-*`   | S3 object key returned by initiate     | `"user_assets/62a34ac6.../clip.mp4"`       |

**Responses**

```json
// 200 — Image generation complete (synchronous)
{
  "success": true,
  "finalStatus": "complete",
  "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/.../result.png"
}
```

```json
// 200 — Video generation started (asynchronous)
{
  "success": true,
  "finalStatus": "pending"
}
```

```json
// 400 — Missing generationId
{ "error": "missing generationId" }

// 400 — Job already started or in wrong state
{ "error": "cannot start in processing state" }

// 401 — Unauthorized
{ "error": "unauthorized" }

// 404 — Generation record not found
{ "error": "generation record not found or access denied" }

// 500 — Workflow execution failed (credits are refunded automatically)
{ "error": "<message>" }
```

**Async video flow**

When `finalStatus` is `"pending"`, the video is being generated by RunPod in the background. When complete (or failed), the user receives a **FCM push notification** with the following data payload:

```json
// Success notification
{
  "type": "generation_complete",
  "generationId": "62a34ac6-...",
  "videoUrl": "https://bucket.s3.amazonaws.com/...",
  "publicUrl": "https://cdn.cloudfront.net/...",
  "status": "complete"
}

// Failure notification (credits automatically refunded)
{
  "type": "generation_failed",
  "generationId": "62a34ac6-...",
  "reason": "runpod_job_failed: ..."
}
```

---

### 3.3 Generation History

Returns the authenticated user's AI generation history for the **last 14 days**, sorted by creation time.

```
GET /api/user/generation-history
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |

**Responses**

```json
// 200
{
  "success": true,
  "history": [
    {
      "id": "user@example.com",
      "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
      "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
      "subTemplateId": "2a9dce00-0554-42b9-a339-842a1676019b",
      "status": "complete",
      "createdAt": 1777534739389,
      "creditUsed": 100,
      "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/.../result.jpg"
    }
  ]
}
```

**History item fields**

| Field           | Type     | Description                                              |
|-----------------|----------|----------------------------------------------------------|
| `generationId`  | `string` | Unique ID of this generation run                         |
| `workflowId`    | `string` | Which workflow was used                                  |
| `subTemplateId` | `string` | Which sub-template was used                              |
| `status`        | `string` | Current status — see table below                         |
| `createdAt`     | `number` | Unix timestamp in milliseconds                           |
| `creditUsed`    | `number` | Credits deducted for this job                            |
| `publicUrl`     | `string` | Signed CloudFront URL to the output (valid for 14 days). Present when `status` is `"complete"`. |

**Status values**

| Status        | Description                                              |
|---------------|----------------------------------------------------------|
| `initiated`   | Job created, credits deducted, media not yet uploaded    |
| `processing`  | AI workflow is actively running                          |
| `pending`     | Async video generation in progress on RunPod             |
| `complete`    | Generation finished successfully                         |
| `failed`      | Generation failed; credits were refunded automatically   |

---

## 4. Plans & Purchases

### 4.1 Get Plans

Returns all **Active** subscription plans and in-app purchases (IAPs) for a given platform. Plans are sorted by `monthlyPrice` ascending; IAPs by `price` ascending.

```
GET /api/user/plans?platform=android
```

**Headers**

| Header          | Required | Value                    |
|-----------------|----------|--------------------------|
| `Authorization` | ✅        | `Bearer <access_token>`  |

**Query Parameters**

| Parameter  | Required | Default     | Description                  |
|------------|----------|-------------|------------------------------|
| `platform` | ❌        | `"android"` | `"android"` or `"ios"`       |

**Responses**

```json
// 200
{
  "success": true,
  "platform": "android",
  "plans": [
    {
      "id": "plan_basic",
      "name": "Basic",
      "category": "plan",
      "status": "Active",
      "features": ["100 credits/month", "No ads"],
      "highlighted": false,
      "billing": "monthly",
      "monthlyPrice": 4.99,
      "yearlyPrice": 49.99,
      "playStoreId": "com.example.app.basic_monthly",
      "appStoreId": "com.example.app.basic_monthly"
    }
  ],
  "iaps": [
    {
      "id": "iap_credits_100",
      "name": "100 Credits",
      "category": "iap",
      "status": "Active",
      "price": 0.99,
      "playStoreId": "com.example.app.credits_100",
      "appStoreId": "com.example.app.credits_100"
    }
  ]
}
```

**Plan fields**

| Field          | Type      | Description                                        |
|----------------|-----------|----------------------------------------------------|
| `id`           | `string`  | Unique plan ID                                     |
| `name`         | `string`  | Display name                                       |
| `category`     | `string`  | `"plan"` (subscription) or `"iap"` (one-time)     |
| `features`     | `string[]`| List of feature descriptions                       |
| `highlighted`  | `boolean` | Whether to visually highlight this plan (recommended) |
| `billing`      | `string`  | `"monthly"` or `"yearly"`                          |
| `monthlyPrice` | `number`  | Price per month                                    |
| `yearlyPrice`  | `number`  | Price per year                                     |
| `playStoreId`  | `string`  | Google Play product ID                             |
| `appStoreId`   | `string`  | Apple App Store product ID                         |
| `price`        | `number`  | IAP price (present on `iap` items only)            |

---

## 5. Error Reference

| HTTP Status | Common Causes                                              |
|-------------|------------------------------------------------------------|
| `400`       | Missing required fields, invalid JSON, wrong state for operation |
| `401`       | Missing `Authorization` header, expired or invalid token   |
| `403`       | Insufficient credits, banned account                        |
| `404`       | Resource not found or does not belong to the authenticated user |
| `429`       | Too many OTP requests (rate limit)                          |
| `500`       | Unexpected server error — check CloudWatch logs             |

---

## 6. Generation Lifecycle

```
Client                          API                        External Services
  │                              │                                │
  ├─ GET /api/user/templates ───►│                                │
  │◄─ [ templates ] ────────────┤                                │
  │                              │                                │
  ├─ GET /api/user/sub-templates/{id} ─►│                         │
  │◄─ [ subTemplates ] ─────────┤                                │
  │                              │                                │
  ├─ POST /api/user/generate/initiate ─►│                         │
  │    workflowId, subTemplateId │                                │
  │    platform, fcmToken        │──► Deduct credits (DynamoDB)   │
  │                              │──► Generate presigned S3 URLs  │
  │◄─ { generationId, uploadUrls}┤                                │
  │                              │                                │
  ├─ PUT <uploadUrl> (S3 direct) ────────────────────────────────►│ S3
  │◄─ 200 OK ─────────────────────────────────────────────────────┤
  │                              │                                │
  ├─ POST /api/user/generate/start ─►│                            │
  │    generationId, userInputs  │──► Run AI workflow             │
  │                              │    ├─ Gemini (image gen) ─────►│ Gemini
  │                              │    └─ RunPod (video gen) ─────►│ RunPod
  │                              │                                │
  │  [Image path — synchronous]  │                                │
  │◄─ { finalStatus: "complete", │                                │
  │     publicUrl } ─────────────┤                                │
  │                              │                                │
  │  [Video path — async]        │                                │
  │◄─ { finalStatus: "pending" } ┤                                │
  │                              │      RunPod completes ────────►│
  │                              │◄─ wan22_callback invoked ───── │
  │                              │──► Update DB, send FCM ───────►│ FCM
  │◄─ Push notification ─────────────────────────────────────────►│
  │                              │                                │
  ├─ GET /api/user/generation-history ─►│                         │
  │◄─ { history: [...] } ────────┤                                │
```

### Credit Behaviour

- Credits are **deducted atomically** during `initiate`. If the user has insufficient credits, initiation is rejected immediately with a `403`.
- If generation **fails** at any point (workflow error, RunPod failure), credits are **automatically refunded** to the user's balance.
- The amount deducted and refunded is determined by the workflow's configured `credits` value.
