---
layout: post
title: "in_app_purchase Pending Transactions - Recover Flutter Purchases Without Duplicate Grants"
description: "Handle Flutter in_app_purchase pending and restored transactions safely with early stream subscription, idempotent delivery, and correct completePurchase timing."
date: 2026-09-23
tags: [in_app_purchase, payments, migration, testing, iOS, Android]
comments: true
share: true
---

![Flutter in_app_purchase transaction flow with pending transaction recovery](/assets/images/in-app-purchase-pending-transaction-recovery.png)

If a Flutter app sells subscriptions or non-consumables, the safest fix for a repeated purchase or `duplicate transaction` error is not to call `buyNonConsumable` again. Subscribe to `purchaseStream` during startup, verify and deliver each transaction idempotently, then call `completePurchase` only after delivery. This applies to `in_app_purchase` apps that keep entitlements on a server; a consumable-only app with no account or restore path needs a different product design.

## Why the error returns after a restart

`in_app_purchase` is event-driven. A purchase result is not returned from `buyNonConsumable`; it arrives on `purchaseStream`. The official package documentation recommends subscribing as early as possible because updates can represent a previous app session. If the app verifies a receipt but never completes the store transaction, the store can send it again on the next launch.

| State | App action | Do not do this |
|---|---|---|
| `pending` | Show a waiting state and keep listening | Complete or grant entitlement yet |
| `purchased` | Verify, grant once, then complete | Grant before verification |
| `restored` | Verify and reconcile entitlement, then complete if pending | Treat every restore event as new revenue |
| `error` | Log the code and show a retry path | Mark the user as entitled |

On Android, the current package documentation warns that failing to complete within three days can result in a refund. On iOS and macOS, an unfinished transaction stays in the queue, is repeatedly delivered, and can make a later purchase of the same product fail with a pending-duplicate error. These are store behaviors, not a reliable signal that the user should be charged again.

## Subscribe once and make delivery idempotent

The listener must exist before the first screen starts a purchase. Keep one subscription for the app lifetime; multiple listeners receive the same events and can grant the same entitlement twice.

```dart
class PurchaseCoordinator {
  PurchaseCoordinator({required this.purchase}) {
    _subscription = purchase.purchaseStream.listen(
      _onPurchases,
      onError: (Object error, StackTrace stack) {
        logPurchaseError(error, stack);
      },
    );
  }

  final InAppPurchase purchase;
  late final StreamSubscription<List<PurchaseDetails>> _subscription;

  Future<void> _onPurchases(List<PurchaseDetails> purchases) async {
    for (final details in purchases) {
      try {
        if (details.status == PurchaseStatus.pending) {
          showPurchasePending();
          continue;
        }

        if (details.status == PurchaseStatus.error) {
          showPurchaseError(details.error);
          continue;
        }

        if (details.status == PurchaseStatus.purchased ||
            details.status == PurchaseStatus.restored) {
          final receiptKey = details.purchaseID ??
              details.verificationData.serverVerificationData;

          // The server must make this operation idempotent by receiptKey.
          final valid = await verifyAndGrantOnce(
            receiptKey: receiptKey,
            verificationData: details.verificationData,
            productId: details.productID,
          );
          if (!valid) continue;
        }

        if (details.pendingCompletePurchase) {
          await purchase.completePurchase(details);
        }
      } catch (error, stack) {
        // Leave the transaction recoverable on the next stream delivery.
        logPurchaseError(error, stack);
      }
    }
  }

  Future<void> dispose() => _subscription.cancel();
}
```

`verifyAndGrantOnce` is intentionally a boundary, not a local boolean. A restore event can arrive more than once, and the process can die after the server grants access but before `completePurchase` returns. Store the receipt or transaction identifier with a unique constraint, make repeated grants a no-op, and return success for an already-granted valid transaction. Only that successful result should unlock the local UI and permit completion.

## Restore is not a consumable backup

`restorePurchases()` emits non-consumable and subscription purchases through the same stream. It does not restore consumed products. If a user changes devices and consumables matter, keep the consumable balance or purchase history on your own authenticated server. Calling restore repeatedly is not a substitute for that record.

Use this release checklist before shipping a purchase-flow change:

- Start the stream before rendering the paywall, and ensure there is exactly one listener.
- Test a process kill after server delivery but before `completePurchase`.
- Test `pending` without granting access, then a later `purchased` or `restored` event.
- Test reinstall or a second device for non-consumables and subscriptions.
- Confirm the server accepts the same receipt twice without double crediting.
- Log product ID, status, receipt key hash, verification result, and completion error without logging the raw receipt.

The short rule is: `pending` waits, `purchased` and `restored` verify, delivery is idempotent, and `completePurchase` closes the store transaction only after the entitlement is safe. The package's [current usage documentation](https://pub.dev/packages/in_app_purchase) and its [purchase-stream API comments](https://flutter.googlesource.com/mirrors/packages/+/refs/tags/go_router-v9.0.1/packages/in_app_purchase/in_app_purchase/lib/in_app_purchase.dart) both support this order.
