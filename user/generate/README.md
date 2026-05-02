# POST /api/user/generate

Executes the AI generation workflow for a previously initiated job. This is the **second and final step** after `/api/user/generate/initiate`.

The endpoint:
1. Verifies the generation record exists and belongs to the authenticated user.
2. Confirms the record is still in `"initiated"` state.
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

The `userInputs` object is **completely dynamic** — its keys are workflow node IDs and its values depend on the node type. Use the `userInputsSchema` from the sub-template (returned by `/api/user/sub-templates/{id}`) to know which `nodeId`s to populate.

| Node ID pattern | Value type | Example value |
|---|---|---|
| `inputImage-*` | `string` — S3 key from the presigned upload (the `key` field returned by `/initiate`) | `"user_assets/a41c5544.../inputImage-177...jpg"` |
| `inputVideo-*` | `string` — S3 key from the presigned upload | `"user_assets/a41c5544.../inputVideo-177...mp4"` |
| `inputText-*` | `string` — free text entered by the user | `"A futuristic city at sunset"` |

> Pass `{}` or omit `userInputs` entirely if the workflow has no user inputs.

---

## Behavior

1. Reads `email` from the verified JWT claims.
2. Validates `generationId` is present in the body.
3. Fetches the generation record from `UserGenerationsTable` using `{ id: email, generationId }` — fails with `404` if not found.
4. Checks that `status === "initiated"` — fails with `400` if the job is already `"processing"`, `"complete"`, `"pending"`, or `"failed"`.
5. Fetches the linked `SubTemplate` and `DeployedWorkflow` from DynamoDB in parallel.
6. Merges user text inputs (`userInputs`) with the S3 media URLs from the initiate step (`uploadUrls`) and the admin-fixed assets from the sub-template.
7. Marks the generation as `"processing"` in DynamoDB.
8. Executes the full AI workflow via `ExecuteWorkflow`.
9. Sets the final status:
   - `"complete"` — all nodes finished synchronously (Gemini image generation).
   - `"pending"` — a Wan 2.2 video node was triggered asynchronously; result will arrive via FCM push notification.
10. **Timeout fallback:** If execution exceeds **30 seconds**, the HTTP response may be dropped by API Gateway's hard timeout. In this case, the result is sent to the app via **FCM push notification** (if `fcmToken` was provided during initiate).
11. On any unhandled error, the function attempts to refund the deducted credits and mark the generation as `"failed"`.

---

## Responses

### ✅ 200 OK — Image Generation Complete (synchronous)

Returned when the workflow finishes synchronously and the output is an image.

```json
{
  "success": true,
  "finalStatus": "complete",
  "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/63e79239-8420-4088-97a4-c0e78cb3931f/2a9dce00-0554-42b9-a339-842a1676019b/gen_geminiGen-1777726411729_1777750509640.png?Expires=1778960110&Key-Pair-Id=K30CDW4COTXU9V&Signature=GTd--qeKhL1Pftk..."
}
```

### ✅ 200 OK — Video Generation Started (asynchronous)

Returned when the workflow triggers a Wan 2.2 video generation job. The result is **not yet available**; the app will receive an FCM notification when the video is ready.

```json
{
  "success": true,
  "finalStatus": "pending"
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `finalStatus` | `string` | `"complete"` — result is ready now; `"pending"` — async video job in progress |
| `publicUrl` | `string` | CloudFront **signed URL** for the generated image. Valid for 14 days. Includes `Expires`, `Key-Pair-Id`, and `Signature` query parameters. **Only present when `finalStatus = "complete"` and `outputType = "image"`** |

---

### FCM Push Notification (Timeout Fallback)

If the generation takes longer than **30 seconds**, the HTTP response may be lost. The result is instead delivered as a Firebase push notification to the `fcmToken` provided during the initiate step.

#### For image output (`finalStatus = "complete"`):
```json
{
  "type": "generation_complete",
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "finalStatus": "complete",
  "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/.../result.png"
}
```

#### For video output (`finalStatus = "pending"`):
```json
{
  "type": "video_generation_started",
  "generationId": "62a34ac6-fb68-481a-bdaf-05b69ea34bc3",
  "finalStatus": "pending"
}
```

> Always implement FCM notification handling as the primary result delivery mechanism for video workflows.

---

### ❌ 400 Bad Request — Validation or State Error

```json
{ "error": "invalid JSON body" }
```
```json
{ "error": "missing generationId" }
```
```json
{ "error": "cannot start in processing state" }
```

> The state error message reflects the actual current status: `processing`, `complete`, `pending`, or `failed`.

---

### ❌ 401 Unauthorized — Missing or Invalid Token

```json
{ "error": "unauthorized" }
```

---

### ❌ 404 Not Found — Record Not Found

Returned when the `generationId` does not exist or does not belong to the authenticated user.

```json
{ "error": "generation record not found or access denied" }
```
```json
{ "error": "linked template or workflow not found" }
```

---

### ❌ 500 Internal Server Error — Execution Failed

Returned when the AI workflow throws an unhandled error. Credits are automatically refunded and the generation is marked as `"failed"`.

```json
{ "error": "<error message string>" }
```

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | Generation complete (`finalStatus: "complete"`) or async video started (`finalStatus: "pending"`) |
| `400` | Invalid body, missing `generationId`, or job is not in `"initiated"` state |
| `401` | Missing / expired / invalid JWT token |
| `404` | Generation record or linked workflow/template not found |
| `500` | AI workflow execution failed (credits refunded automatically) |
