# Google Play Webhook

This endpoint receives real-time notifications from Google Play for both **IAP (one-time credit packs)** and subscriptions. It is called automatically by Google's infrastructure via Cloud Pub/Sub — **your Android app never calls this endpoint directly**.

---

## Endpoint

```
POST /api/user/webhook/google-play
```

**Auth:** None — called by Google's servers, not the app  
**Your app's role:** None. Call `GET /api/user/profile` when the app comes to the foreground to pick up any changes.

---

## What This Webhook Handles

Google Play sends one of three notification shapes depending on what happened.

### IAP (One-Time Product) events

Fired when a user buys or cancels a one-time credit pack.

| `notificationType` | Name | What it means | Action taken |
|---|---|---|---|
| `1` | `ONE_TIME_PRODUCT_PURCHASED` | Payment completed | Idempotency check → credits added only if `verify-purchase` has not already run for this token |
| `2` | `ONE_TIME_PRODUCT_CANCELED` | Purchase canceled before fulfillment (pending payment failed or timed out) | Logged — no credits were ever added, nothing to revert |

### Voided purchase events

Fired when Google or the developer issues a refund after the purchase was already fulfilled.

| `productType` | What it means | Action taken |
|---|---|---|
| `0` (subscription) | Subscription purchase voided | `subscriptionStatus` set to `expired` |
| `1` (IAP) | Credit pack purchase voided / refunded | Credits deducted from user (reset to `0` if user already spent them) |

### Subscription events

| Event | What it means | Resulting `subscriptionStatus` |
|---|---|---|
| `SUBSCRIPTION_PURCHASED` | New subscription started | `active` |
| `SUBSCRIPTION_RENEWED` | Auto-renewed successfully | `active` |
| `SUBSCRIPTION_RECOVERED` | Recovered after billing hold | `active` |
| `SUBSCRIPTION_IN_GRACE_PERIOD` | Billing failed, grace period active | `active` |
| `SUBSCRIPTION_RESTARTED` | User reactivated a cancelled subscription | `active` |
| `SUBSCRIPTION_PRICE_CHANGE_CONFIRMED` | User accepted a price change | `active` |
| `SUBSCRIPTION_CANCELED` | User cancelled auto-renew (still active until period end) | `canceling` |
| `SUBSCRIPTION_EXPIRED` | Subscription has fully expired | `expired` |
| `SUBSCRIPTION_REVOKED` | Subscription revoked by Google | `expired` |
| `SUBSCRIPTION_ON_HOLD` | Account on hold due to billing failure | `on_hold` |
| `SUBSCRIPTION_PAUSED` | User paused the subscription | `paused` |

---

## IAP Lifecycle

### Normal flow — `verify-purchase` already ran

```mermaid
flowchart TD
    A([User buys credit pack]) --> B[Google processes payment]
    B --> C[App gets purchaseToken]
    C --> D[App calls POST /api/user/verify-purchase]
    D --> E[Credits added\nToken written to UserPurchasesTable]
    E --> F[Google fires webhook\nnotificationType = 1]
    F --> G{Token already in\nUserPurchasesTable?}
    G -- Yes --> H([Skip — already credited\nNo double-credit])
```

### Fallback flow — app crashed before calling `verify-purchase`

```mermaid
flowchart TD
    A([User buys credit pack]) --> B[Google processes payment]
    B --> C[App gets purchaseToken]
    C --> D[App crashes or loses network]
    D --> E[verify-purchase never called\nToken NOT in UserPurchasesTable]
    E --> F[Google fires webhook\nnotificationType = 1]
    F --> G{Token already in\nUserPurchasesTable?}
    G -- No --> H[Add credits + write token\natomically via TransactWrite]
    H --> I([User gets credits])
    I --> J[App relaunches\ncalls verify-purchase again]
    J --> K([ACID condition blocks\ndouble-credit])
```

### Refund / Void flow

```mermaid
flowchart TD
    A([Google voids IAP purchase]) --> B[voidedPurchaseNotification\nproductType = 1]
    B --> C[Look up user from UserPurchasesTable\nusing purchaseToken]
    C --> D[Look up plan.credits from PlansTable\nusing productId]
    D --> E{User has enough\ncredits?}
    E -- Yes --> F[credits = credits - plan.credits]
    E -- No, already spent --> G[credits = 0]
    F --> H([Done])
    G --> H
```

---

## Subscription Lifecycle

```mermaid
stateDiagram-v2
    direction TB

    state "Pending" as Pending
    state "Active\nsubscriptionStatus: active" as Active
    state "Grace Period\nsubscriptionStatus: active" as GracePeriod
    state "On Hold\nsubscriptionStatus: on_hold" as OnHold
    state "Paused\nsubscriptionStatus: paused" as Paused
    state "Canceling\nsubscriptionStatus: canceling" as Canceling
    state "Expired\nsubscriptionStatus: expired" as Expired

    [*] --> Pending : SUBSCRIPTION_PURCHASED
    Pending --> Active : payment confirmed
    Active --> Active : SUBSCRIPTION_RENEWED
    Active --> GracePeriod : billing failed
    GracePeriod --> Active : payment recovered
    GracePeriod --> OnHold : grace period ends
    OnHold --> Active : SUBSCRIPTION_RECOVERED
    OnHold --> Expired : SUBSCRIPTION_REVOKED
    Active --> Paused : SUBSCRIPTION_PAUSED
    Paused --> Active : SUBSCRIPTION_RESTARTED
    Active --> Canceling : SUBSCRIPTION_CANCELED
    Canceling --> Expired : SUBSCRIPTION_EXPIRED
    Active --> Expired : SUBSCRIPTION_EXPIRED or SUBSCRIPTION_REVOKED
    Expired --> [*]
```

---

## What Your App Should Do

Your app does not react to these webhook events in real time. Instead:

1. **After `verify-purchase` returns 200** — refresh the user profile to show the new credit balance.
2. **On app foreground / resume** — call `GET /api/user/profile` to pick up any changes that happened while the app was closed.
3. **Read the `credits` field** from the profile response to show the current balance.
4. **Read `subscriptionStatus`** if you also use subscriptions:

| `subscriptionStatus` | Recommended UI |
|---|---|
| `active` | Full access |
| `canceling` | Show "Renew?" prompt — user still has access |
| `on_hold` | Prompt to fix payment method in Play Store |
| `paused` | Inform user subscription is paused |
| `expired` | Show paywall / upsell screen |
| *(absent / null)* | User has never subscribed |

---

## PlansTable Requirement

The webhook resolves the number of credits to add or deduct by scanning `PlansTable` for a row where `playStoreId` matches the purchased `sku` / `productId`.

**Each plan row must have:**

| Field | Example | Description |
|---|---|---|
| `playStoreId` | `"credits_100"` | Must exactly match the Product ID in Google Play Console |
| `credits` | `100` | Number of credits to add (purchase) or deduct (refund/void) |

If `playStoreId` is missing or doesn't match, the webhook logs a warning and takes no action.

---

## Notes

- **No double-crediting** — the webhook checks `UserPurchasesTable` before adding credits. If `verify-purchase` already ran for that token, the webhook skips silently.
- **Pub/Sub retries** — if your server returns anything other than `2xx`, Google retries with exponential backoff. The server always returns `200` to prevent infinite retries; idempotency handles any duplicate deliveries.
- **Voided vs refunded** — Google's `voidedPurchaseNotification` covers both developer-initiated refunds and Google-support refunds for IAP. Apple handles refunds via its own `REFUND` notification.
- **`ONE_TIME_PRODUCT_CANCELED` (type 2)** fires when a pending purchase fails or times out before payment is collected — no money was taken, so no credits were ever added. The server logs it and does nothing.

---

## Notification and Sync Flow

```mermaid
sequenceDiagram
    participant G as Google Play
    participant P as Cloud Pub/Sub
    participant S as Your Server
    participant A as Android App

    Note over G,P: IAP purchase event
    G->>P: Publishes OneTimeProductNotification
    P->>S: POST /api/user/webhook/google-play
    Note right of P: message.data is base64-encoded JSON
    S->>S: Check UserPurchasesTable for token
    alt Token not found — fallback path
        S->>S: Add credits + write token atomically
    else Token found — verify-purchase already ran
        S->>S: Skip — already credited
    end
    S-->>P: 200 OK

    Note over G,P: Void / refund event
    G->>P: Publishes VoidedPurchaseNotification
    P->>S: POST /api/user/webhook/google-play
    S->>S: Deduct credits from user
    S-->>P: 200 OK

    Note over A: User opens the app
    A->>S: GET /api/user/profile
    S-->>A: credits balance updated
    A->>A: Refresh credit display
```
