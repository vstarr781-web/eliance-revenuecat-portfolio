---
id: content_003
type: tutorial
topic: Handling RevenueCat Webhooks in Node.js
audience: backend developers
tags: [webhooks, nodejs, revenuecat, express, server-side, subscriptions]
status: ready
generated: 2026-03-05T21:58 UTC
pipeline_version: v3
techniques: [CG02_prompt_template_randomizer, CG03_langgraph_stateful_post_generator]
word_count: ~1600
seo_title: "How to Handle RevenueCat Webhooks in Node.js (Express)"
seo_description: "A complete guide to building a production-ready RevenueCat webhook receiver in Node.js with Express. Covers auth, idempotency, event routing, and error handling."
social_hook_tweet: "Your RevenueCat webhook handler is probably broken. Here's how to build one that actually works in Node.js (idempotency, auth, retry-safe) 🧵"
social_hook_bluesky: "Your RevenueCat webhook handler is probably broken. Here's how to build one that actually works in Node.js — idempotency, auth, and safe retries included."
social_hook_linkedin: "Most backend devs skip 3 critical things when building RevenueCat webhook handlers: authorization, idempotency, and async processing. Here's the complete Node.js pattern that handles all three."
next_in_queue: content_004
---

# Handling RevenueCat Webhooks in Node.js

RevenueCat can fire real-time events to your server every time a subscription changes —
a new purchase, a renewal, a cancellation, a billing failure. If your backend isn't
listening, you're flying blind on your own revenue.

This tutorial walks you through building a production-ready webhook receiver in **Node.js
with Express** — including authorization, idempotency, async processing, and event routing.

---

## Prerequisites

- Node.js 18+ and npm
- A RevenueCat account on the **Pro plan** (webhooks require Pro)
- An Express app (or we'll create one from scratch)
- A publicly reachable HTTPS URL (use [ngrok](https://ngrok.com) for local testing)

---

## Step 1 — Install Dependencies

```bash
npm init -y
npm install express body-parser
npm install --save-dev nodemon
```

Your `package.json` scripts:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

---

## Step 2 — Create the Express Server

```javascript
// server.js
const express = require('express');
const bodyParser = require('body-parser');

const app = express();
const PORT = process.env.PORT || 3000;

// Parse JSON body — required for RevenueCat webhook payloads
app.use(bodyParser.json());

// --- Webhook Route ---
app.post('/webhooks/revenuecat', handleRevenueCatWebhook);

app.listen(PORT, () => {
  console.log(`Webhook server listening on port ${PORT}`);
});
```

---

## Step 3 — Authenticate Incoming Requests

RevenueCat lets you configure a custom **Authorization header** in the dashboard. You
should verify this on every incoming request — otherwise anyone can POST fake events to
your endpoint.

Set your secret in the RevenueCat dashboard under:
**Project → Integrations → Webhooks → Authorization Header**

Then verify it server-side:

```javascript
// middleware/verifyRevenueCatAuth.js
const REVENUECAT_WEBHOOK_SECRET = process.env.REVENUECAT_WEBHOOK_SECRET;

function verifyRevenueCatAuth(req, res, next) {
  const authHeader = req.headers['authorization'];

  if (!authHeader || authHeader !== REVENUECAT_WEBHOOK_SECRET) {
    console.warn('[Webhook] Unauthorized request rejected');
    return res.status(401).json({ error: 'Unauthorized' });
  }

  next();
}

module.exports = { verifyRevenueCatAuth };
```

Apply the middleware to your route:

```javascript
const { verifyRevenueCatAuth } = require('./middleware/verifyRevenueCatAuth');

app.post('/webhooks/revenuecat', verifyRevenueCatAuth, handleRevenueCatWebhook);
```

Store your secret in `.env`:

```env
REVENUECAT_WEBHOOK_SECRET=Bearer your-secret-token-here
PORT=3000
```

---

## Step 4 — Respond Fast, Process Async

RevenueCat will **disconnect after 60 seconds** and retry if it doesn't get a `200`.
The golden rule: **respond immediately, process later**.

```javascript
async function handleRevenueCatWebhook(req, res) {
  // ✅ Respond immediately — before any processing
  res.status(200).json({ received: true });

  // Defer actual processing (don't await here)
  processWebhookAsync(req.body).catch((err) => {
    console.error('[Webhook] Processing error:', err);
  });
}
```

> **Why?** Heavy work like DB writes, external API calls, or email triggers can take
> seconds. If RevenueCat times out waiting, it retries — leading to duplicate processing.
> Responding immediately prevents this.

---

## Step 5 — Handle Duplicate Events (Idempotency)

RevenueCat guarantees **at-least-once delivery**. The same event can arrive more than once.
You **must** deduplicate by the event `id` field.

```javascript
// A simple in-memory seen-events store (replace with Redis or DB in production)
const processedEventIds = new Set();

async function processWebhookAsync(payload) {
  const { api_version, event } = payload;

  if (!event || !event.id) {
    console.warn('[Webhook] Malformed payload — missing event.id');
    return;
  }

  // --- Idempotency check ---
  if (processedEventIds.has(event.id)) {
    console.log(`[Webhook] Duplicate event skipped: ${event.id}`);
    return;
  }
  processedEventIds.add(event.id);

  // Route to the appropriate handler
  await routeEvent(event);
}
```

> **Production tip**: Replace the `Set` with a Redis `SETNX` or a database unique
> constraint on `event_id`. The `Set` works for single-process servers but won't survive
> restarts or multiple instances.

---

## Step 6 — Route Events by Type

RevenueCat sends ~15 event types. Build a clean router:

```javascript
async function routeEvent(event) {
  console.log(`[Webhook] Received event: ${event.type} | user: ${event.app_user_id}`);

  switch (event.type) {
    case 'INITIAL_PURCHASE':
      await onInitialPurchase(event);
      break;

    case 'RENEWAL':
      await onRenewal(event);
      break;

    case 'CANCELLATION':
      await onCancellation(event);
      break;

    case 'EXPIRATION':
      await onExpiration(event);
      break;

    case 'BILLING_ISSUE':
      await onBillingIssue(event);
      break;

    case 'UNCANCELLATION':
      await onUncancellation(event);
      break;

    case 'PRODUCT_CHANGE':
      await onProductChange(event);
      break;

    case 'TEST':
      console.log('[Webhook] Test event received ✅');
      break;

    default:
      console.log(`[Webhook] Unhandled event type: ${event.type}`);
  }
}
```

---

## Step 7 — Implement Key Event Handlers

Here are the handlers for the most important lifecycle events:

```javascript
// handlers/subscriptionHandlers.js

async function onInitialPurchase(event) {
  console.log(`[INITIAL_PURCHASE] New subscriber: ${event.app_user_id}`);
  console.log(`  Product: ${event.product_id} | Store: ${event.store}`);
  console.log(`  Entitlements: ${event.entitlement_ids?.join(', ')}`);

  // ✅ Best practice: fetch full subscriber state from RevenueCat REST API
  // rather than trusting only the webhook payload
  // await syncSubscriberFromAPI(event.app_user_id);

  // Example: update your DB
  // await db.users.update({ userId: event.app_user_id }, { isPro: true });

  // Example: trigger welcome email
  // await emailService.sendWelcomeEmail(event.app_user_id);
}

async function onRenewal(event) {
  console.log(`[RENEWAL] Subscription renewed: ${event.app_user_id}`);
  console.log(`  Next expiration: ${new Date(event.expiration_at_ms)}`);
  // Update subscription expiry in your DB
}

async function onCancellation(event) {
  console.log(`[CANCELLATION] Cancelled: ${event.app_user_id}`);
  console.log(`  Cancel reason: ${event.cancel_reason}`);
  // cancel_reason: UNSUBSCRIBE | BILLING_ERROR | DEVELOPER | PRICE_INCREASE | CUSTOMER_SUPPORT | UNKNOWN

  // Don't revoke access immediately — user keeps access until expiration_at_ms
  // await scheduleAccessRevocation(event.app_user_id, event.expiration_at_ms);
}

async function onExpiration(event) {
  console.log(`[EXPIRATION] Access expired: ${event.app_user_id}`);
  // NOW revoke access
  // await db.users.update({ userId: event.app_user_id }, { isPro: false });
}

async function onBillingIssue(event) {
  console.log(`[BILLING_ISSUE] Payment failed: ${event.app_user_id}`);
  // Send win-back email, don't revoke access yet
  // The subscription may recover before EXPIRATION is sent
}

async function onUncancellation(event) {
  console.log(`[UNCANCELLATION] Resubscribed: ${event.app_user_id}`);
  // User changed their mind — ensure access is still active
}

async function onProductChange(event) {
  console.log(`[PRODUCT_CHANGE] Product change: ${event.app_user_id}`);
  console.log(`  New product: ${event.product_id}`);
  // Handle upgrade/downgrade logic
}

module.exports = {
  onInitialPurchase, onRenewal, onCancellation,
  onExpiration, onBillingIssue, onUncancellation, onProductChange
};
```

---

## Step 8 — Sync from the REST API (Recommended)

RevenueCat's own recommendation: after **any** webhook, call `GET /subscribers/{app_user_id}`
to get the full, normalized subscriber state instead of parsing each event individually.

```javascript
const https = require('https');

async function syncSubscriberFromAPI(appUserId) {
  const options = {
    hostname: 'api.revenuecat.com',
    path: `/v1/subscribers/${encodeURIComponent(appUserId)}`,
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${process.env.REVENUECAT_SECRET_API_KEY}`,
      'Content-Type': 'application/json',
      'X-Platform': 'stripe' // or ios, android, etc.
    }
  };

  return new Promise((resolve, reject) => {
    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve(JSON.parse(data)));
    });
    req.on('error', reject);
    req.end();
  });
}
```

This pattern is more resilient than tracking state purely from webhook events — it
handles edge cases like out-of-order delivery or missed events.

---

## Step 9 — Test Locally with ngrok

1. Start your server: `npm run dev`
2. In another terminal: `ngrok http 3000`
3. Copy the HTTPS forwarding URL (e.g. `https://abc123.ngrok.io`)
4. In RevenueCat dashboard → **Integrations → Webhooks → Add configuration**
5. Set URL: `https://abc123.ngrok.io/webhooks/revenuecat`
6. Set Authorization header to match your `REVENUECAT_WEBHOOK_SECRET`
7. Click **Send Test** to fire a `TEST` event

Expected server output:
```
[Webhook] Received event: TEST | user: test_user
[Webhook] Test event received ✅
```

---

## Full File Structure

```
webhook-server/
├── server.js                     # Main Express app
├── middleware/
│   └── verifyRevenueCatAuth.js  # Auth verification
├── handlers/
│   └── subscriptionHandlers.js  # Event handlers
├── .env                          # Secrets (never commit)
└── package.json
```

---

## Key Takeaways

| Concern | Solution |
|---|---|
| **Security** | Verify `Authorization` header on every request |
| **Reliability** | Respond `200` immediately, process async |
| **Idempotency** | Deduplicate using event `id` (Redis in production) |
| **Access control** | Don't revoke on `CANCELLATION` — wait for `EXPIRATION` |
| **State sync** | Call `GET /subscribers` after any event for full state |
| **Retry behavior** | RevenueCat retries 5x at 5→10→20→40→80 min intervals |
| **Future-proofing** | Handle unknown event types gracefully — new types ship without notice |

---

## What's Next?

- **[content_004]** Cross-Platform Subscription Management: RevenueCat vs Native APIs
- Add a queue (BullMQ / Redis) to buffer webhook processing at scale
- Deploy to Railway, Render, or Fly.io for a production HTTPS endpoint
- Monitor event delivery in RevenueCat dashboard → Integrations → Webhooks

---

*Generated by ELIANCE content pipeline v3 | 2026-03-05*
