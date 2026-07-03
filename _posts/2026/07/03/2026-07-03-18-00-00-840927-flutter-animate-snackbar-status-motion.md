---
layout: post
title: "flutter_animate SnackBar Motion - status feedback without custom overlays"
description: "Use flutter_animate with Flutter SnackBar and inline status banners so success, error, and undo feedback feels clear without custom overlay code."
date: 2026-07-03
tags: [flutter_animate, animation, Flutter, UI, state_management]
comments: true
share: true
---

![flutter_animate SnackBar status feedback](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

This image frames the kind of compact mobile feedback surface where snackbar timing and inline status motion need to stay predictable.

The useful pattern is simple: keep `SnackBar` responsible for temporary messaging, and use `flutter_animate` only for the small status surface that appears in your screen. I do not try to rebuild Flutter's overlay system just to get a nicer success message. The animation should clarify what happened, not compete with route transitions, keyboard insets, or scaffold layout.

The mistake I made first was wrapping every saved form with a custom overlay entry. It looked polished in one demo screen. Then a bottom sheet opened, the keyboard shifted the layout, and the overlay floated in the wrong place. A normal `ScaffoldMessenger` would have handled most of that for free.

## Choose the feedback surface

Use `SnackBar` for short-lived actions and an inline animated banner for state that belongs to the page. The split matters because users read them differently.

| Situation | Better surface | Animation job |
| --- | --- | --- |
| Item deleted with undo | `SnackBar` | keep motion minimal |
| Form saved successfully | inline banner | short fade and slide |
| Background sync failed | inline banner | persistent warning state |
| Copy to clipboard | `SnackBar` | no extra animation needed |

That table is the design guardrail. If the feedback needs an undo button, `SnackBar` is usually right. If the message changes the screen's current state, animate a local widget inside the screen.

## Keep SnackBar boring

I usually show a standard `SnackBar` and spend the animation budget on the content that changed. This keeps accessibility, timing, and placement predictable.

```dart
void showDeletedSnackBar(BuildContext context, VoidCallback onUndo) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: const Text('Task deleted'),
      action: SnackBarAction(
        label: 'Undo',
        onPressed: onUndo,
      ),
      behavior: SnackBarBehavior.floating,
      duration: const Duration(seconds: 4),
    ),
  );
}
```

The important part is what is not here. No custom `OverlayEntry`, no manual timer, no second fade wrapped around the snackbar. Flutter already animates snackbars in and out. Adding another slide or scale often makes the message feel late, especially when it appears after a swipe-to-delete gesture.

## Animate the page state

For save and sync feedback, an inline banner is cleaner. The widget is part of the layout, so `flutter_animate` can make it appear without fighting the scaffold.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class SaveStatusBanner extends StatelessWidget {
  const SaveStatusBanner({
    super.key,
    required this.status,
  });

  final SaveStatus status;

  @override
  Widget build(BuildContext context) {
    if (status == SaveStatus.idle) return const SizedBox.shrink();

    final isError = status == SaveStatus.failed;

    return Container(
      key: ValueKey(status),
      margin: const EdgeInsets.fromLTRB(16, 12, 16, 0),
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: isError ? Colors.red.shade50 : Colors.green.shade50,
        borderRadius: BorderRadius.circular(10),
        border: Border.all(
          color: isError ? Colors.red.shade200 : Colors.green.shade200,
        ),
      ),
      child: Row(
        children: [
          Icon(
            isError ? Icons.error_outline : Icons.check_circle_outline,
            color: isError ? Colors.red.shade700 : Colors.green.shade700,
          ),
          const SizedBox(width: 10),
          Expanded(
            child: Text(isError ? 'Save failed. Try again.' : 'Saved'),
          ),
        ],
      ),
    )
        .animate()
        .fadeIn(duration: 140.ms)
        .slideY(begin: -0.12, end: 0, duration: 180.ms, curve: Curves.easeOutCubic);
  }
}

enum SaveStatus { idle, saving, saved, failed }
```

`ValueKey(status)` is the small detail that saves the pattern. Without it, a failed state replacing a saved state can reuse the same subtree and skip the entrance. With the key, the banner replays only when the status identity changes.

## Do not animate every state

The `saving` state is where I usually avoid entrance motion. A spinner or disabled button is enough. Animating a banner for `saving`, then replacing it 300 ms later with `saved`, creates two pieces of feedback for one action. It feels busy even though each animation is short.

This is the split I use in production screens:

```dart
Widget buildSaveFeedback(SaveStatus status) {
  return switch (status) {
    SaveStatus.idle => const SizedBox.shrink(),
    SaveStatus.saving => const LinearProgressIndicator(minHeight: 2),
    SaveStatus.saved || SaveStatus.failed => SaveStatusBanner(status: status),
  };
}
```

The progress indicator handles waiting. The animated banner handles the result. Those are different jobs.

## Common traps

- Animating the `SnackBar` content itself can look clipped because the snackbar has its own route-like entrance.
- Reusing the same key for success and error states can prevent the animation from replaying.
- Showing both a snackbar and an inline banner for the same success message usually reads as duplicate feedback.
- Long slide distances make status messages feel like navigation. Keep movement under roughly 12 percent of the widget height.

For destructive actions, I still prefer `SnackBar` with undo. For form and sync states, I prefer an inline banner with a short `fadeIn` and tiny `slideY`. `flutter_animate` works best here when it stays local: the page owns page state, `ScaffoldMessenger` owns temporary messages, and no custom overlay has to guess where the app chrome is.
