---
layout: post
title: "flutter_animate SwapEffect - Replace Flutter Widgets Without a Hard Layout Jump"
description: "Learn how flutter_animate SwapEffect replaces a Flutter widget mid-animation, preserves space, and avoids stale builder and state mistakes."
date: 2026-08-09
tags: [flutter_animate, animation, performance, state_management]
comments: true
share: true
---

![flutter_animate SwapEffect replacing a Flutter status card](/assets/images/flutter-animate-swap-effect-state-replacement.png)

The useful case for `flutter_animate` `SwapEffect` is not a simple fade between two colors. It is replacing one widget with another at a known point in an animation while keeping the surrounding layout stable. A saving card can fade out, become a success card, and fade in again without manually coordinating two animation controllers.

## Why a normal rebuild feels abrupt

Changing `isSaving` in `build` immediately changes the child tree. Flutter lays out the new card on the next frame, but there is no transition boundary unless you add one. `AnimatedSwitcher` is a good general solution, yet it can be more machinery than needed when the sequence is fixed: animate the current child, swap once, then animate the replacement.

`SwapEffect` is designed for that sequence. Its builder receives the original child, and the returned widget becomes the replacement at the effect's point on the timeline. Effects before the swap belong to the outgoing child; effects inside the returned `Animate` instance control the incoming child.

## A saving-to-success transition

The following example deliberately gives both cards the same outer constraints. That decision matters more than the glow or easing curve: if the replacement has a different height, the parent can still jump even though the visual transition is smooth.

```dart
Widget buildUploadStatus(String uploadId) {
  return UploadStatusCard(
    label: 'Uploading',
    icon: Icons.cloud_upload,
  ).animate(
    key: ValueKey('upload-$uploadId'),
  ).fadeOut(
    duration: 180.ms,
    curve: Curves.easeOut,
  ).swap(
    builder: (context, originalChild) {
      return UploadStatusCard(
        label: 'Uploaded',
        icon: Icons.check_circle,
      ).animate().fadeIn(
            duration: 220.ms,
            curve: Curves.easeOut,
          );
    },
  );
}
```

The `SwapEffect` inherits the timing boundary from `fadeOut`. The outgoing card fades away, the builder's card replaces it, and that new `Animate` instance owns the fade-in. The key is intentional: a new upload ID creates a fresh animation. Without it, a rebuild can preserve the existing animation state and make a second upload appear to skip the transition.

In a real widget, avoid using `completed` as the only state if the same state can be entered repeatedly. A status enum or upload ID gives you a clearer replay rule:

```dart
enum UploadPhase { uploading, complete, failed }

Key animationKey(UploadPhase phase, String uploadId) {
  return ValueKey('$uploadId-$phase');
}
```

Use the upload ID when every new upload should replay. Use only the phase when the animation should happen once per phase change.

## The builder is not a reactive slot

This is the trap that caused the most confusing result in my test. `SwapEffect`'s builder is called once when the effect initially builds. It is not a callback that keeps reading the latest provider, `ValueNotifier`, or `setState` value. If the replacement depends on changing data, build a new `Animate` subtree with a deliberate key, or use `AnimatedSwitcher` for a continuously reactive child.

| Requirement | Better fit | Reason |
| --- | --- | --- |
| Fixed outgoing-to-incoming sequence | `SwapEffect` | One timeline and one replacement point |
| Arbitrary reactive child replacement | `AnimatedSwitcher` | Reconciles changing children continuously |
| Several independent effects on one child | `ThenEffect` | Sequences effects without replacing the child |
| Replaying a known animation | `Animate` with a new key | Resets the controller intentionally |

Do not place a changing network value directly inside the `SwapEffect` builder and expect it to update during the same animation. Capture the value when constructing the animation, or move the changing content below a stable animated shell.

## Keep the layout stable

`SwapEffect` changes the child, not the parent's sizing rules. I use one of these approaches when the two states are visually different:

- Give both cards the same `SizedBox` height and width.
- Put the animated content inside a fixed-height `ConstrainedBox`.
- Keep the icon and label regions aligned even when the text length changes.

For a compact status row, a 64 px fixed height is usually enough. A failure message may need two lines while a success message needs one, so reserve the larger height before tuning the animation. Otherwise the user sees a layout shift underneath a perfectly timed fade.

## Practical checks

- Use `SwapEffect` for a deliberate one-time replacement, not as a state-management primitive.
- Give the outer `Animate` a key when a state change must restart the timeline.
- Keep the replacement builder stable and remember that it is called once initially.
- Match the outgoing and incoming constraints before adjusting duration.
- Test a repeated transition, a fast state change, and a reduced-motion setting.

The current API exposes `SwapEffect` with a builder that receives both `BuildContext` and the original child. The [official flutter_animate API documentation](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/SwapEffect-class.html) also points out that the builder returns a new `Animate` instance with its own controller. That separation is useful: the outgoing fade can finish independently from the incoming card's fade-in.
