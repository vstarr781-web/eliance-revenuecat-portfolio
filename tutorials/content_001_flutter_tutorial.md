# Getting Started with RevenueCat in Flutter: In-App Subscriptions in 30 Minutes

> **Audience:** Flutter developers | **Read time:** 8 minutes | **Tags:** `#flutter`, `#revenuecat`, `#subscriptions`, `#mobile`

---

## Why RevenueCat?

Building a subscription-based Flutter app? Managing in-app purchases manually across iOS and Android is a nightmare — different APIs, receipt validation, server-side logic. RevenueCat wraps all of it in a single SDK so you can ship subscriptions in under an hour.

## Prerequisites

- Flutter 3.x project set up
- A RevenueCat account (free at revenuecat.com)
- A RevenueCat project with at least one product/entitlement configured
- iOS Simulator or Android Emulator for testing

---

## Step-by-Step Tutorial

### Step 1: Install the RevenueCat Flutter SDK

Add `purchases_flutter` to your `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  purchases_flutter: ^7.0.0
```

Then run:

```bash
flutter pub get
```

---

### Step 2: Configure the SDK on App Launch

In your `main.dart`, configure RevenueCat with your API key before `runApp()`.

```dart
import 'dart:io';
import 'package:purchases_flutter/purchases_flutter.dart';

Future<void> initRevenueCat() async {
  await Purchases.setLogLevel(LogLevel.debug); // helpful during development
  PurchasesConfiguration config;

  if (Platform.isAndroid) {
    config = PurchasesConfiguration('<google_api_key>');
  } else {
    config = PurchasesConfiguration('<apple_api_key>');
  }

  await Purchases.configure(config);
}
```

Call `initRevenueCat()` inside `main()` before `runApp()`:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initRevenueCat();
  runApp(MyApp());
}
```

> **Note:** Use only your **public API key** here. Find it in the RevenueCat dashboard under API Keys.

---

### Step 3: Fetch Offerings (Your Paywall Products)

Offerings are paywall configurations managed from the RevenueCat dashboard — change pricing without shipping a new app version.

```dart
Future<void> loadOfferings() async {
  try {
    Offerings offerings = await Purchases.getOfferings();
    if (offerings.current != null) {
      for (var pkg in offerings.current!.availablePackages) {
        print('${pkg.packageType}: ${pkg.storeProduct.priceString}');
      }
    }
  } catch (e) {
    print('Error fetching offerings: $e');
  }
}
```

Each `Package` in `availablePackages` maps to a product you configured in the dashboard (monthly, annual, lifetime, etc.).

---

### Step 4: Trigger a Purchase

When the user taps 'Subscribe', call `purchasePackage()`. RevenueCat handles the store transaction and receipt validation server-side — automatically.

```dart
Future<void> subscribe(Package package) async {
  try {
    CustomerInfo customerInfo = await Purchases.purchasePackage(package);
    if (customerInfo.entitlements.active.containsKey('pro')) {
      unlockProFeatures(); // ✅ User is now subscribed
    }
  } on PurchasesErrorCode catch (e) {
    if (e != PurchasesErrorCode.purchaseCancelledError) {
      print('Purchase error: $e');
    }
  }
}
```

> **Tip:** `PurchasesErrorCode.purchaseCancelledError` fires when the user cancels — that's normal, don't treat it as an error.

---

### Step 5: Check Entitlement Status on App Launch

Always check entitlement status at startup — this handles subscription renewals, restores, and cross-device access.

```dart
Future<void> checkSubscriptionStatus() async {
  try {
    CustomerInfo customerInfo = await Purchases.getCustomerInfo();
    if (customerInfo.entitlements.active.containsKey('pro')) {
      showPremiumUI(); // User is subscribed
    } else {
      showPaywall();   // User is on free tier
    }
  } catch (e) {
    print('Error checking subscription status: $e');
  }
}
```

`customerInfo.entitlements.active` is your **single source of truth** for access — it's always up to date with the store.

---

### Step 6: Restore Purchases

Apple requires a 'Restore Purchases' button in your app. One call handles it:

```dart
Future<void> restorePurchases() async {
  try {
    CustomerInfo customerInfo = await Purchases.restorePurchases();
    if (customerInfo.entitlements.active.containsKey('pro')) {
      unlockProFeatures(); // ✅ Restored successfully
    }
  } catch (e) {
    print('Restore failed: $e');
  }
}
```

---

## Testing Your Integration

**Use RevenueCat's Test Store** during development — no Apple or Google credentials needed. Your project ships with a Test Store automatically. Initialize the SDK normally; sandbox purchases will appear in your RevenueCat dashboard under Sandbox data.

For production testing:
- **iOS:** Use Apple Sandbox testers (App Store Connect → Users and Access → Sandbox)
- **Android:** Use License Testers (Google Play Console → Setup → License Testing)

---

## Key Takeaways

- ✅ One SDK handles iOS, Android, and more — no store-specific code needed
- ✅ Offerings let you change pricing remotely without shipping a new app version
- ✅ Entitlements decouple your feature logic from product IDs
- ✅ `CustomerInfo.entitlements.active` is your single source of truth for access
- ✅ RevenueCat handles receipt validation server-side — no custom backend needed

---

## 📣 Promotional Posts

**Thread hook (Twitter/X):**
```
🐦 In-app subscriptions in Flutter used to be painful.

Here's how to implement them with RevenueCat in 6 steps (30 min total) 🧵

#Flutter #RevenueCat #MobileDev
```

**Standalone tip tweet:**
```
💡 Flutter tip: Never hardcode subscription product IDs in your app.

Use RevenueCat Offerings — change pricing remotely from your dashboard.
No app update. No App Store review. Just ship.

#Flutter #Subscriptions
```

---

## SEO Metadata

**Meta description:** Learn how to add in-app subscriptions to your Flutter app using RevenueCat. Step-by-step tutorial with full Dart code: install SDK, fetch offerings, trigger purchases, and check entitlement status.

**Keywords:** flutter in-app purchases, revenuecat flutter, flutter subscriptions, purchases_flutter, flutter monetization, dart subscription tutorial

---

*Generated by ELIANCE Content Pipeline — 2026-03-05 UTC*
