# GET /api/user/templates

Fetches all **Active** deployed workflow templates available for end-users, along with the admin-configured display order. This is a **read-only, authenticated** endpoint consumed by Android / iOS mobile clients to populate the template selection screen.

---

## Base URL

```
https://<UserHttpApi-ID>.execute-api.<region>.amazonaws.com
```

> The exact invoke URL is printed in the SAM stack `Outputs` after deployment.

---

## Authentication

| Type | Details |
|---|---|
| **Scheme** | Bearer token (custom Lambda JWT authorizer – `UserJwtAuth`) |
| **Header** | `Authorization: Bearer <access_token>` |
| **Token source** | Obtained from `/api/auth/email/verify-otp`, `/api/auth/social`, or `/api/auth/refresh-token` |

> The authorizer validates the JWT against the `ACCESS_SECRET` environment variable. A missing or invalid token returns **`401 Unauthorized`** before this function is ever invoked.

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
| `Content-Type` | ❌ No | Not required for GET |

### Query Parameters

_None_

### Request Body

_None_ — this is a `GET` request.

---

## Behavior

1. Reads the **`template_order`** configuration item (`ConfigKey = "template_order"`) from the `ConfigurationsTable`. If the item does not exist, `templateOrder` defaults to an empty array `[]`.
2. Scans the **`DeployedWorkflowsTable`** for all items where `status = "Active"`.
3. Returns only the following projected fields for each template:
   - `workflowId`
   - `schema.summary`
   - `category`
   - `description`
   - `thumbnail`
   - `name`
4. Returns both the template list and the ordering array in a single response.

---

## Responses

### ✅ 200 OK — Success

Returned when the scan completes successfully. The list may be empty if no templates are currently active.

```json
{
  "success": true,
  "templates": [
    {
      "category": "",
      "workflowId": "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
      "description": "",
      "thumbnail": "",
      "name": "Text to image",
      "schema": {
        "summary": {
          "hasWan22": false,
          "userImageInputCount": 0,
          "outputVideoCount": 0,
          "outputImageCount": 1,
          "fixedTextCount": 0,
          "fixedVideoCount": 0,
          "fixedImageCount": 0,
          "hasGemini": true,
          "outputType": "image",
          "userTextInputCount": 1,
          "userVideoInputCount": 0
        }
      }
    },
    {
      "category": "General",
      "workflowId": "63e79239-8420-4088-97a4-c0e78cb3931f",
      "description": "Powerful automated workflow ready for deployment.",
      "thumbnail": "",
      "name": "Blur to Clean & Sharp",
      "schema": {
        "summary": {
          "hasWan22": false,
          "userImageInputCount": 1,
          "outputVideoCount": 0,
          "outputImageCount": 1,
          "fixedTextCount": 1,
          "fixedVideoCount": 0,
          "outputType": "image",
          "fixedImageCount": 0,
          "hasGemini": true,
          "userTextInputCount": 0,
          "userVideoInputCount": 0
        }
      }
    }
  ],
  "templateOrder": [
    "8d0c436e-7225-4796-8ae1-a58c26b3e28d",
    "b85b782e-b186-437e-9006-ea64a765889e",
    "976707b2-afa8-43d6-a0bd-bf269b1ca19e",
    "4bbf2083-6a23-4022-bce8-f813d6c0a8ca",
    "f5254af5-c756-4bda-a4fa-a8616ccd71e0",
    "63f952c8-917a-48d6-9904-7e7501e3943c",
    "d59d2ad2-fb2d-4f32-9880-9833a712aefc",
    "63e79239-8420-4088-97a4-c0e78cb3931f",
    "5b952c74-1f0c-4096-a525-66c04dace950",
    "8f177549-b9aa-43e4-ac43-74612701fb05"
  ]
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `templates` | `array` | List of active template objects (may be `[]`) |
| `templates[].workflowId` | `string` | Unique UUID of the deployed workflow / template |
| `templates[].name` | `string` | Display name shown in the UI |
| `templates[].description` | `string` | Short description; may be an empty string `""` |
| `templates[].category` | `string` | Category tag (e.g. `"General"`); may be an empty string `""` |
| `templates[].thumbnail` | `string` | URL of the template preview image; may be an empty string `""` |
| `templates[].schema.summary` | `object` | Workflow capability metadata — see table below |
| `templateOrder` | `array<string>` | Admin-defined ordered list of `workflowId` values; `[]` if not configured |

#### `schema.summary` Fields

| Field | Type | Description |
|---|---|---|
| `outputType` | `string` | Primary output kind — `"image"` or `"video"` |
| `outputImageCount` | `number` | Number of images the workflow produces |
| `outputVideoCount` | `number` | Number of videos the workflow produces |
| `userTextInputCount` | `number` | Number of free-text inputs the user must provide |
| `userImageInputCount` | `number` | Number of image inputs the user must upload |
| `userVideoInputCount` | `number` | Number of video inputs the user must upload |
| `fixedTextCount` | `number` | Number of admin-fixed text nodes (not editable by the user) |
| `fixedImageCount` | `number` | Number of admin-fixed image nodes (not editable by the user) |
| `fixedVideoCount` | `number` | Number of admin-fixed video nodes (not editable by the user) |
| `hasGemini` | `boolean` | `true` if the workflow includes a Gemini AI processing step |
| `hasWan22` | `boolean` | `true` if the workflow includes a Wan 2.2 video generation step |

---

### ❌ 401 Unauthorized — Missing or Invalid Token

Returned by the **Lambda Authorizer** before the function executes. This happens when:
- The `Authorization` header is absent.
- The token is malformed, expired, or signed with a wrong secret.

```json
{
  "message": "Unauthorized"
}
```

> **Note:** This response is generated by API Gateway / the authorizer, **not** by the Lambda function itself, so the body format may vary slightly depending on API Gateway's default error response configuration.

---

### ❌ 403 Forbidden — Access Denied by Authorizer

Returned by the Lambda Authorizer when the token is structurally valid but the authorizer explicitly denies access (e.g. user is banned / inactive at token validation time).

```json
{
  "message": "Forbidden"
}
```

---

### ❌ 500 Internal Server Error — Unexpected Lambda Error

Returned when an unhandled exception occurs inside the function — most commonly a DynamoDB connectivity failure or IAM permission error.

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
| `200` | Templates (possibly empty list) returned successfully |
| `401` | Missing / expired / invalid JWT token |
| `403` | Token valid but access denied by the authorizer |
| `500` | Internal DynamoDB error or unhandled Lambda exception |
