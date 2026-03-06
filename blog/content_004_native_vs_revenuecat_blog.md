---
title: "Cross-Platform Subscription Management: RevenueCat vs Native APIs"
type: blog
topic: Cross-Platform Subscription Management: RevenueCat vs Native APIs
audience: mobile developers
tags: [cross-platform, storekit, google-play-billing, revenuecat, monetization, comparison]
content_id: content_004
word_count_target: 1400
---

# Cross-Platform Subscription Management: RevenueCat vs Native APIs

Every mobile app that wants recurring revenue faces the same decision: implement subscriptions natively (StoreKit 2 on iOS, Google Play Billing Library on Android) or reach for an abstraction layer like RevenueCat. This post breaks down the trade-offs with actual code, real numbers, and a clear decision framework.

---

## The Hidden Cost of "Just Using the Native API"

On the surface, native APIs are free. No third-party dependency, full control, no revenue share. That math looks obvious — until you measure the engineering time.

A complete, production-ready subscription implementation on a **single platform** requires:

- Product/pricing fetch and display logic
- Purchase flow (initiate, handle pending states, handle interrupted purchases)
- Server-side receipt validation (required — client-side is not secure)
- Entitlement checking across app lifecycle
- Subscription renewal event handling
- Family Sharing edge cases (iOS)
- Grace period handling (billing retry)
- Refund detection
- Restore purchases flow

On iOS with **StoreKit 2** (Swift), a minimal but correct implementation runs ~200–300 lines of code. On **Android with Google Play Billing Library 7**, add another ~250–350. Then there's the server — you need separate endpoints for Apple Server Notifications v2 and Google Real-Time Developer Notifications, with different schemas, different auth mechanisms, and different retry behaviors.

That's a significant surface area to own, test, and maintain — across two evolving APIs that change every WWDC and Google I/O.

---

## Side-by-Side: RevenueCat vs Native

| Dimension | StoreKit 2 + GPBL (Native) | RevenueCat |
|-----------|---------------------------|------------|
| **SDK lines of code** | ~200–300 per platform | ~10 per platform |
| **Cross-platform consistency** | Separate codebases, separate logic | Unified entitlements across all platforms |
| **Receipt validation** | Build it yourself (server-side) | Automatic, handles all edge cases |
| **Analytics** | None — pipe events manually | Built-in MRR, ARR, churn, LTV |
| **A/B testing paywalls** | Manual implementation | Experiments — configured in dashboard |
| **Server event schema** | Apple + Google (different formats) | Single normalized webhook schema |
| **API change maintenance** | Your responsibility | RevenueCat absorbs changes |
| **Cost** | Free (engineering time) | Free to $2.5K MRR; 1% above that |

---

## Code Comparison: Fetching and Presenting Products

### Native — StoreKit 2 (Swift)

```swift
import StoreKit

class SubscriptionManager: ObservableObject {
    var products: [Product] = []
    var purchasedProductIDs = Set<String>()
    
    private let productIdentifiers = ["com.myapp.monthly", "com.myapp.annual"]
    
    func loadProducts() async {
        do {
            products = try await Product.products(for: productIdentifiers)
        } catch {
            print("Failed to load products: \(error)")
        }
    }
    
    func purchase(_ product: Product) async throws {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            switch verification {
            case .verified(let transaction):
                await transaction.finish()
                await updatePurchasedProducts()
            case .unverified:
                throw SubscriptionError.failedVerification
            }
        case .userCancelled:
            break
        case .pending:
            // Deferred payments: ask-to-buy, SCA
            break
        @unknown default:
            break
        }
    }
    // Still need: server-side validation, Family Sharing, grace period handling
}
```

### Native — Google Play Billing Library (Kotlin)

```kotlin
class BillingManager(private val activity: Activity) : PurchasesUpdatedListener {
    private lateinit var billingClient: BillingClient
    
    fun init() {
        billingClient = BillingClient.newBuilder(activity)
            .setListener(this)
            .enablePendingPurchases()
            .build()
        
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    loadProducts()
                }
            }
            override fun onBillingServiceDisconnected() {
                // Must implement retry logic yourself
            }
        })
    }
    
    override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
        // Handle result — then validate server-side yourself
    }
}
// Lines so far: ~100+ per platform. No analytics, no webhooks, no A/B testing.
```

### RevenueCat — Both Platforms, Same Logic

**iOS (Swift):**
```swift
import RevenueCat

// Configure once at app launch
Purchases.configure(withAPIKey: "YOUR_PUBLIC_API_KEY", appUserID: currentUserID)

// Fetch remote-configured offerings (paywalls)
let offerings = try await Purchases.shared.offerings()
let currentOffering = offerings.current

// Purchase a package
let result = try await Purchases.shared.purchase(package: selectedPackage)

// Check entitlement — identical on iOS and Android
let customerInfo = try await Purchases.shared.customerInfo()
if customerInfo.entitlements["premium"]?.isActive == true {
    // Unlock premium features
}
```

**Android (Kotlin):**
```kotlin
// Application.onCreate()
Purchases.configure(PurchasesConfiguration.Builder(this, "YOUR_PUBLIC_API_KEY").build())

// Fetch offerings
Purchases.sharedInstance.getOfferingsWith { offerings ->
    offerings.current?.availablePackages?.forEach { pkg -> /* display */ }
}

// Purchase + check entitlement
Purchases.sharedInstance.purchaseWith(
    PurchaseParams.Builder(activity, selectedPackage).build(),
    onError = { error, _ -> /* handle */ },
    onSuccess = { _, customerInfo ->
        if (customerInfo.entitlements["premium"]?.isActive == true) {
            // Unlock — same check as iOS, no platform branching
        }
    }
)
```

The entitlement check is **identical** across platforms. No `if (Platform.isIOS)` branching in your feature-gating logic.

---

## Analytics: The Invisible Advantage

Native APIs give you purchase events. That's it. To get MRR, churn rate, LTV, or trial conversion you build a data pipeline: capture events → push to analytics → write SQL → maintain dashboards.

RevenueCat provides out of the box:

- **MRR and ARR** — real-time
- **Churn and retention cohort charts** — month-by-month
- **Trial conversion rate** — segmented by offering and platform
- **Customer Timeline** — per-user purchase history for debugging
- **Integrations** — normalized events pushed to Amplitude, Mixpanel, Segment, Braze with a single toggle

For a 2–5 person team, this is months of data engineering work eliminated.

---

## Experiments: A/B Testing Paywalls Without a Release

Changing your paywall price or trial length natively means a code change → review cycle → release → wait for adoption. Testing two variants means building assignment logic, tracking exposure events, calculating significance yourself.

With RevenueCat Experiments:

1. Define Variant A and Variant B offerings in the dashboard
2. Set traffic split (e.g. 50/50)
3. SDK automatically routes users to their assigned variant
4. Dashboard shows conversion, revenue impact, and statistical significance

No code change. No release. Revenue impact automatically calculated.

---

## Webhook Events: One Schema for All Platforms

Apple Server Notifications v2 and Google RTDN use completely different schemas:

**Apple (JWT payload):**
```json
{ "signedPayload": "eyJhbGci..." }
```

**Google (Pub/Sub):**
```json
{ "message": { "data": "base64_encoded" }, "subscription": "projects/..." }
```

**RevenueCat normalized:**
```json
{
  "event": {
    "type": "RENEWAL",
    "app_user_id": "user_12345",
    "product_id": "monthly_premium",
    "period_type": "NORMAL",
    "purchased_at_ms": 1709654400000,
    "expiration_at_ms": 1712332800000,
    "store": "APP_STORE",
    "currency": "USD",
    "price": 9.99
  }
}
```

One schema. One parser. One test suite. iOS renewals, Android renewals, and web purchases handled by a single switch statement.

---

## When to Use Native APIs

Native is the right call when:

- **Single platform, single subscription tier** — minimal complexity, one-time setup cost
- **Hard vendor lock-in policy** — no third-party SDKs (enterprise or regulated industries)
- **Full data ownership required** — no external processing of transaction data
- **Very early stage** — less than $500 MRR; the 1% fee is premature to optimize

## When RevenueCat Wins

RevenueCat is worth it when:

- **Multi-platform** — any combination of iOS, Android, web, or cross-platform frameworks
- **Multiple products** — subscriptions + one-time purchases + consumables
- **Analytics matters now** — MRR/churn without a data team
- **Paywall testing is a growth lever** — Experiments eliminates release-cycle friction
- **Small team** — billing is not your core competency; don't make it one

---

## The Real Math

RevenueCat is free up to **$2,500 MRR**. Above that: 1% of tracked revenue (Starter plan).

At $10,000 MRR:
- RevenueCat cost: **$100/month**
- Engineering cost to maintain equivalent native implementation: conservatively 2 hrs/month (Apple/Google API changes, edge case debugging, notification monitoring). At any reasonable developer rate, RevenueCat pays for itself.

At $100,000 MRR, some teams move to native or negotiate Enterprise pricing. But for 90% of indie developers and small studios, the ROI is unambiguous.

---

## Decision Framework

```
Multi-platform app?          YES → Use RevenueCat
         ↓ NO
Multiple product types?      YES → Use RevenueCat
         ↓ NO
Need analytics/webhooks?     YES → Use RevenueCat
         ↓ NO
Hard no-SDK vendor policy?   YES → Native
         ↓ NO
                             → RevenueCat (revisit at $100K+ MRR)
```

---

## Bottom Line

Native APIs are powerful. RevenueCat is built on top of them — it calls StoreKit 2 and GPBL under the hood. What you're paying for is the abstraction layer, the server infrastructure, the analytics, and the maintenance contract.

For most apps with serious subscription revenue goals, the time-to-revenue advantage far outweighs the 1% fee. The question isn't "can you build it natively?" — of course you can. The question is: **what else could your team build with those weeks?**

---

## Social Hooks

### Twitter/X Hook
```
You can implement subscriptions natively on iOS + Android.

Here's what that actually costs:
→ ~300 LOC per platform
→ Server-side receipt validation x2 (different schemas)
→ Separate Apple + Google webhook parsers
→ Zero built-in analytics

RevenueCat does all of this. Costs 1% above $2.5K MRR.

The math writes itself 🧵
```

### Bluesky Hook
```
RevenueCat vs native StoreKit 2 + Google Play Billing.

8 dimensions. Actual Swift + Kotlin code. Clear decision framework.

TL;DR: unless you're single-platform with a no-SDK policy, RevenueCat wins on engineering hours every time.
```

### LinkedIn Hook
```
"Just use the native APIs" sounds right and costs teams weeks.

We compared RevenueCat vs StoreKit 2 + Google Play Billing across 8 dimensions: setup complexity, receipt validation, analytics, webhook schemas, A/B testing, and cost.

The decision framework is simple. The math is clear.

Link in comments 👇
```
