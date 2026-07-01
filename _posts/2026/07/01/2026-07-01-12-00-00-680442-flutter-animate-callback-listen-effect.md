---
layout: post
title: "flutter_animate CallbackEffect - Timeline Hooks Without Extra Controllers"
description: "Use flutter_animate CallbackEffect and ListenEffect to trigger haptics, logging, and progress updates from an animation timeline."
date: 2026-07-01
tags: [flutter_animate, animation, performance, state_management]
comments: true
share: true
---

![Flutter animation timeline with callback checkpoints on a laptop screen](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

Short answer: `CallbackEffect` is the cleanest way to attach a one-time side effect to a `flutter_animate` timeline, while `ListenEffect` is better when you need continuous progress. I use callbacks for haptics, sound cues, and logging checkpoints, and listeners for UI state that must track the animation value.

The mistake I made early was putting every side effect in `onComplete`. That works for a simple fade, but it breaks down when the animation has phases. A card may scale in, pause, shimmer, and then reveal an action. If the tap feedback should happen halfway through the scale, `onComplete` is too late and a separate `AnimationController` is too much code.

## CallbackEffect Fits Timeline Events

`CallbackEffect` runs at a specific point in the chain. It receives a boolean that tells you whether the animation is currently reversing, which matters if the same timeline plays forward and backward.

This example fires haptic feedback after the button has visibly started moving, not at tap time.

```dart
ElevatedButton.icon(
  onPressed: save,
  icon: const Icon(Icons.cloud_upload),
  label: const Text('Upload'),
)
    .animate(target: isUploading ? 1 : 0)
    .scale(
      begin: const Offset(1, 1),
      end: const Offset(.96, .96),
      duration: 180.ms,
      curve: Curves.easeOut,
    )
    .callback(
      duration: 90.ms,
      callback: (isReverse) {
        if (!isReverse) HapticFeedback.selectionClick();
      },
    )
    .then()
    .shimmer(duration: 900.ms);
```

The important detail is placement. The callback inherits timing context from the chain, so putting it after `scale()` makes it part of that sequence. If you move it after `.then()`, it belongs to the next phase instead.

## Use ListenEffect For Progress

`ListenEffect` is different. It reports animation values over time. That makes it useful for progress indicators, scrubbing labels, debug overlays, or feeding a lightweight `ValueNotifier`.

Here the visual animation and the external percentage stay tied to the same curve.

```dart
final revealProgress = ValueNotifier<double>(0);

Widget buildRevealCard() {
  return ProductCard()
      .animate(target: expanded ? 1 : 0)
      .fadeIn(duration: 240.ms, curve: Curves.easeOutCubic)
      .listen(
        duration: 240.ms,
        curve: Curves.easeOutCubic,
        callback: (value) => revealProgress.value = value,
      )
      .slideY(begin: .08, end: 0, duration: 240.ms);
}
```

I avoid calling `setState` directly inside `listen()`. It can rebuild too much of the screen while the animation ticks. A `ValueNotifier`, small provider update, or isolated `AnimatedBuilder` keeps the blast radius smaller.

## A Practical Pattern

For production code, I keep visual effects, timeline callbacks, and state mutations separate. The animation chain should still read like a timeline, not like a business workflow hidden inside widget decoration.

```dart
Widget buildStatusPill() {
  return StatusPill(status: status)
      .animate(target: status == SyncStatus.done ? 1 : 0)
      .fadeIn(duration: 160.ms)
      .scale(begin: const Offset(.92, .92), duration: 220.ms)
      .callback(
        callback: (reverse) {
          if (!reverse) analytics.logEvent(name: 'sync_badge_revealed');
        },
      )
      .then(delay: 120.ms)
      .shake(hz: 4, duration: 300.ms);
}
```

This keeps the analytics event aligned with what the user actually sees. If the widget never reaches that phase because the target flips back, the callback does not pretend the reveal happened.

## Things That Bite

Callbacks can run again when the animation reverses. Always check the `reverse` flag when the side effect should only happen on forward playback.

Do not put network writes, database commits, or navigation decisions inside animation callbacks. Those actions are too important to depend on a widget timeline. Use callbacks for presentation-adjacent effects: haptics, sound, analytics, debug traces, or small local flags.

Be careful in lists. If a row gets rebuilt with a new key, its animation can restart and the callback can fire again. Stable keys and idempotent callbacks matter more than clever timing.

## Quick Recap

Use `CallbackEffect` when one event belongs at one point in the animation. Use `ListenEffect` when another part of the UI needs the current animated value. Keep heavy app logic outside the timeline, guard reverse playback, and make callbacks safe to repeat when widgets rebuild.
