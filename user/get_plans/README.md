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

> A missing or invalid token returns **`401 Unauthorized`** before the function is invoked.

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
| `platform` | ❌ No | `android` | Target platform — `android` or `ios` (case-insensitive). Filters which plans and IAPs are returned based on the platform they are available on. |

### Request Body

_None_ — this is a `GET` request.

---

## Behavior

1. Reads the `platform` query parameter (defaults to `"android"` if omitted).
2. Scans the `PlansTable` filtering for items where `status = "Active"` AND `platform` list `contains` the requested platform value.
3. Splits results into two groups:
   - **`plans`** — items with `category = "plan"`, sorted by `monthlyPrice` ascending.
   - **`iaps`** — items with `category = "iap"`, sorted by `price` ascending.
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
      "yearlyPrice": 200,
      "highlighted": false,
      "playStoreId": "com.playstore.plan_19",
      "status": "Active",
      "category": "plan",
      "appStoreId": "com.appstore.plan_19",
      "id": "plan_1777564651293",
      "name": "Ad Free",
      "monthlyPrice": 19.99,
      "billing": "monthly",
      "features": [
        "No Ads",
        "100 Credit each month",
        "Premium models"
      ]
    },
    {
      "yearlyPrice": 400,
      "highlighted": false,
      "playStoreId": "com.playstore.plan_40",
      "status": "Active",
      "category": "plan",
      "appStoreId": "com.appstore.plan_40",
      "id": "plan_1777564996050",
      "name": "Starter",
      "monthlyPrice": 40,
      "billing": "monthly",
      "features": [
        "No Ads",
        "Daily 20 Images",
        "Daily 10 Videos"
      ]
    },
    {
      "yearlyPrice": 1000,
      "highlighted": false,
      "playStoreId": "com.playstore.pro_100",
      "status": "Active",
      "category": "plan",
      "appStoreId": "",
      "id": "plan_1777751259737",
      "name": "Pro Android",
      "monthlyPrice": 100,
      "billing": "monthly",
      "features": [
        "Ad Free",
        "20 Images Daily",
        "5 Videos Daily",
        "250 Credit per Month"
      ]
    }
  ],
  "iaps": [
    {
      "playStoreId": "com.playstore.premium_50",
      "status": "Active",
      "category": "iap",
      "appStoreId": "",
      "id": "iap_1777751097290",
      "price": 50,
      "name": "Premium"
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|---|---|---|
| `success` | `boolean` | Always `true` on this status code |
| `platform` | `string` | The resolved platform value used for filtering (`"android"` or `"ios"`) |
| `plans` | `array` | Active subscription plans for the platform, sorted by `monthlyPrice` ascending. May be `[]`. |
| `iaps` | `array` | Active in-app purchases for the platform, sorted by `price` ascending. May be `[]`. |

#### Plan Object Fields

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique plan identifier |
| `name` | `string` | Display name of the plan |
| `category` | `string` | Always `"plan"` for subscription plans |
| `status` | `string` | Always `"Active"` in this response |
| `billing` | `string` | Billing cycle — e.g. `"monthly"` |
| `monthlyPrice` | `number` | Monthly price in USD |
| `yearlyPrice` | `number` | Yearly price in USD |
| `highlighted` | `boolean` | `true` if the plan should be visually highlighted as recommended |
| `features` | `array<string>` | List of feature strings to display on the plan card |
| `playStoreId` | `string` | Google Play product ID for purchase; `""` if not available on Android |
| `appStoreId` | `string` | Apple App Store product ID for purchase; `""` if not available on iOS |

#### IAP Object Fields

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique IAP identifier |
| `name` | `string` | Display name of the in-app purchase |
| `category` | `string` | Always `"iap"` for in-app purchases |
| `status` | `string` | Always `"Active"` in this response |
| `price` | `number` | One-time purchase price in USD |
| `playStoreId` | `string` | Google Play product ID; `""` if not available on Android |
| `appStoreId` | `string` | Apple App Store product ID; `""` if not available on iOS |

---

### ❌ 401 Unauthorized — Missing or Invalid Token

```json
{
  "message": "Unauthorized"
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
  "error": "Internal server error while fetching plans"
}
```

---

## Summary of Status Codes

| HTTP Status | Scenario |
|---|---|
| `200` | Plans and IAPs returned successfully (lists may be empty) |
| `401` | Missing / expired / invalid JWT token |
| `403` | Token valid but access denied by the authorizer |
| `500` | Internal DynamoDB error |
