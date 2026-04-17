# User API Documentation

> **Base URL:** `https://{UserHttpApi}.execute-api.{region}.amazonaws.com`
>
> All endpoints are prefixed as shown. JSON is the only supported content type for request/response bodies unless noted otherwise.

---

## Table of Contents

1. [Authentication Overview](#authentication-overview)
2. [Common Error Responses](#common-error-responses)
3. [Endpoints](#endpoints)
   - [Health Check](#1-health-check)
   - [Send OTP](#2-send-otp)
   - [Verify OTP](#3-verify-otp)
   - [Resend OTP](#4-resend-otp)
   - [Social Auth (Google / Apple)](#5-social-auth-google--apple)
   - [Refresh Token](#6-refresh-token)
   - [Logout](#7-logout)
   - [Get Profile](#8-get-profile)
4. [User Object Reference](#user-object-reference)
5. [Rate Limits & Constraints](#rate-limits--constraints)

---

## Authentication Overview

The User API uses a **custom JWT authorizer** (`UserJwtAuthorizerFunction`).

| Mechanism | Detail |
|-----------|--------|
| Algorithm | `HS256` |
| Header | `Authorization: Bearer <accessToken>` |
| Access Token TTL | **6 hours** |
| Refresh Token TTL | **15 days** |
| Token rotation | Full rotation on every `/refresh-token` call |

Endpoints that require authentication are marked with 🔒. Unauthenticated endpoints are marked with 🔓.

---

## Common Error Responses

These shapes can be returned by **any** endpoint regardless of the specific endpoint logic.

| Status Code | When it occurs | Body shape |
|-------------|---------------|------------|
| `400` | Request body fails schema validation | `{ "success": false, "message": "Invalid input", "errors": { "fieldErrors": {}, "formErrors": [] } }` |
| `403` | The user account is banned | `{ "success": false, "message": "Your account has been banned. Please contact support." }` |
| `500` | Unhandled internal error | `{ "success": false, "message": "Internal server error" }` |

---

## Endpoints

---

### 1. Health Check

> 🔓 No authentication required

**`GET /api/user/hello`**

A simple liveness probe. Useful to verify the API Gateway and Lambda are reachable.

#### Request

No headers, query parameters, or body required.

#### Responses

| Status | Description |
|--------|-------------|
| `200` | Service is up |

##### `200 OK`
```json
{
  "message": "Hello from User API!"
}
```

---

### 2. Send OTP

> 🔓 No authentication required

**`POST /api/auth/email/send-otp`**

Initiates email-based authentication. Depending on whether the email already exists in the system, this endpoint either **creates a new unverified user** (sign-up flow) or **updates the OTP for an existing user** (login flow). In both cases a 6-digit OTP is sent to the provided email address.

- OTP validity window: **120 seconds**
- Resend cooldown: **60 seconds** between requests

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | `string` | ✅ | A valid email address. Normalised to lowercase. |

**Example**
```json
{
  "email": "jane.doe@example.com"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | OTP sent successfully |
| `400` | Validation error (missing / malformed email) |
| `403` | Account is banned |
| `409` | Race condition — user was just created concurrently; retry |
| `429` | Resend cooldown active |
| `500` | Internal server error |
| `502` | Downstream email service failed to send |

##### `200 OK` — New user (sign-up)
```json
{
  "success": true,
  "message": "OTP sent to your email.",
  "expiresIn": 120,
  "isNewUser": true
}
```

##### `200 OK` — Existing user (login)
```json
{
  "success": true,
  "message": "OTP sent to your email.",
  "expiresIn": 120,
  "isNewUser": false
}
```

##### `429 Too Many Requests` — Cooldown active
```json
{
  "success": false,
  "message": "Please wait 42 second(s) before requesting another OTP.",
  "retryAfter": 42
}
```

##### `409 Conflict` — Concurrent registration race
```json
{
  "success": false,
  "message": "User already exists. Please request OTP again."
}
```

##### `502 Bad Gateway` — Email delivery failure
```json
{
  "success": false,
  "message": "Failed to send OTP email. Please retry."
}
```

---

### 3. Verify OTP

> 🔓 No authentication required

**`POST /api/auth/email/verify-otp`**

Validates the OTP that was sent to the user's email. On success, issues a short-lived **access token** (6 h) and a long-lived **refresh token** (15 d). The OTP and all related temporary fields are removed from the database after successful verification.

- Grace period after OTP expiry: **3 minutes** (the OTP can still be accepted within this window)
- Maximum verification attempts: **5** (a new OTP must be requested after 5 failures)

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | `string` | ✅ | The same email address used in `send-otp`. |
| `otp` | `string` | ✅ | The 6-digit OTP received by email. |

**Example**
```json
{
  "email": "jane.doe@example.com",
  "otp": "482951"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | OTP verified; tokens issued |
| `400` | Validation error |
| `401` | OTP is incorrect |
| `403` | Account is banned |
| `404` | User not found (no OTP was ever requested for this email) |
| `410` | OTP has expired (including grace period) |
| `429` | Maximum OTP attempts exceeded |
| `500` | Internal server error |

##### `200 OK`
```json
{
  "success": true,
  "message": "OTP verified successfully.",
  "accessToken": "<jwt>",
  "refreshToken": "<jwt>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `accessToken` | `string` | HS256 JWT. Include in `Authorization: Bearer <accessToken>` for protected endpoints. Expires in **6 hours**. |
| `refreshToken` | `string` | HS256 JWT. Use with `/auth/refresh-token` to obtain new tokens. Expires in **15 days**. Store securely (e.g., HttpOnly cookie or secure storage). |

##### `401 Unauthorized` — Wrong OTP
```json
{
  "success": false,
  "message": "Invalid OTP. Please check and try again."
}
```

##### `404 Not Found`
```json
{
  "success": false,
  "message": "User not found. Please request an OTP first."
}
```

##### `410 Gone` — OTP expired
```json
{
  "success": false,
  "message": "OTP has expired. Please request a new one."
}
```

##### `429 Too Many Requests` — Max attempts reached
```json
{
  "success": false,
  "message": "Maximum OTP attempts reached. Please request a new OTP."
}
```

---

### 4. Resend OTP

> 🔓 No authentication required

**`POST /api/auth/email/resend-otp`**

Generates and sends a fresh OTP to a registered email address. Resets the attempt counter. Subject to the same **60-second cooldown** as `send-otp`.

> **Note:** Unlike `send-otp`, this endpoint does **not** create new users. The email must already exist in the system.

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | `string` | ✅ | Email address of the user who needs a new OTP. |

**Example**
```json
{
  "email": "jane.doe@example.com"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | New OTP sent |
| `400` | Validation error |
| `403` | Account is banned |
| `404` | User not found |
| `429` | Resend cooldown active |
| `500` | Internal server error |
| `502` | Downstream email delivery failure |

##### `200 OK`
```json
{
  "success": true,
  "message": "OTP resent successfully.",
  "expiresIn": 120
}
```

##### `429 Too Many Requests` — Cooldown active
```json
{
  "success": false,
  "message": "Please wait 38 second(s) before requesting another OTP.",
  "retryAfter": 38
}
```

##### `404 Not Found`
```json
{
  "success": false,
  "message": "User not found. Please request an OTP first."
}
```

---

### 5. Social Auth (Google / Apple)

> 🔓 No authentication required

**`POST /api/auth/social`**

Authenticates a user using a platform-issued identity token (Google ID Token for Android, Apple ID Token for iOS). Automatically creates a new user record on first sign-in, or signs in an existing user. Returns access and refresh tokens on success.

| Platform | Provider | Token type |
|----------|----------|-----------|
| `android` | Google | Google ID Token (`idToken`) from Firebase / Google Sign-In SDK |
| `ios` | Apple | Apple ID Token from Sign in with Apple SDK |

> **Apple note:** Apple only returns the user's `name` and `email` on the **very first** sign-in. After the first sign-in, the name is no longer available from the token. The full name is stored during initial account creation.

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | `string` | ✅ | The platform identity token obtained from the native SDK. |
| `platform` | `string` | ✅ | Must be `"android"` (Google SSO) or `"ios"` (Apple SSO). |
| `fullName` | `string` | ❌ | User's display name. Only relevant for iOS first-time Apple sign-in where the token does not carry the name. |

**Example — Android (Google)**
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "platform": "android"
}
```

**Example — iOS (Apple)**
```json
{
  "token": "eyJraWQiOiJXNldjT0tCIiwiYWxnIjoiUlMyNTYifQ...",
  "platform": "ios",
  "fullName": "Jane Doe"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | Authenticated successfully (new or existing user) |
| `400` | Validation error or email not available in Apple token |
| `401` | Token verification failed (invalid or expired platform token) |
| `403` | Account is banned |
| `500` | Internal server error |

##### `200 OK` — New user created
```json
{
  "success": true,
  "message": "Account created successfully.",
  "accessToken": "<jwt>",
  "refreshToken": "<jwt>"
}
```

##### `200 OK` — Existing user signed in
```json
{
  "success": true,
  "message": "Logged in successfully.",
  "accessToken": "<jwt>",
  "refreshToken": "<jwt>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `accessToken` | `string` | HS256 JWT valid for **6 hours**. |
| `refreshToken` | `string` | HS256 JWT valid for **15 days**. |

##### `401 Unauthorized` — Bad platform token
```json
{
  "success": false,
  "message": "Invalid Google token. Please try again."
}
```
```json
{
  "success": false,
  "message": "Invalid Apple token. Please try again."
}
```

##### `400 Bad Request` — Apple email not available
```json
{
  "success": false,
  "message": "Could not retrieve email from Apple token. Ensure email permission is granted."
}
```

---

### 6. Refresh Token

> 🔓 No authentication required *(the refresh token itself is the credential)*

**`POST /api/auth/refresh-token`**

Exchanges a valid, non-expired refresh token for a **new access token and a new refresh token** (token rotation). The previous refresh token is immediately invalidated.

> ⚠️ **Token rotation:** Each call issues a brand-new refresh token. The old refresh token cannot be reused. If two concurrent refresh calls are made, the second will fail with `401`.

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refreshToken` | `string` | ✅ | The refresh token previously issued by `verify-otp`, `social`, or a prior `refresh-token` call. |

**Example**
```json
{
  "refreshToken": "<jwt>"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | New token pair issued |
| `400` | Validation error (missing field) |
| `401` | Token expired, invalid, already rotated, or no active session |
| `403` | Account is banned |
| `404` | User not found |
| `500` | Internal server error |

##### `200 OK`
```json
{
  "success": true,
  "accessToken": "<new-jwt>",
  "refreshToken": "<new-jwt>"
}
```

##### `401 Unauthorized` — Token expired
```json
{
  "success": false,
  "message": "Refresh token has expired. Please log in again."
}
```

##### `401 Unauthorized` — Token invalid / already rotated
```json
{
  "success": false,
  "message": "Refresh token is invalid or has already been rotated."
}
```

##### `401 Unauthorized` — No active session
```json
{
  "success": false,
  "message": "No active session. Please log in again."
}
```

---

### 7. Logout

> 🔓 No authentication required *(the refresh token itself is the credential)*

**`POST /api/auth/logout`**

Invalidates the user's current session by removing the stored refresh token from the database. The access token will remain technically valid until its natural expiry (up to 6 hours), but no new tokens can be issued without logging in again.

> If the provided refresh token is already invalid or expired, the endpoint still returns `200` with a note that the session was already invalid — this prevents confusing UX flows.

#### Request

**Headers**
```
Content-Type: application/json
```

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refreshToken` | `string` | ✅ | The active refresh token to be invalidated. |

**Example**
```json
{
  "refreshToken": "<jwt>"
}
```

#### Responses

| Status | Description |
|--------|-------------|
| `200` | Logged out (or session was already invalid) |
| `400` | Validation error |
| `500` | Internal server error |

##### `200 OK` — Session cleared
```json
{
  "success": true,
  "message": "Logged out successfully."
}
```

##### `200 OK` — Session was already invalid
```json
{
  "success": true,
  "message": "Logged out successfully (session was already invalid)."
}
```

---

### 8. Get Profile

> 🔒 **Authentication required** — `Authorization: Bearer <accessToken>`

**`GET /api/user/profile`**

Returns the authenticated user's profile. Sensitive fields (OTP state, hashed tokens, failed login counters, etc.) are stripped before the response is returned.

#### Request

**Headers**
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | ✅ | `Bearer <accessToken>` — JWT issued by `verify-otp`, `social`, or `refresh-token`. |

No request body is needed.

#### Responses

| Status | Description |
|--------|-------------|
| `200` | Profile returned |
| `401` | Missing or invalid `Authorization` header / token |
| `403` | Account is banned |
| `404` | User record not found |
| `500` | Internal server error |

##### `200 OK`
```json
{
  "success": true,
  "profile": {
    "email": "jane.doe@example.com",
    "fullName": "Jane Doe",
    "isVerified": true,
    "role": "user",
    "subscriptionStatus": "free",
    "credits": 0,
    "lifetimeCreditsUsed": 0,
    "status": "active",
    "authProvider": "email",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:45:00.000Z"
  }
}
```

> **Stripped fields:** `passwordHash`, `otp`, `otpExpiresAt`, `otpLastSentAt`, `otpAttempts`, `resetOtp`, `resetOtpHash`, `resetOtpExpiresAt`, `resetOtpLastSentAt`, `resetOtpAttempts`, `refreshToken`, `failedLoginAttempts`, `accountLockedUntil`.

##### `401 Unauthorized` — Token missing
```json
{
  "success": false,
  "message": "Unauthorized missing identity context."
}
```

##### `404 Not Found`
```json
{
  "success": false,
  "message": "User not found."
}
```

---

## User Object Reference

The following describes all fields stored in the `UsersTable` DynamoDB table. The table uses `email` as its partition key (hash key).

| Field | Type | Description |
|-------|------|-------------|
| `email` | `string` | **Partition key.** Normalised to lowercase. |
| `fullName` | `string` | Display name of the user. |
| `isVerified` | `boolean` | `true` once the user completes their first OTP verification or social sign-in. |
| `role` | `string` | User role. Always `"user"` for end-user accounts. |
| `subscriptionStatus` | `string` | Subscription tier. Default: `"free"`. |
| `credits` | `number` | Current credit balance. |
| `lifetimeCreditsUsed` | `number` | Total credits consumed since account creation. |
| `status` | `string` | Account status. Values: `"active"` \| `"banned"`. |
| `authProvider` | `string` | How the account was created. Values: `"email"` \| `"android"` \| `"ios"`. |
| `providerId` | `string` | (Social accounts only) Provider subject ID (`sub` claim). |
| `refreshToken` | `string` | Bcrypt hash of the active refresh token. Not returned in any API response. |
| `otp` | `string` | SHA-256 + HMAC hash of the pending OTP. Removed after verification. |
| `otpExpiresAt` | `number` | Unix timestamp (seconds) when the OTP expires. |
| `otpLastSentAt` | `number` | Unix timestamp of the most recent OTP send. Used to enforce the 60 s cooldown. |
| `otpAttempts` | `number` | Failed verification attempts since last OTP send. Resets on new OTP. Max 5. |
| `createdAt` | `string` | ISO 8601 timestamp of account creation. |
| `updatedAt` | `string` | ISO 8601 timestamp of the last update. |

---

## Rate Limits & Constraints

| Constraint | Value | Scope |
|------------|-------|-------|
| OTP validity | 120 seconds | Per OTP |
| OTP grace period | +3 minutes | After `otpExpiresAt` |
| OTP resend cooldown | 60 seconds | Per user, per OTP request |
| Max OTP verification attempts | 5 | Per OTP; resets when a new OTP is issued |
| Access token TTL | 6 hours | Per token |
| Refresh token TTL | 15 days | Per token; rotated on every use |

---

## Authentication Flow Diagrams

### Email OTP Flow

```
Client                          API
  │                              │
  │── POST /auth/email/send-otp ─▶│  Create/update user record
  │                              │  Send OTP email
  │◀─ 200 { isNewUser, expiresIn }│
  │                              │
  │── POST /auth/email/verify-otp▶│  Validate OTP
  │                              │  Issue access + refresh tokens
  │◀─ 200 { accessToken, refreshToken } │
  │                              │
  │── GET /api/user/profile ─────▶│  Authorizer validates accessToken
  │    Authorization: Bearer ... │  Return profile
  │◀─ 200 { profile }            │
```

### Token Refresh Flow

```
Client                          API
  │                              │
  │  (accessToken expires)       │
  │                              │
  │── POST /auth/refresh-token ──▶│  Validate refreshToken
  │    { refreshToken }          │  Issue new token pair
  │◀─ 200 { accessToken, refreshToken } │
  │    (old refreshToken is now invalid) │
```

### Social Auth Flow

```
Client                          API
  │                              │
  │  (User taps "Sign in with Google/Apple")
  │◀─── Platform SDK ────────────│  Returns idToken
  │                              │
  │── POST /auth/social ─────────▶│  Verify idToken with Google/Apple
  │    { token, platform }       │  Upsert user in DynamoDB
  │                              │  Issue access + refresh tokens
  │◀─ 200 { accessToken, refreshToken } │
```

---
