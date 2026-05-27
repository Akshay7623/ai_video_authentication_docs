# IAP & Webhook API Docs

This folder documents the APIs related to in-app purchases (IAP) for Android and iOS developers.

| Document | Endpoint | Who calls it | Auth |
|---|---|---|---|
| [verify-purchase.md](./verify-purchase.md) | `POST /api/user/verify-purchase` | Mobile app — immediately after checkout | JWT required |
| [google-play-webhook.md](./google-play-webhook.md) | `POST /api/user/webhook/google-play` | Google Play servers (automatic) | None |
| [apple-appstore-webhook.md](./apple-appstore-webhook.md) | `POST /api/user/webhook/apple-appstore` | Apple servers (automatic) | None |

## How It Works

```
1. User completes purchase in-app
         │
         ▼
2. App calls POST /api/user/verify-purchase
   with the purchase token from the store SDK
         │
         ▼
3. Server verifies with the store and activates
   the subscription or adds credits immediately
         │
         ▼
4. Later — store sends lifecycle events automatically
   (renewals, cancellations, refunds, billing issues)
   directly to the webhook endpoints.
   The app does not need to do anything for step 4.
```

## What the App Needs to Do

- **Call `verify-purchase` once** right after the user completes checkout. This is the only IAP-related API call your app makes.
- **Refresh the user profile** (`GET /api/user/profile`) after a successful verify-purchase response and whenever the app returns to the foreground, to pick up any subscription status changes that came in via webhooks while the app was closed.
- The webhook endpoints are called by Google and Apple's servers automatically — **your app never calls them directly**.
