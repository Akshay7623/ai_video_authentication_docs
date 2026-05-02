# User API — Reference Documentation

**Base URL:** `https://{UserHttpApi}.execute-api.ap-south-1.amazonaws.com`  
**Content-Type:** `application/json` (all requests and responses)  
**Auth header (protected routes):** `Authorization: Bearer <access_token>`

---

## Table of Contents

1. [User Profile](#1-user-profile)
   - [Get Profile](#11-get-profile)
2. [Templates](#2-templates)
   - [List Templates](#21-list-templates)
   - [Get Sub-Templates (Examples)](#22-get-sub-templates-examples)
3. [Generation](#3-generation)
   - [Initiate Generation](#31-initiate-generation)
   - [Start Generation](#32-start-generation)
4. [History](#4-history)
   - [Get Generation History](#41-get-generation-history)
   - [Get Sub-Template History](#42-get-sub-template-history)
5. [Plans](#5-plans)
   - [Get Plans](#51-get-plans)
6. [Notification Service](#6-notification-service)
   - [FCM Push Notification Payloads](#fcm-push-notification-payloads)
   - [Notification Flow Diagrams](#notification-flow-diagrams)

---

## 1. User Profile

### 1.1 Get Profile

Returns the authenticated user's profile. Sensitive fields (OTP hash, refresh token, failed login attempts, etc.) are always stripped from the response.

**`GET /api/user/profile`** — 🔒 Protected

#### Request

No body. Auth header required.

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "profile": {
    "email": "user@example.com",
    "credits": 250,
    "status": "active",
    "createdAt": 1714247549000,
    "name": "Jane Doe",
    "profilePicture": "https://..."
  }
}
```

> **Note:** The `profile` object contains all non-sensitive user fields. The exact set of fields depends on what the user has set. Sensitive fields that are **always excluded**: `passwordHash`, `otp`, `otpExpiresAt`, `otpLastSentAt`, `otpAttempts`, `resetOtp`, `resetOtpHash`, `resetOtpExpiresAt`, `resetOtpAttempts`, `refreshToken`, `failedLoginAttempts`, `accountLockedUntil`.

**`401 Unauthorized` — Missing or invalid token**
```json
{
  "success": false,
  "message": "Unauthorized missing identity context."
}
```

**`403 Forbidden` — Banned account**
```json
{
  "success": false,
  "message": "Your account has been banned. Please contact support."
}
```

**`404 Not Found` — User record missing**
```json
{
  "success": false,
  "message": "User not found."
}
```

**`500 Internal Server Error`**
```json
{
  "success": false,
  "message": "Internal server error"
}
```

---

## 2. Templates

### 2.1 List Templates

Returns all active (deployed) templates along with the admin-defined display order.

**`GET /api/user/templates`** — 🔒 Protected

#### Query Parameters

None.

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "templates": [
    {
      "category": "General",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
      "description": "Powerful automated workflow ready for deployment.",
      "thumbnail": "",
      "name": "Bike Ride",
      "schema": {
        "summary": {
          "hasWan22": true,
          "userImageInputCount": 2,
          "outputVideoCount": 1,
          "outputImageCount": 0,
          "fixedTextCount": 2,
          "fixedImageCount": 2,
          "fixedVideoCount": 0,
          "hasGemini": true,
          "outputType": "video",
          "userTextInputCount": 2,
          "userVideoInputCount": 0
        }
      }
    },
    {
      "category": "General",
      "workflowId": "8f177549-b9aa-43e4-ac43-74612701fb05",
      "description": "Powerful automated workflow ready for deployment.",
      "thumbnail": "",
      "name": "Man in the city",
      "schema": {
        "summary": {
          "hasWan22": true,
          "userImageInputCount": 1,
          "outputVideoCount": 1,
          "outputImageCount": 0,
          "fixedTextCount": 2,
          "fixedImageCount": 1,
          "fixedVideoCount": 0,
          "outputType": "video",
          "hasGemini": true,
          "userTextInputCount": 2,
          "userVideoInputCount": 0
        }
      }
    }
  ],
  "templateOrder": [
    "8d0c436e-7225-4796-8ae1-a58c26b3e28d",
    "5b952c74-1f0c-4096-a525-66c04dace950",
    "8f177549-b9aa-43e4-ac43-74612701fb05",
    "63f952c8-917a-48d6-9904-7e7501e3943c"
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `templates` | array | Active templates. Only summary fields are returned (not full node graphs). |
| `templates[].workflowId` | string | Unique template ID. Use this in `/initiate` and `/sub-templates/{id}`. |
| `templates[].name` | string | Display name of the template. |
| `templates[].category` | string | Category label (e.g. `"General"`). May be empty string. |
| `templates[].description` | string | Short description. May be empty string. |
| `templates[].thumbnail` | string | URL to thumbnail image. May be empty string. |
| `templates[].schema.summary.outputType` | string | `"video"`, `"image"`, or `"none"` — the type of output this template produces. |
| `templates[].schema.summary.hasWan22` | boolean | Whether this template produces a video via WAN 2.2 (async). |
| `templates[].schema.summary.hasGemini` | boolean | Whether this template uses Gemini for image generation. |
| `templates[].schema.summary.userImageInputCount` | number | Number of image inputs the user must supply. |
| `templates[].schema.summary.userTextInputCount` | number | Number of text inputs the user must supply. |
| `templateOrder` | array | Ordered list of `workflowId` strings defining display order. Use this to sort `templates` on the client. |

**`401 Unauthorized`**
```json
{ "success": false, "error": "Unauthorized" }
```

**`500 Internal Server Error`**
```json
{ "success": false, "error": "Internal server error" }
```

---

### 2.2 Get Sub-Templates (Examples)

Returns example outputs for a given template. These are admin-curated sample generations with `isExample: true`. Used to show "before/after" previews in the app.

**`GET /api/user/sub-templates/{id}`** — 🔒 Protected

#### Path Parameters

| Parameter | Type | Description |
|---|---|---|
| `id` | string | `workflowId` of the template |

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "subTemplates": [
    {
      "id": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
      "title": "This text is for video",
      "outputs": {
        "wan22-1776880930661": "https://...s3.amazonaws.com/workflows/5b952c74-.../62a34ac6-....mp4",
        "geminiGen": "https://...s3.amazonaws.com/outputs/5b952c74-.../62a34ac6-.../gen_geminiGen-....png",
        "wan22": []
      }
    },
    {
      "id": "e017f52b-2cae-48d4-b143-68874d7f6899",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
      "title": "Generate image of these two person travelling in bike...",
      "outputs": {
        "wan22-1776880930661": "https://...s3.amazonaws.com/workflows/5b952c74-.../e017f52b-....mp4",
        "geminiGen": "https://...s3.amazonaws.com/outputs/5b952c74-.../e017f52b-.../gen_geminiGen-....png",
        "wan22": []
      },
      "userInputsSchema": [
        {
          "nodeId": "inputText-1776956660757",
          "inputType": "text",
          "feedsInto": ["wan22"]
        },
        {
          "nodeId": "inputImage-1776862028188",
          "inputType": "image",
          "feedsInto": ["geminiGen"]
        },
        {
          "nodeId": "inputImage-1776956621606",
          "inputType": "image",
          "feedsInto": ["geminiGen"]
        },
        {
          "nodeId": "inputText-1776862010470",
          "inputType": "text",
          "feedsInto": ["geminiGen"]
        }
      ]
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `subTemplates[].id` | string | Sub-template ID. Pass this as `subTemplateId` in `/initiate`. |
| `subTemplates[].title` | string | Human-readable label for this example. |
| `subTemplates[].outputs` | object | Keyed by node ID. Values are S3/CloudFront URLs to the generated assets. `wan22` key holds the video URL; `geminiGen` holds the intermediate image. |
| `subTemplates[].userInputsSchema` | array | Present when the sub-template has user-facing inputs. Each entry describes one input node the user must supply when calling `/initiate` and `/start`. |
| `userInputsSchema[].nodeId` | string | Node ID to use as the key in `userInputs` on `/start`, and to match `mediaFiles[].nodeId` on `/initiate`. |
| `userInputsSchema[].inputType` | string | `"text"` or `"image"`. Determines whether to pass a string or an S3 key. |
| `userInputsSchema[].feedsInto` | array | Which AI nodes this input drives (informational). |

**`500 Internal Server Error`**
```json
{ "success": false, "error": "error message" }
```

---

## 3. Generation

Generation is a **two-step process**:

1. **Initiate** — debit credits, get presigned upload URLs.
2. **Upload** — client uploads files directly to S3 using the presigned URLs.
3. **Start** — trigger AI execution.

```
Client                    API                    S3
  │                        │                      │
  │── POST /initiate ─────►│                      │
  │                        │── debit credits       │
  │                        │── create DB record    │
  │◄── { generationId,     │                      │
  │      uploadUrls } ─────│                      │
  │                        │                      │
  │── PUT (presigned) ─────────────────────────►  │
  │◄── 200 ────────────────────────────────────── │
  │                        │                      │
  │── POST /start ─────────►│                      │
  │                        │── execute workflow    │
  │◄── { finalStatus } ────│                      │
```

---

### 3.1 Initiate Generation

Debits credits, creates a generation record, and returns presigned S3 upload URLs for any required media inputs.

**`POST /api/user/generate/initiate`** — 🔒 Protected

#### Request Body

```json
{
  "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
  "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
  "platform": "android",
  "fcmToken": "dAFe8N5GfAyVOKW1L_4jh9:APA91bErYXPEIFR44U443cRi122PmxHx..."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `workflowId` | string | Yes | ID of the deployed workflow/template |
| `subTemplateId` | string | Yes | ID of the sub-template (example) to base this run on |
| `platform` | string | Yes | `"android"` or `"ios"` — used for FCM push routing |
| `fcmToken` | string | No | Device FCM token. Required to receive push notifications |

> **Note:** `mediaFiles` is not required. The API infers which nodes need uploads from the sub-template's `userInputsSchema` and auto-generates presigned URLs for all image/video input nodes.

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "generationId": "0579ccff-eab6-46be-b767-681f71bbe04d",
  "uploadUrls": [
    {
      "nodeId": "inputImage-1776956621606",
      "uploadUrl": "https://my-bucket.s3.ap-south-1.amazonaws.com/user_assets/...?X-Amz-Signature=...",
      "key": "user_assets/0579ccff-eab6-46be-b767-681f71bbe04d/inputImage-1776956621606_1777723489645_upload_1777723489645.jpg",
      "inputType": "image"
    },
    {
      "nodeId": "inputImage-1776862028188",
      "uploadUrl": "https://my-bucket.s3.ap-south-1.amazonaws.com/user_assets/...?X-Amz-Signature=...",
      "key": "user_assets/0579ccff-eab6-46be-b767-681f71bbe04d/inputImage-1776862028188_1777723489806_upload_1777723489806.jpg",
      "inputType": "image"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `generationId` | string | UUID to use in the `/start` call |
| `uploadUrls` | array | One entry per image/video input node in the sub-template |
| `uploadUrls[].nodeId` | string | Node ID — use this as the key in `userInputs` when calling `/start` |
| `uploadUrls[].uploadUrl` | string | Presigned S3 PUT URL. Valid for **1 hour**. Upload the binary file here directly |
| `uploadUrls[].key` | string | S3 key of the uploaded object — pass this as the value for the corresponding `nodeId` in `userInputs` on `/start` |
| `uploadUrls[].inputType` | string | `"image"` or `"video"` |

> **Important:** Upload each file to S3 using its `uploadUrl` with a `PUT` request before calling `/start`.

```
PUT {uploadUrl}
Content-Type: image/jpeg

<binary file data>
```

**`400 Bad Request` — Missing required fields**
```json
{
  "success": false,
  "error": "missing workflowId or subTemplateId"
}
```

**`400 Bad Request` — Invalid platform**
```json
{
  "success": false,
  "error": "invalid or missing platform. Must be 'android' or 'ios'"
}
```

**`401 Unauthorized`**
```json
{ "success": false, "error": "unauthorized" }
```

**`403 Forbidden` — Insufficient credits**
```json
{
  "success": false,
  "error": "insufficient credits"
}
```

**`404 Not Found` — Workflow or sub-template not found**
```json
{
  "success": false,
  "error": "workflow not found"
}
```

**`500 Internal Server Error`**
```json
{ "success": false, "error": "error message" }
```

---

### 3.2 Start Generation

Executes the AI workflow. This call may take up to 90 seconds (Lambda timeout). For video generations, the response arrives quickly with `finalStatus: "pending"` and the result is delivered via push notification.

**`POST /api/user/generate/start`** — 🔒 Protected

> **Prerequisite:** Files must already be uploaded to S3 via the presigned URLs returned by `/initiate` before calling this endpoint.

#### Request Body

```json
{
  "generationId": "e5fef083-8bde-45c6-8ff0-cf019c250fa8",
  "userInputs": {
    "inputImage-1776956621606": "user_assets/e5fef083-8bde-45c6-8ff0-cf019c250fa8/inputImage-1776956621606_1777657863703_upload_1777657863703.jpg",
    "inputImage-1776862028188": "user_assets/e5fef083-8bde-45c6-8ff0-cf019c250fa8/inputImage-1776862028188_1777657863864_upload_1777657863864.jpg",
    "inputText-1776956660757": "This text is for wan22 node",
    "inputText-1776862010470": "This text is for geminiGen node"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `generationId` | string | Yes | ID returned from `/initiate` |
| `userInputs` | object | Yes | Map of `nodeId → value`. For image/video nodes, pass the `key` returned by `/initiate`. For text nodes, pass the string value directly. |

> **How to build `userInputs`:**
> - **Image/video nodes** → use the `key` from the matching `uploadUrls` entry returned by `/initiate` (e.g. `"user_assets/generationId/inputImage-xxx_timestamp.jpg"`)
> - **Text nodes** → pass the raw string the user typed
> - Node IDs to use come from `userInputsSchema[].nodeId` in the sub-template response

#### Responses

**`200 OK` — Image generation complete**
```json
{
  "success": true,
  "finalStatus": "complete",
  "publicUrl": "https://d1abc.cloudfront.net/outputs/...?Signature=...&Expires=..."
}
```

**`200 OK` — Video generation pending (async)**
```json
{
  "success": true,
  "finalStatus": "pending"
}
```

> When `finalStatus` is `"pending"`, the video is being processed by RunPod. The result will be delivered as an FCM push notification (see [Section 6](#6-notification-service)). Poll `/api/user/generation-history` to check status if push is unavailable.

**`400 Bad Request` — Missing generationId**
```json
{ "success": false, "error": "missing generationId" }
```

**`400 Bad Request` — Generation not in initiatable state**
```json
{ "success": false, "error": "cannot start in processing state" }
```

**`401 Unauthorized`**
```json
{ "success": false, "error": "unauthorized" }
```

**`404 Not Found` — Generation record not found**
```json
{ "success": false, "error": "generation record not found or access denied" }
```

**`404 Not Found` — Linked template or workflow missing**
```json
{ "success": false, "error": "linked template or workflow not found" }
```

**`500 Internal Server Error` — AI execution failed**
```json
{ "success": false, "error": "error message" }
```

> On a `500`, credits are automatically refunded to the user and a `generation_failed` push notification is sent.

---

## 4. History

### 4.1 Get Generation History

Returns the authenticated user's generation records from the last **14 days**, sorted by DynamoDB's default key order.

**`GET /api/user/generation-history`** — 🔒 Protected

#### Query Parameters

None.

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "history": [
    {
      "id": "user@example.com",
      "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
      "subTemplateId": "abc123",
      "status": "complete",
      "creditUsed": 100,
      "publicUrl": "https://d1abc.cloudfront.net/outputs/...?Signature=...",
      "createdAt": 1714247549000
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `status` | string | `initiated` · `processing` · `pending` · `complete` · `failed` |
| `publicUrl` | string | CloudFront signed URL to the output. Present when `status` is `complete` and output is an image |
| `creditUsed` | number | Credits debited for this generation |
| `createdAt` | number | Unix timestamp in milliseconds |

**`401 Unauthorized`**
```json
{ "success": false, "error": "Unauthorized" }
```

**`500 Internal Server Error`**
```json
{ "success": false, "error": "error message" }
```

---

### 4.2 Get Sub-Template History

Returns the authenticated user's sub-template generation history, queried by `userId` GSI on the SubTemplates table.

**`GET /api/user/history`** — 🔒 Protected

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "history": [
    {
      "id": "sub-template-uuid",
      "workflowId": "5b952c74-...",
      "userId": "user@example.com",
      "title": "My animated portrait",
      "outputs": {
        "videoUrl": "https://..."
      },
      "createdAt": 1714247549000
    }
  ]
}
```

**`401 Unauthorized`**
```json
{ "success": false, "error": "Unauthorized" }
```

**`500 Internal Server Error`**
```json
{ "success": false, "error": "error message" }
```

---

## 5. Plans

### 5.1 Get Plans

Returns active subscription plans and in-app purchases for the specified platform.

**`GET /api/user/plans`** — 🔒 Protected

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `platform` | string | No | `android` | `"android"` or `"ios"` |

#### Responses

**`200 OK`**
```json
{
  "success": true,
  "platform": "android",
  "plans": [
    {
      "id": "plan-basic-uuid",
      "name": "Basic",
      "category": "plan",
      "status": "Active",
      "features": ["50 credits/month", "Standard quality"],
      "highlighted": false,
      "billing": "monthly",
      "monthlyPrice": "4.99",
      "yearlyPrice": "49.99",
      "playStoreId": "com.example.app.basic_monthly",
      "appStoreId": null
    }
  ],
  "iaps": [
    {
      "id": "iap-100credits-uuid",
      "name": "100 Credits",
      "category": "iap",
      "status": "Active",
      "price": "1.99",
      "playStoreId": "com.example.app.credits_100",
      "appStoreId": null
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `plans` | array | Subscription plans, sorted by `monthlyPrice` ascending |
| `iaps` | array | One-time in-app purchases, sorted by `price` ascending |
| `highlighted` | boolean | Whether to visually highlight this plan as recommended |
| `billing` | string | `"monthly"` or `"yearly"` |
| `playStoreId` | string | Product ID for Google Play billing |
| `appStoreId` | string | Product ID for Apple App Store billing |

**`500 Internal Server Error`**
```json
{
  "success": false,
  "error": "Internal server error while fetching plans"
}
```

---

## 6. Notification Service

The backend uses **Firebase Cloud Messaging (FCM)** to deliver real-time results to mobile devices. This is critical for video generation, which is asynchronous and can take minutes.

### FCM Push Notification Payloads

All FCM payloads follow this envelope:

```json
{
  "token": "<device_fcm_token>",
  "platform": "android" | "ios",
  "data": { ... }
}
```

The `data` object varies by event type:

---

#### `generation_complete` — Image or Video Ready

Sent when a generation finishes successfully.

```json
{
  "type": "generation_complete",
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "finalStatus": "complete",
  "publicUrl": "https://d1abc.cloudfront.net/outputs/...?Signature=...",
  "videoUrl": "https://s3.amazonaws.com/...",
  "status": "complete"
}
```

| Field | Description |
|---|---|
| `publicUrl` | CloudFront signed URL (14-day expiry). Use this for display. |
| `videoUrl` | Raw S3 URL. Present on video completions from callback path. |
| `generationId` | Match against generation history records. |

---

#### `video_generation_started` — Async Video Queued

Sent via the `/start` HTTP fallback when the Lambda took >30 seconds. Tells the app the video is in queue.

```json
{
  "type": "video_generation_started",
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "finalStatus": "pending"
}
```

---

#### `generation_failed` — Generation Failed, Credits Refunded

Sent when the AI job fails at any stage. Credits are refunded before this notification fires.

```json
{
  "type": "generation_failed",
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "reason": "runpod_job_failed: CUDA out of memory"
}
```

| Field | Description |
|---|---|
| `reason` | Human-readable failure reason. For display or logging only. |

---

### Notification Flow Diagrams

#### Image Generation Flow

```
User App                 API Lambda              Gemini AI
   │                         │                       │
   │── POST /initiate ───────►│                       │
   │◄── { generationId,       │                       │
   │      uploadUrls } ───────│                       │
   │                          │                       │
   │── PUT file to S3 ──────────────────────────────► S3
   │◄── 200 ──────────────────────────────────────── S3
   │                          │                       │
   │── POST /start ───────────►│                       │
   │                          │── ExecuteWorkflow()    │
   │                          │── geminiGen node ─────►│
   │                          │◄── generated image ───│
   │                          │── save to S3           │
   │                          │── sign CloudFront URL  │
   │                          │── update DB: complete  │
   │◄── { finalStatus:        │                       │
   │      "complete",         │                       │
   │      publicUrl } ────────│                       │
```

---

#### Video Generation Flow (Async)

```
User App            API Lambda           RunPod (WAN 2.2)      Callback Lambda
   │                    │                       │                      │
   │── POST /initiate ──►│                       │                      │
   │◄── { generationId } │                       │                      │
   │── upload to S3 ─────────────────────────────────────────────────► S3
   │── POST /start ──────►│                       │                      │
   │                      │── ExecuteWorkflow()   │                      │
   │                      │── wan22 node          │                      │
   │                      │── endpoint.run() ─────►│                      │
   │                      │◄── { jobId } ─────────│                      │
   │                      │── update DB: pending   │                      │
   │◄── { finalStatus:    │                       │                      │
   │     "pending" } ─────│                       │                      │
   │                      │                       │                      │
   │    (waiting...)       │         (processing…) │                      │
   │                      │                       │                      │
   │                      │                       │── job complete        │
   │                      │                       │── invoke callback ───►│
   │                      │                       │                      │── sign CloudFront URL
   │                      │                       │                      │── update DB: complete
   │                      │                       │                      │── invoke UserNotify
   │                      │                       │                      │
   │◄── FCM push ─────────────────────────────────────────────────────── UserNotify
   │    { type: "generation_complete",
   │      publicUrl: "https://..." }
```

---

#### Generation Failure & Credit Refund Flow

```
User App            API Lambda          UserGenerationFailed        UsersTable
   │                    │                       │                       │
   │── POST /start ──────►│                       │                       │
   │                      │── ExecuteWorkflow()   │                       │
   │                      │   ← throws error      │                       │
   │                      │                       │                       │
   │                      │── handleJobFailed() ──────────────────────────►│
   │                      │   (or invokes         │     ADD credits :N     │
   │                      │    FailureFunc)        │◄─────────────────────│
   │                      │                       │                       │
   │                      │── SET status=failed    │                       │
   │                      │── SET failReason       │                       │
   │                      │                       │                       │
   │                      │── invoke UserNotify    │                       │
   │◄── FCM push ──────────────────────────────────────────────────────── UserNotify
   │    { type: "generation_failed",
   │      generationId: "...",
   │      reason: "..." }
   │
   │◄── HTTP 500 ─────────│
   │    { error: "..." }
```

> **Credit Safety:** Credits are refunded atomically using DynamoDB's `ADD credits :N` operation before any notification fires. If the refund itself fails, the error is logged but the failure status is still written to the database.

---

#### FCM Token Lifecycle

```
App Launch / Login
       │
       ▼
Get FCM token from Firebase SDK
       │
       ▼
Pass token + platform in /initiate request body
  { "fcmToken": "...", "platform": "android" }
       │
       ▼
Token stored in UserGenerations record
       │
       ├── Passed to Wan22CallbackFunction on video completion
       ├── Passed to UserGenerationFailedFunction on failure
       └── Used in 30s fallback notification on /start timeout
```

---

## Status Code Summary

| Code | Meaning | When You'll See It |
|---|---|---|
| `200` | Success | Normal successful response |
| `400` | Bad Request | Missing required fields, invalid enum value, wrong state |
| `401` | Unauthorized | Missing/expired/invalid access token or OTP |
| `403` | Forbidden | Banned account, insufficient credits |
| `404` | Not Found | Workflow, template, user, or generation record not found |
| `429` | Too Many Requests | OTP send/resend rate limit exceeded |
| `500` | Internal Server Error | Unhandled exceptions — credits are refunded for generation failures |

---

## Generation Status Reference

| Status | Description | Next State |
|---|---|---|
| `initiated` | Credits debited, upload URLs issued. Waiting for `/start`. | `processing` |
| `processing` | `/start` received, AI workflow executing. | `pending` or `complete` or `failed` |
| `pending` | Video submitted to RunPod. Awaiting async callback. | `complete` or `failed` |
| `complete` | Output ready. `publicUrl` is populated. | — |
| `failed` | Execution failed. Credits refunded. `failReason` populated. | — |
