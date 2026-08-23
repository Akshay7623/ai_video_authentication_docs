# FCM Push Notification — Integration Guide for Mobile Developers

> All routes on this page require an `Authorization: Bearer <access_token>` header unless stated otherwise.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Firebase Project Setup](#2-firebase-project-setup)
   - [Android Setup](#21-android-setup)
   - [iOS Setup](#22-ios-setup)
3. [Generating an FCM Device Token](#3-generating-an-fcm-device-token)
4. [Test Notification Endpoint](#4-test-notification-endpoint)
5. [Using FCM Tokens in the App](#5-using-fcm-tokens-in-the-app)
6. [FCM Notification Payloads](#6-fcm-notification-payloads)
7. [Error Reference](#7-error-reference)

---

## 1. Overview

This app uses **Firebase Cloud Messaging (FCM)** to deliver push notifications to users when AI generation jobs complete or fail.

### How it works

```
Mobile App (Android / iOS)
      │
      │  1. Firebase SDK auto-generates FCM token on app launch
      │
      ▼
Backend API   ← 2. App sends FCM token + platform when starting a generation
      │
      │  3. AI job runs (could take 30s–3min for video)
      │
      ▼
Firebase FCM Servers
      │
      │  4. Backend sends notification via FCM when job completes
      ▼
User's Device  ← 5. App receives push notification with result URL
```

**The mobile app's responsibility:**
- Integrate the Firebase SDK
- Generate and provide an FCM device token when calling generation endpoints
- Handle incoming FCM data payloads (see [Section 6](#6-fcm-notification-payloads))

---

## 2. Firebase Project Setup

The Firebase project is already created and managed by the backend team. You only need the config file for your platform — **do not create a new Firebase project**.

> **Which project:** mobile (Android + iOS) uses **`silverenterprise`** — sender ID `303852353527`, Android package `com.aiapp`, iOS bundle `com.astound.aiapp`. The admin web dashboard uses a *separate* project (`ai-video-6d7e3`); the backend holds credentials for both and picks one based on the `platform` you send. If you are handed a config file whose `project_id` is not `silverenterprise`, it is the wrong one — sends will fail with `messaging/mismatched-credential`.

### 2.1 Android Setup

**What you need:** `google-services.json`

Ask the backend team to provide this file. Place it in your Android app module directory:

```
app/
├── google-services.json   ← place here
├── src/
│   └── ...
└── build.gradle
```

**In your project-level `build.gradle`:**
```groovy
plugins {
    id 'com.google.gms.google-services' version '4.4.0' apply false
}
```

**In your app-level `build.gradle`:**
```groovy
plugins {
    id 'com.google.gms.google-services'
}

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.7.0')
    implementation 'com.google.firebase:firebase-messaging'
}
```

### 2.2 iOS Setup

**What you need:** `GoogleService-Info.plist`

Ask the backend team to provide this file. Place it in the root of your Xcode project:

```
MyApp/
├── GoogleService-Info.plist   ← place here
├── AppDelegate.swift
└── ...
```

**In your `Podfile`:**
```ruby
pod 'Firebase/Messaging'
```

**In `AppDelegate.swift`:**
```swift
import Firebase

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        FirebaseApp.configure()
        return true
    }
}
```

> **iOS only:** The backend team also needs to configure APNs credentials in Firebase. If push notifications aren't working on iOS, check with the backend team that APNs is set up in the Firebase project.

---

## 3. Generating an FCM Device Token

The Firebase SDK automatically generates a unique FCM token for each device. You retrieve this token and pass it to the backend when starting a generation.

### Android (Kotlin)

```kotlin
import com.google.firebase.messaging.FirebaseMessaging

FirebaseMessaging.getInstance().token.addOnCompleteListener { task ->
    if (!task.isSuccessful) {
        Log.w(TAG, "Fetching FCM token failed", task.exception)
        return@addOnCompleteListener
    }
    val fcmToken = task.result
    Log.d(TAG, "FCM Token: $fcmToken")
    // Store this token and pass it to the backend
}
```

### iOS (Swift)

```swift
import FirebaseMessaging

Messaging.messaging().token { token, error in
    if let error = error {
        print("Error fetching FCM token: \(error)")
    } else if let token = token {
        print("FCM Token: \(token)")
        // Store this token and pass it to the backend
    }
}
```

> **Token refresh:** FCM tokens can change. Implement `MessagingDelegate` (iOS) or `FirebaseMessagingService` (Android) to detect token refreshes and update any stored token accordingly.

---

## 4. Test Notification Endpoint

Use this endpoint to verify your FCM token is valid and that push notifications are working **before** integrating with the full generation flow.

### Request

```
POST /api/user/test-notify
Authorization: Bearer <access_token>
Content-Type: application/json
```

**Body**

```json
{
  "token":    "<your FCM device token>",
  "platform": "android",
  "data": {
    "title": "Hello!",
    "body":  "Push notifications are working."
  }
}
```

**Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | `string` | YES | FCM device token generated by the Firebase SDK |
| `platform` | `string` | YES | `"android"` or `"ios"` |
| `data` | `object` | NO | Custom key-value payload. Defaults to a sample test message if omitted. `title` and `body` are lifted out and used for the banner FCM displays; any remaining keys arrive in the data map |

> **This endpoint is the one exception to the data-only rule in [Section 6](#6-fcm-notification-payloads).** It attaches a `notification` block, so the banner is rendered by FCM itself and appears even if the app has not implemented `onMessageReceived` yet. That is the point — it isolates "is my token valid and does push reach this handset" from "does my client render notifications correctly." Every other endpoint is data-only, so a banner appearing here does **not** mean production notifications will display.

**Minimal request (uses default payload):**

```json
{
  "token": "dAFe8N5GfAyVOKW1L_4jh9:APA91bErYXPEIF...",
  "platform": "android"
}
```

### Response 200 — Success

```json
{
  "success": true,
  "messageId": "projects/silverenterprise/messages/0:1234567890abcdef",
  "project": "silverenterprise",
  "platform": "android",
  "timestamp": "2026-05-29T06:48:00.000Z"
}
```

A push notification will arrive on your device within a few seconds.

### Error Responses

| Status | Body | Reason |
|--------|------|--------|
| 400 | `{ "success": false, "error": "Missing required field: token" }` | `token` not provided |
| 400 | `{ "success": false, "error": "Missing required field: platform" }` | `platform` not provided |
| 400 | `{ "success": false, "error": "Invalid platform. Must be 'android' or 'ios'" }` | Wrong platform value |
| 400 | `{ "success": false, "error": "FCM token is invalid or expired...", "code": "messaging/registration-token-not-registered" }` | Token is stale — regenerate it |
| 400 | `{ "success": false, "error": "FCM token format is invalid.", "code": "messaging/invalid-registration-token" }` | Token is malformed |
| 401 | `{ "message": "Unauthorized" }` | Missing or invalid JWT token |
| 500 | `{ "success": false, "error": "<message>", "code": "<fcm-error-code>" }` | FCM server-side error |

### Checklist — if notification doesn't arrive

- [ ] `200` response received with a valid `messageId`?
- [ ] App is running in **foreground or background** (not force-killed)?
- [ ] Firebase SDK initialized correctly in the app?
- [ ] Correct `google-services.json` / `GoogleService-Info.plist` in use?
- [ ] iOS: APNs permission granted by user?
- [ ] iOS: APNs key configured in Firebase by backend team?
- [ ] Token is fresh? (Re-fetch and try again)

---

## 5. Using FCM Tokens in the App

### When to pass the FCM token

Pass the FCM token when calling the **Initiate Generation** endpoint. The backend stores it and uses it to send the push notification when the job completes.

```
POST /api/user/generate/initiate
Authorization: Bearer <access_token>
Content-Type: application/json
```

```json
{
  "workflowId":    "f5536d77-c551-44fc-a2ca-fc3fdf5da528",
  "subTemplateId": "4b8ffff8-0c32-4fbb-a56f-92f4e076e238",
  "platform":      "android",
  "fcmToken":      "dAFe8N5GfAyVOKW1L_4jh9:APA91bErYXPEIF..."
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `platform` | YES (for FCM) | `"android"` or `"ios"` |
| `fcmToken` | NO | If omitted, no push notification is sent. App must poll history instead |

> If `fcmToken` is not provided, the backend will not send any push notification. The app can still poll `/api/user/generation-history` to check job status.

---

## 6. FCM Notification Payloads

All notifications from this backend arrive as **data-only FCM messages** (no `notification` block) — the sole exception being `POST /api/user/test-notify`, described in [Section 4](#4-test-notification-endpoint). This means:
- The app must handle them in a **background service / message handler**
- The OS will **not** automatically display a notification banner
- You are responsible for building and showing the local notification from the data payload

This is deliberate: the payloads below carry `generationId`, `publicUrl`, and `subTemplateId` so the client can deep-link to the finished job and render a thumbnail. FCM's automatic banner cannot do either.

> **Android:** data messages are delivered to `FirebaseMessagingService.onMessageReceived()` and nowhere else. Without that override the payload arrives and is silently discarded — no tray entry, no error. Note also that data messages are **not** delivered at all while the app is force-stopped.

> **iOS:** these are background pushes (`apns-push-type: background`, `apns-priority: 5`). Enable the Remote notifications background mode and handle them in `didReceiveRemoteNotification`.

### 6.1 Generation Complete

Sent when an image or video generation job finishes successfully.

```json
{
  "type":          "generation_complete",
  "generationId":  "8dc4e5cf-d180-42df-856d-3b6cc2c8fd7c",
  "finalStatus":   "complete",
  "publicUrl":     "https://d397ajnx16aos2.cloudfront.net/outputs/...?Expires=...&Key-Pair-Id=...&Signature=...",
  "workflowId":    "ac005294-d724-4380-8c31-b68a79978b8b",
  "subTemplateId": "4b8ffff8-0c32-4fbb-a56f-92f4e076e238",
  "status":        "complete"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Always `"generation_complete"` |
| `generationId` | `string` | Matches the `generationId` from the initiate response |
| `finalStatus` | `string` | `"complete"` |
| `publicUrl` | `string` | Signed CloudFront URL — valid for **14 days** |
| `workflowId` | `string` | Which workflow was used |
| `subTemplateId` | `string` | Which sub-template was used |
| `status` | `string` | `"complete"` |

**Recommended app behavior:**

| Situation | What to do |
|-----------|------------|
| App is in foreground | Match `generationId` to pending job → display `publicUrl` |
| App is in background | Show local notification → on tap, navigate to result |
| App was force-killed | On next launch, refresh `/api/user/generation-history` |

### 6.2 Generation Failed

Sent when a job fails. Credits are automatically refunded by the backend.

```json
{
  "type":         "generation_failed",
  "generationId": "8dc4e5cf-d180-42df-856d-3b6cc2c8fd7c",
  "reason":       "runpod_job_failed: worker timeout"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Always `"generation_failed"` |
| `generationId` | `string` | Matches the `generationId` from the initiate response |
| `reason` | `string` | Human-readable failure reason (for logging/debugging) |

**Recommended app behavior:**
- Match `generationId` → mark job as failed in local state
- Show an error message with a retry option
- Credits have already been refunded automatically

### 6.3 Video Generation Started

Sent **only** when `POST /api/user/generate/start` takes longer than 30 seconds for a *video* job.
API Gateway's 29-second cap means the HTTP response is already lost by then, so the backend pushes
this instead to tell the app the job is genuinely underway.

```json
{
  "type":         "video_generation_started",
  "generationId": "8dc4e5cf-d180-42df-856d-3b6cc2c8fd7c",
  "finalStatus":  "processing"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `string` | Always `"video_generation_started"` |
| `generationId` | `string` | Matches the `generationId` from the initiate response |
| `finalStatus` | `string` | Job state at the moment the fallback fired |

**Recommended app behavior:**
- This is **not** a completion. No `publicUrl` is included — do not show a result.
- Treat it as confirmation that a request whose HTTP call timed out is still running, and keep
  showing progress.
- The real result arrives later as a separate `generation_complete` (§6.1) or
  `generation_failed` (§6.2) message.

> An image job that crosses the same 30-second threshold sends `generation_complete` with a
> `publicUrl` instead — images finish inside the call, videos do not.

---

## 7. Error Reference

### Common FCM token errors

| Error Code | Meaning | Fix |
|------------|---------|-----|
| `messaging/registration-token-not-registered` | Token is expired or unregistered | Re-fetch token from Firebase SDK |
| `messaging/invalid-registration-token` | Token string is malformed | Ensure you're copying the full token |
| `messaging/mismatched-credential` | Firebase project mismatch | Ensure you're using the correct `google-services.json` / `GoogleService-Info.plist` |

### HTTP errors from test endpoint

| Status | Meaning |
|--------|---------|
| 401 | JWT token missing or expired — re-login |
| 400 | Bad request body — check required fields |
| 500 | Firebase Admin error — contact backend team |

---

## Quick Reference

```
Firebase Project ID:   silverenterprise        (mobile — Android + iOS)
FCM Sender ID:         303852353527
Android package:       com.aiapp
iOS bundle:            com.astound.aiapp

Test endpoint:         POST /api/user/test-notify   (requires JWT auth)
Initiate generation:   POST /api/user/generate/initiate
```

