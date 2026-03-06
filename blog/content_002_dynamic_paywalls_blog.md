---
id: content_002
type: blog
status: ready
topic: Building Dynamic Paywalls with RevenueCat Offerings
audience: mobile developers (iOS, Android, Flutter, React Native)
tags: [paywalls, offerings, revenuecat, monetization, subscriptions]
created: 2026-03-05T21:42:15Z
generated: 2026-03-05T21:52:00Z
pipeline_version: v1
word_count: ~1300
read_time: 8 min
seo_title: "Building Dynamic Paywalls with RevenueCat Offerings | Complete Guide"
seo_description: "Learn how to build remotely-configurable paywalls with RevenueCat Offerings. Code examples for iOS (SwiftUI), Android (Jetpack Compose), Flutter, and React Native. No more app updates for paywall changes."
seo_keywords: [revenuecat paywalls, revenuecat offerings, dynamic paywall ios, revenuecat flutter paywall, mobile subscription monetization, presentPaywallIfNeeded, PaywallDialog android]

social_hooks:
  tweet: |
    Hard-coded paywalls are shipping you toward irrelevance.

    Every price change = an app update = days of review.

    RevenueCat Offerings + Paywalls let you change everything — copy, layout, pricing — without touching your code.

    Here's how to set it up: 🧵
  bluesky: |
    Hard-coded paywalls are an antipattern.

    RevenueCat Offerings + Paywalls = change pricing, copy, and UI without an app update.

    Full setup guide for iOS, Android, Flutter, RN 👇
    #mobiledev #subscriptions #revenuecat
  linkedin: |
    Most mobile apps still ship paywall changes in app updates.
    That's a 2–7 day feedback loop every time you want to test a headline, price point, or CTA.
    RevenueCat solves this with Offerings + Paywalls — a remote-config system for your entire monetization layer.
    Key insight: your app code calls one line — offerings.current — and RevenueCat handles the rest.

next_in_queue: content_003 (Handling RevenueCat Webhooks in Node.js)
---

# Building Dynamic Paywalls with RevenueCat Offerings

> **Audience:** Mobile developers — iOS, Android, Flutter, React Native  
> **Level:** Intermediate  
> **Time to read:** ~8 min  
> **Tags:** `paywalls` `offerings` `revenuecat` `monetization` `subscriptions`

---

## The Old Way Is Costing You Conversions

Hard-coded paywalls are an antipattern. Every time you need to change a price, tweak copy, or run an experiment, you push an app update — and wait days for review. By the time your change ships, your A/B test window has closed.

RevenueCat's **Offerings + Paywalls** system flips this model. Define your products and UI in the RevenueCat dashboard. Control everything remotely. No app updates required.

This post covers exactly how to set it up and how to use it strategically.

---

## Core Concepts: Offerings, Packages, Paywalls

Before writing a single line of code, you need to understand the three-layer model:

```
Project
└── Offering  (a named "offer" you show users — e.g., "default", "summer_sale")
    ├── Package  (a product + platform wrapper — monthly, annual, lifetime)
    │   └── Product  (the actual StoreKit / Play Billing product ID)
    └── Paywall  (the UI template bound to this offering — configured in dashboard)
```

**Key insight:** Your app code never hardcodes product IDs or paywall layouts. It asks RevenueCat: *"What should I show this user right now?"* — and RevenueCat returns the current Offering with its attached Paywall.

This is the foundation for everything that follows.

---

## Step 1: Configure Your Offering in the Dashboard

1. Go to **Project → Product Catalog → Offerings**
2. Click **+ New Offering** — give it an identifier like `default` or `premium_annual_promo`
3. Add **Packages** — each Package maps to a platform product:
   - iOS: your App Store Connect product ID
   - Android: your Google Play product ID
   - RevenueCat normalises them under a single package identifier

**Pro tip:** Always create a `default` offering. RevenueCat's SDK will fall back to it when no targeted offering is active for a user.

---

## Step 2: Attach a Paywall

With your Offering created:

1. Click into the Offering → **Create Paywall**
2. Choose a template (RevenueCat provides 5+ pre-built templates: classic, condensed, feature-list, etc.)
3. Configure colours, copy, CTA text, and feature highlights — **all without code**
4. Hit **Publish**

Your paywall is now live. Any user your app fetches this offering for will see the new UI immediately — no app update.

**Minimum SDK versions required for Paywalls:**

| Platform      | Min SDK Version |
|---------------|----------------|
| iOS           | 5.27.1+        |
| Android       | 8.19.2+        |
| React Native  | 8.11.3+        |
| Flutter       | 8.10.1+        |
| Capacitor     | 10.3.3+        |

---

## Step 3: Fetch and Display the Offering in Code

### iOS (SwiftUI)

The simplest integration — let RevenueCatUI handle everything:

```swift
import SwiftUI
import RevenueCat
import RevenueCatUI

struct App: View {
    var body: some View {
        ContentView()
            // Option A: Show paywall automatically if entitlement is missing
            .presentPaywallIfNeeded(
                requiredEntitlementIdentifier: "pro",
                purchaseCompleted: { customerInfo in
                    print("Purchase completed: \(customerInfo.entitlements)")
                },
                restoreCompleted: { customerInfo in
                    print("Restore completed")
                }
            )
    }
}
```

Or for full manual control:

```swift
import SwiftUI
import RevenueCat
import RevenueCatUI

struct UpgradeScreen: View {
    @State private var showPaywall = false

    var body: some View {
        Button("Upgrade to Pro") {
            showPaywall = true
        }
        .sheet(isPresented: $showPaywall) {
            // RevenueCatUI renders the dashboard-configured paywall
            PaywallView()
        }
    }
}
```

### Android (Jetpack Compose)

```kotlin
@OptIn(ExperimentalPreviewRevenueCatUIPurchasesAPI::class)
@Composable
fun LockedFeatureScreen() {
    YourContent()
    PaywallDialog(
        PaywallDialogOptions.Builder()
            .setRequiredEntitlementIdentifier("pro")
            .setListener(object : PaywallListener {
                override fun onPurchaseCompleted(
                    customerInfo: CustomerInfo,
                    storeTransaction: StoreTransaction
                ) {
                    // Unlock the feature
                }
                override fun onRestoreCompleted(customerInfo: CustomerInfo) {}
            })
            .build()
    )
}
```

### Flutter

```dart
import 'package:purchases_flutter/purchases_flutter.dart';
import 'package:purchases_ui_flutter/purchases_ui_flutter.dart';

Future<void> showPaywall(BuildContext context) async {
  // Fetch current offering
  final offerings = await Purchases.getOfferings();
  final current = offerings.current;

  if (current != null) {
    // Display the dashboard-configured paywall for this offering
    await RevenueCatUI.presentPaywall(offering: current);
  }
}
```

### React Native

```typescript
import Purchases from 'react-native-purchases';
import RevenueCatUI from 'react-native-purchases-ui';

async function showPaywall() {
  const offerings = await Purchases.getOfferings();
  if (offerings.current !== null) {
    await RevenueCatUI.presentPaywall({
      offering: offerings.current,
    });
  }
}
```

---

## Step 4: Target Different Offerings to Different Users

Here's where dynamic paywalls become a competitive advantage. You can serve **different offerings** to different user segments — by country, by platform, by cohort — all from the dashboard.

Use cases:
- 🌍 **Regional pricing** — `annual_us`, `annual_br`, `annual_in` with different price points
- 🎯 **Retention offers** — show `win_back_50off` to churned users detected via webhook
- 🔬 **Experiments** — assign `offering_a` vs `offering_b` via RevenueCat Experiments

Your app code stays the same. Just fetch `offerings.current` — RevenueCat handles the targeting logic server-side.

```swift
// This one line fetches whatever offering RevenueCat has decided
// this user should see right now — could be any of your offerings
let offerings = try await Purchases.shared.offerings()
let currentOffering = offerings.current
```

---

## Step 5: Handle the Paywall Result

After a user interacts with the paywall, check entitlements — don't trust the purchase response alone:

```swift
// iOS — after paywall dismissal
let customerInfo = try await Purchases.shared.customerInfo()

if customerInfo.entitlements["pro"]?.isActive == true {
    unlockProFeatures()
} else {
    showFreeContent()
}
```

```kotlin
// Android — after PaywallDialog interaction
override fun onPurchaseCompleted(
    customerInfo: CustomerInfo,
    storeTransaction: StoreTransaction
) {
    val proEntitlement = customerInfo.entitlements["pro"]
    if (proEntitlement?.isActive == true) {
        unlockProFeatures()
    }
}
```

**Critical rule:** Always gate features on entitlements, not on purchase events. Entitlements handle renewals, restores, and cross-platform access automatically.

---

## The Power Pattern: Offerings + Experiments

The real ROI of this system is pairing Offerings with RevenueCat Experiments:

1. Create two Offerings: `paywall_v1` (monthly focus) and `paywall_v2` (annual focus)
2. Create an Experiment in the RevenueCat dashboard
3. Set traffic split: 50% → `paywall_v1`, 50% → `paywall_v2`
4. Run for 2–4 weeks
5. Compare conversion rates in the RevenueCat analytics dashboard

Zero app changes required. You get statistically-valid paywall data without shipping a single line of code.

---

## Common Mistakes to Avoid

| ❌ Mistake | ✅ Fix |
|---|---|
| Hardcoding product IDs in app code | Always fetch from `offerings.current` |
| Not handling `offerings.current == nil` | Add graceful fallback or degraded free UI |
| Checking purchase events instead of entitlements | Always check `customerInfo.entitlements["pro"]?.isActive` |
| Using Paywalls with outdated SDK | Check minimum SDK version table — old SDKs get beta Paywalls only |
| Not installing `RevenueCatUI` package | Required separately from the core SDK |

---

## Installation Checklist

```bash
# iOS (Swift Package Manager)
# Add: https://github.com/RevenueCat/purchases-ios
# Add both packages: RevenueCat + RevenueCatUI

# Android (Gradle)
implementation("com.revenuecat.purchases:purchases:8.x.x")
implementation("com.revenuecat.purchases:purchases-ui:8.x.x")

# Flutter
flutter pub add purchases_flutter
flutter pub add purchases_ui_flutter

# React Native
npm install react-native-purchases
npm install react-native-purchases-ui
```

---

## Key Takeaways

1. **Offerings decouple your products from your code** — change pricing, packaging, or descriptions without an app update
2. **Paywalls decouple your UI from your code** — iterate on paywall copy and design in the dashboard
3. **The pattern is always the same:** fetch `offerings.current` → display it → check entitlements
4. **Targeting + Experiments** is where the real conversion wins live — use them
5. **Install `RevenueCatUI`** separately from the core SDK — easy to miss, breaks paywall rendering

---

## What's Next

- **Part 3:** Handling RevenueCat Webhooks in Node.js — process subscription events server-side
- **Part 4:** RevenueCat vs Native StoreKit 2 and Play Billing — when does the abstraction cost you?
- **Experiments deep dive:** A/B Testing Paywalls with RevenueCat Experiments

---

*Generated by ELIANCE content pipeline | RevenueCat knowledge base v2026-03-05 | content_002*
