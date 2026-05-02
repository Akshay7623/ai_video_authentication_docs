# POST /api/user/generate/initiate

Initiates a new AI generation job for a user. This endpoint:
1. Validates the user has enough credits and atomically deducts the cost.
2. Creates a generation record in DynamoDB with status `"initiated"`.
3. Returns a `generationId` and pre-signed S3 upload URLs if the workflow requires user media inputs.

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
POST /api/user/generate/initiate
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
  "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950",
  "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
  "platform": "android",
  "fcmToken": "dAFe8N5GfAyVOKW1L_4jh9:APA91bErYXPEIFR44U443cRi122PmxHxIbW2Qq5MQ-A5gRaz32eoRDSxDkw9YhdFsWLwfY7Ed1NmpotIvfVKYuH0n0CKicF6pmzZJiCXEeJevIiG_XmyPck"
}
```

#### Request Body Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `workflowId` | `string` | ✅ Yes | UUID of the deployed workflow / template to run |
| `subTemplateId` | `string` | ✅ Yes | UUID of the sub-template (example) to use as the generation base |
| `platform` | `string` | ✅ Yes | Device platform — must be `"android"` or `"ios"` (case-insensitive) |
| `fcmToken` | `string` | ❌ No | Firebase Cloud Messaging token for push notifications when the job completes; omit or set to `null` if not available |

---

## Behavior

1. Parses and validates the JSON request body.
2. Reads the user's `email` from the verified JWT claims.
3. Validates that `workflowId` and `subTemplateId` are present and `platform` is `"android"` or `"ios"`.
4. Fetches the workflow and sub-template records from DynamoDB in parallel.
5. Deducts `workflow.credits` from the user's credit balance atomically using a `ConditionExpression`. If the user has insufficient credits, returns `403`.
6. Generates a new `generationId` (UUID v4).
7. For every image/video input node in the sub-template's `userInputs`, generates a **pre-signed S3 PUT URL** (valid for **1 hour**) and a permanent S3 object URL.
8. Saves a new record to `UserGenerationsTable` with `status = "initiated"`.
9. Returns the `generationId` and the presigned upload URLs to the client.

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
      "uploadUrl": "https://admin-backend-users.s3.ap-south-1.amazonaws.com/user_assets/62a34ac6.../inputImage-1776862028188_1714247549000_upload.jpg?X-Amz-Signature=...",
      "key": "user_assets/62a34ac6.../inputImage-1776862028188_1714247549000_upload.jpg",
      "inputType": "image"
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `generationId` | `string` | UUID for this generation job — store this; it is needed to poll job status and retrieve results |
| `uploadUrls` | `array` | List of presigned S3 upload slots; empty `[]` if the workflow has no user media inputs |
| `uploadUrls[].nodeId` | `string` | The input node this upload slot corresponds to |
| `uploadUrls[].uploadUrl` | `string` | Pre-signed S3 PUT URL — use this to upload the file directly from the client (valid for **1 hour**) |
| `uploadUrls[].key` | `string` | The S3 object key where the file will be stored |
| `uploadUrls[].inputType` | `string` | `"image"` or `"video"` |

> **Important:** After receiving this response, the client must `PUT` each media file to its corresponding `uploadUrl` before calling the generate endpoint.

---

### ❌ 400 Bad Request — Validation Error

Returned when the request body is malformed or required fields are missing/invalid.

```json
{ "error": "invalid JSON body" }
```
```json
{ "error": "missing workflowId or subTemplateId" }
```
```json
{ "error": "invalid or missing platform. Must be 'android' or 'ios'" }
```

---

### ❌ 401 Unauthorized — Missing or Invalid Token

Returned by the authorizer or when the JWT contains no `email` claim.

```json
{ "error": "unauthorized" }
```

---

### ❌ 403 Forbidden — Insufficient Credits

Returned when the user's credit balance is lower than the cost of the workflow.

```json
{
  "success": false,
  "error": "insufficient credits"
}
```

---

### ❌ 404 Not Found — Workflow or Sub-Template Not Found

Returned when the provided `workflowId` or `subTemplateId` does not exist in the database.

```json
{ "error": "workflow not found" }
```
```json
{ "error": "sub-template not found" }
```

---

### ❌ 500 Internal Server Error — Unexpected Error

Returned on any unhandled exception (e.g. DynamoDB connectivity failure, S3 presign error).

```json
{ "error": "<error message string>" }
```

---

## How to Upload Media Files

After a successful `200` response, upload each file using an HTTP `PUT` directly to S3:

```http
PUT <uploadUrl>
Content-Type: image/jpeg

<binary file data>
```

> Do **not** include the `Authorization` header when uploading to the presigned URL — it is already authenticated via the query parameters in the URL.

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | Generation initiated; `generationId` and upload URLs returned |
| `400` | Invalid JSON, missing required field, or invalid platform value |
| `401` | Missing / expired / invalid JWT token or no email in claims |
| `403` | User has insufficient credits |
| `404` | `workflowId` or `subTemplateId` not found |
| `500` | Internal DynamoDB or S3 error |
