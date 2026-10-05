---
layout: post
title: "in_app_review Quotas - Build a Flutter Review Flow with a Reliable Store Fallback"
description: "Use Flutter's in_app_review without treating isAvailable as a guarantee: separate quota-limited prompts from an explicit store link, persist your own cooldown, and test the real release paths."
date: 2026-10-05
tags: [in_app_review, Flutter, Android, iOS, testing, monetization]
comments: true
share: true
---
![Flutter in_app_review prompt and store listing fallback flow](https://images.unsplash.com/photo-1512428559087-560fa5ceab42?auto=format&fit=crop&w=1600&q=80)

The safe choice is to call `requestReview()` after a successful user milestone, while keeping `openStoreListing()` for a visible “Write a review” action. This is for Flutter apps that need both a native review prompt and a dependable store destination. It does not make the native prompt appear on demand: Google Play and App Store policies can suppress it even when the plugin reports that the device supports reviews.

## Why `isAvailable()` is not a display result

`in_app_review` exposes three useful operations, but they have different contracts:

| Operation | Use it for | What can still happen |
| --- | --- | --- |
| `isAvailable()` | Capability check before a prompt attempt | `true` does not mean a dialog will be shown |
| `requestReview()` | A contextual, low-frequency prompt after success | Android quota or store policy can suppress it |
| `openStoreListing()` | A persistent settings/help action | Leaves the app and needs a valid store listing |

Google documents the review quota as time-bound and changeable. Apple also decides whether to show the request and limits prompts to at most three times in a 365-day period under its documented conditions. Treat the native request as best effort, not as a button result that the UI must wait for.

## Keep the prompt path separate from the button path

The following service deliberately does not expose `requestReview()` directly to a button. `shouldAskAfterMilestone` represents an app-owned rule such as “the user completed a purchase” or “finished three lessons”; the value should be persisted so a process restart does not reset the cooldown.

```dart
import 'package:in_app_review/in_app_review.dart';

class ReviewFlow {
  ReviewFlow({InAppReview? api}) : _api = api ?? InAppReview.instance;

  final InAppReview _api;

  Future<void> afterSuccessfulMilestone({
    required bool shouldAskAfterMilestone,
  }) async {
    if (!shouldAskAfterMilestone) return;

    try {
      if (await _api.isAvailable()) {
        // Best effort: the platform may suppress the dialog because of quota.
        await _api.requestReview();
      }
    } catch (_) {
      // Review prompts must never block the completed product action.
    }
  }

  Future<void> openReviewSettings({String? appStoreId}) {
    return _api.openStoreListing(appStoreId: appStoreId);
  }
}
```

The `catch` is intentional. A review request is feedback infrastructure, not part of checkout, upload, or account creation. Record an internal attempt if you need diagnostics, but do not show “review failed” as if the user’s main task failed.

## A release checklist that catches the misleading failures

| Check | Expected decision |
| --- | --- |
| Android emulator or sideloaded APK | Do not use it to prove that a real review can be submitted |
| Android internal testing | Verify availability and flow; the submit button may remain disabled |
| Android production track | Validate the final package ID and store listing |
| iOS simulator | Use it for UI wiring, not App Store submission behavior |
| TestFlight | Do not expect `requestReview()` to show a real prompt |
| Settings or support screen | Use `openStoreListing()`, not `requestReview()` |

Also verify the Android device has Google Play Store and Play Services, and pass the iOS App Store ID when opening the listing. The package documentation lists both requirements and platform-specific testing limits. Never gate a successful user action on the prompt appearing; there is no reliable callback that means “the user saw and submitted a review.”

The short rule is: own the timing and cooldown, let the platform own the decision, and provide a separate store link when the user explicitly asks to review.

Sources: [in_app_review package documentation](https://pub.dev/packages/in_app_review), [Google Play In-App Reviews API](https://developer.android.com/guide/playcore/in-app-review), [Apple ratings and reviews guidance](https://developer.apple.com/app-store/ratings-and-reviews/)
