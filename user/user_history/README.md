# GET /api/user/generation-history

Returns the authenticated user's AI generation history for the **last 14 days**. Each record represents one generation job, including its current status and output URL (if available).

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

> The user's `email` is extracted from the verified JWT claims. A missing or invalid token returns **`401 Unauthorized`**.

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

### Query Parameters

_None_

### Request Body

_None_ — this is a `GET` request.

---

## Behavior

1. Reads the user's `email` from the verified JWT claims.
2. Queries the `UserGenerationsTable` using `id = email` (partition key).
3. Filters results to only include records where `createdAt >= now - 14 days`.
4. Returns the following projected fields per record: `id`, `generationId`, `createdAt`, `creditUsed`, `publicUrl`, `status`, `subTemplateId`, `workflowId`.
5. `publicUrl` is only present on records with `status = "complete"`.

---

## Responses

### ✅ 200 OK — Success

```json
{
  "success": true,
  "history": [
    {
      "createdAt": 1777534739389,
      "creditUsed": 100,
      "generationId": "050b201e-bb13-4e14-9460-77eafd677c4e",
      "id": "merakshay2000@gmail.com",
      "publicUrl": "https://d397ajnx16aos2.cloudfront.net/outputs/5b952c74-1f0c-4096-a525-66c04dace950/e017f52b-2cae-48d4-b143-68874d7f6899/050b201e-bb13-4e14-9460-77eafd677c4e.mp4?Expires=1778744967&Key-Pair-Id=K30CDW4COTXU9V&Signature=TFz8ihUT3qdFclc...",
      "status": "complete",
      "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950"
    },
    {
      "createdAt": 1777723489864,
      "creditUsed": 100,
      "generationId": "0579ccff-eab6-46be-b767-681f71bbe04d",
      "id": "merakshay2000@gmail.com",
      "status": "initiated",
      "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950"
    },
    {
      "createdAt": 1777499549183,
      "creditUsed": 100,
      "generationId": "f5ea8f62-e918-4c72-97e2-313744ffc980",
      "id": "merakshay2000@gmail.com",
      "status": "pending",
      "subTemplateId": "e017f52b-2cae-48d4-b143-68874d7f6899",
      "workflowId": "5b952c74-1f0c-4096-a525-66c04dace950"
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `history` | `array` | List of generation records from the last 14 days, in no guaranteed order. May be `[]`. |

#### History Item Fields

| Field | Type | Description |
|---|---|---|
| `id` | `string` | The user's email — partition key used internally |
| `generationId` | `string` | Unique UUID for this generation job |
| `workflowId` | `string` | UUID of the workflow / template used |
| `subTemplateId` | `string` | UUID of the sub-template (example) used as the generation base |
| `status` | `string` | Current job status — see status values below |
| `createdAt` | `number` | Unix timestamp (milliseconds) when the job was initiated |
| `creditUsed` | `number` | Number of credits deducted for this generation |
| `publicUrl` | `string` | CloudFront **signed URL** for the generated output (image or video). **Only present when `status = "complete"`**. Valid for 14 days. |

#### `status` Values

| Value | Description |
|---|---|
| `"initiated"` | Job was created and credits deducted, but generation has not started yet |
| `"processing"` | AI workflow is currently running |
| `"pending"` | Async video generation (Wan 2.2) was triggered — waiting for result via FCM |
| `"complete"` | Generation finished successfully; `publicUrl` is available |
| `"failed"` | Generation failed; credits were refunded |

---

### ❌ 401 Unauthorized — Missing or Invalid Token

Returned when the JWT is missing, expired, or contains no `email` claim.

```json
{
  "success": false,
  "error": "Unauthorized"
}
```

---

### ❌ 403 Forbidden — Access Denied by Authorizer

```json
{
  "message": "Forbidden"
}
```

---

### ❌ 500 Internal Server Error — Unexpected Error

```json
{
  "success": false,
  "error": "<error message string>"
}
```

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | History returned successfully (may be empty list) |
| `401` | Missing / expired / invalid JWT or no email in claims |
| `403` | Token valid but access denied by the authorizer |
| `500` | Internal DynamoDB error |
