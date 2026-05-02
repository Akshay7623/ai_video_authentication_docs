# GET /api/user/sub-templates/{id}

Fetches all **example sub-templates** that belong to a specific deployed workflow template. Sub-templates are pre-run generation examples marked as `isExample = true` by admins. Mobile clients use this list to show users sample outputs before they start their own generation.

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

> A missing or invalid token returns **`401 Unauthorized`** before the function is invoked.

---

## Request

### Method & Path

```
GET /api/user/sub-templates/{id}
```

### Path Parameters

| Parameter | Required | Description |
|---|---|---|
| `id` | ✅ Yes | The `workflowId` of the parent template (e.g. `63e79239-8420-4088-97a4-c0e78cb3931f`) |

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

1. Extracts `workflowId` from the `{id}` path parameter.
2. Queries the `SubTemplatesTable` on the `workflowId-index` GSI, filtering for items where `isExample = true`.
3. Returns the following projected fields for each sub-template:
   - `id`
   - `workflowId`
   - `userInputsSchema`
   - `title`
   - `outputType`
   - `url`
4. If `url` is not stored at the top level, it falls back to deriving it from the `outputs` field (`wan22` → `saveVideo` → `geminiGen`).
5. If no `url` can be resolved, the field is returned as an empty string `""`.

---

## Responses

### ✅ 200 OK — Success

Returned when the query completes successfully. The list may be empty if no example sub-templates exist for the given workflow.

```json
{
  "success": true,
  "subTemplates": [
    {
      "id": "2a9dce00-0554-42b9-a339-842a1676019b",
      "outputType": "image",
      "userInputsSchema": [
        {
          "nodeId": "inputImage-1777726635413",
          "feedsInto": ["geminiGen"],
          "inputType": "image"
        }
      ],
      "url": "https://admin-backend-admins3bucket-tbyvrgfoedpz.s3.amazonaws.com/outputs/63e79239-8420-4088-97a4-c0e78cb3931f/2a9dce00-0554-42b9-a339-842a1676019b/gen_geminiGen-1777726411729_1777735029246109282.png",
      "title": "Restore and enhance this old blurred photo...",
      "workflowId": "63e79239-8420-4088-97a4-c0e78cb3931f"
    },
    {
      "id": "c67d101f-6182-4fec-b559-9c15b6fe6195",
      "outputType": "image",
      "userInputsSchema": [
        {
          "nodeId": "inputImage-1777726635413",
          "feedsInto": ["geminiGen"],
          "inputType": "image"
        }
      ],
      "title": "Restore and enhance this old blurred photo...",
      "workflowId": "63e79239-8420-4088-97a4-c0e78cb3931f",
      "url": "https://admin-backend-admins3bucket-tbyvrgfoedpz.s3.amazonaws.com/outputs/63e79239-8420-4088-97a4-c0e78cb3931f/c67d101f-6182-4fec-b559-9c15b6fe6195/gen_geminiGen-1777726411729_1777735056967336144.png"
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `subTemplates` | `array` | List of example sub-template objects (may be `[]`) |
| `subTemplates[].id` | `string` | Unique UUID of the sub-template record |
| `subTemplates[].workflowId` | `string` | UUID of the parent workflow / template |
| `subTemplates[].outputType` | `string` | Output kind — `"image"` or `"video"` |
| `subTemplates[].url` | `string` | Direct URL to the generated output file; `""` if not yet available |
| `subTemplates[].title` | `string` | The prompt or description used to produce this example |
| `subTemplates[].userInputsSchema` | `array` | Describes the inputs required from the user — see table below |

#### `userInputsSchema` Item Fields

| Field | Type | Description |
|---|---|---|
| `nodeId` | `string` | Internal node identifier in the workflow graph |
| `inputType` | `string` | Type of user input required — `"image"`, `"video"`, or `"text"` |
| `feedsInto` | `array<string>` | List of downstream node IDs this input connects to |

---

### ❌ 401 Unauthorized — Missing or Invalid Token

Returned by the Lambda Authorizer when the `Authorization` header is absent, the token is expired, or the signature is invalid.

```json
{
  "message": "Unauthorized"
}
```

---

### ❌ 403 Forbidden — Access Denied by Authorizer

Returned when the token is structurally valid but the authorizer explicitly denies access.

```json
{
  "message": "Forbidden"
}
```

---

### ❌ 500 Internal Server Error — Unexpected Lambda Error

Returned when the `{id}` path parameter is missing or an unhandled exception occurs (e.g. DynamoDB error).

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
| `200` | Sub-templates (possibly empty list) returned successfully |
| `401` | Missing / expired / invalid JWT token |
| `403` | Token valid but access denied by the authorizer |
| `500` | Missing path parameter or internal DynamoDB error |
