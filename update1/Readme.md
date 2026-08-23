# Changes Since the Last App Version


## TL;DR

| | |
|---|---|
| **Endpoints removed** | None |
| **Endpoints with changed contracts** | None |
| **New endpoints for the app** | 3 |
| **Breaking field changes** | 1 (`lifetimeCreditsUsed` → `creditUsed`) |
| **Auth changes** | None |
| **IAP changes** | None |
| **Behavioural change to watch** | Generated media is now actually deleted after 14 days (§5) |

The last release's integration still works. This is an **additive** update — the work is adopting
push notifications, one field rename, and handling expired media.

---

## 1. What Did Not Change

Verified content-identical. If the app already handles these, nothing needs revisiting.

| Area | Status |
|---|---|
| Auth — all 8 endpoints | Identical but for one field (§2) |
| Verify purchase | **Byte-identical** |
| Google Play webhook | **Byte-identical** |
| App Store webhook | **Byte-identical** |
| IAP overview | Retitled only |
| Profile, templates, generation, plans | Identical |
| Error reference, generation lifecycle | Identical |

Auth constants are unchanged and can be relied on as-is:

| Constant | Value |
|---|---|
| Access token TTL | 6 hours |
| Refresh token TTL | 15 days (full rotation every refresh) |
| OTP validity | 120 seconds |
| OTP grace period | +3 minutes after expiry |
| OTP resend cooldown | 60 seconds |
| Max OTP attempts | 5 |

The diff of the old user API doc against the current one contains **no removed content** — only the
push-notification section was added.

---

## 2. The One Breaking Change

The user profile object renamed a field:

```diff
- "lifetimeCreditsUsed": 0
+ "creditUsed": 0
```

This affects `GET /api/user/profile` and the stored user record. Same meaning — total credits
consumed since account creation.

> **The backend already shipped this.** A search across the whole codebase finds `creditUsed`
> throughout and **zero** occurrences of `lifetimeCreditsUsed`. The old docs were behind the code.
> The mobile app is the only consumer that may still be parsing the old name — if it is, that field
> is silently reading as null in production today.

---

## 3. New Endpoints

Three endpoints exist that the last app version had no knowledge of.

| Method & Path | Purpose | Priority |
|---|---|---|
| `POST /api/user/device-token` | Register the device's FCM token | **Required for push** |
| `DELETE /api/user/device-token` | Unregister on sign-out | **Required for push** |
| `POST /api/user/test-notify` | Push integration test | Dev aid |

App-callable endpoint count: **15 → 18**.

---

## 4. Push Notifications — The Real Functional Shift

This is the substantive change and it needs app work.

### What was wrong before

An FCM token only ever reached the backend attached to a single `/api/user/generate/initiate` call,
and was stored on that one generation row. That was enough to push the result of *that* job and
nothing else. There was no way to reach a user between jobs, and a broadcast had no addressees at
all.

### What exists now

Devices are stored keyed `(email, token)`. One account can hold several devices.
**Nothing can be delivered to a user who has never registered a device.**

### 4.1 Register a device

```
POST /api/user/device-token
Authorization: Bearer <token>
```
```json
{ "token": "<fcm-registration-token>", "platform": "android" }
```

Send `android` or `ios`.

**Response 200**
```json
{ "success": true, "registered": true, "platform": "android" }
```

> **Call this on every app launch, not just the first.** FCM reissues registration tokens
> periodically — on reinstall, on restore to a new device, or at its own discretion — and a stale
> token silently stops receiving. Registration is an upsert keyed on `(email, token)`, so repeat
> calls are safe.

| Status | Response |
|---|---|
| 400 | `{ "success": false, "error": "token is required" }` |
| 400 | `{ "success": false, "error": "platform must be 'android', 'ios' or 'web'" }` |
| 401 | `{ "success": false, "error": "Unauthorized" }` |

### 4.2 Unregister a device

```
DELETE /api/user/device-token
Authorization: Bearer <token>
```
```json
{ "token": "<fcm-registration-token>" }
```

**Response 200**
```json
{ "success": true, "removed": 1 }
```

> **Call this on sign-out.** Otherwise the next person to sign in on that device keeps receiving the
> previous user's notifications. Omitting `token` clears every device registered to the account,
> which is the right call when signing out without a token to name.

### 4.3 New inbound payload

The app must now handle a broadcast notification, which the old version never received:

```json
{
  "type": "admin_notification",
  "notificationId": "…",
  "title": "…",
  "body": "…",
  "link": "…",
  "imageUrl": "…"
}
```

Two details that bite:

- `link` and `imageUrl` are **absent, not empty**, when unset — parse defensively.
- Content is sent in **both** the notification and data blocks, so it renders whether the app is
  backgrounded or handling the message itself.

### 4.4 Token lifecycle

FCM rejections meaning *this token will never work again* (uninstalled app, malformed token) cause
the backend to delete the device row. Transient failures (quota, outage, network) never do — a bad
afternoon at FCM must not unsubscribe the fleet. Re-registering on next launch restores a pruned
device.

Full setup detail is in [fcm-integration.md](./fcm-integration.md).

---

## 5. 14-Day Retention Is Now Enforced

No API contract changed here, but the app's runtime behaviour does.

The product has always promised generated media stays available for 14 days. Previously nothing
enforced it: records and their S3 objects accumulated indefinitely while
`/api/user/generation-history` merely filtered older entries out of the response. Media that had
"expired" was still sitting there and still resolvable.

Server-side cleanup now deletes the record together with every S3 object it points at.

**What this means for the app:**

- A `publicUrl` cached locally beyond 14 days will stop resolving. Treat media fetch failures on old
  entries as expected, not as an error state.
- Don't build features that assume indefinite media availability — no permanent local gallery backed
  by remote URLs, no "all time" history view fed from `publicUrl`.
- `/api/user/generation-history` returns the last 14 days, which is now the true extent of what
  exists rather than a display filter.

---

## 6. Documentation to Read

| Document | Why |
|---|---|
| [fcm-integration.md](./fcm-integration.md) | Push setup in depth |

Prefer `mobile-integration.md` over `user_api_documentation.md`; the latter is backend-oriented and
covers the same endpoints from the server's point of view.

---

## 7. App Update Checklist

- [ ] Rename `lifetimeCreditsUsed` → `creditUsed` wherever the profile is parsed
- [ ] Call `POST /api/user/device-token` on **every** app launch after authentication
- [ ] Call `DELETE /api/user/device-token` on sign-out, before clearing tokens
- [ ] Handle inbound `type: "admin_notification"` payloads
- [ ] Treat `link` and `imageUrl` as optional/absent, not empty strings
- [ ] Verify push renders both backgrounded and foregrounded
- [ ] Use `POST /api/user/test-notify` to confirm the integration end to end
- [ ] Handle expired media gracefully — cached URLs past 14 days will fail
- [ ] No auth changes needed — token handling is unchanged
- [ ] No IAP changes needed — purchase flow is unchanged
