---
layout: post
title: "CallbackEffect and ListenEffect in flutter_animate — Hooking Into Animation Lifecycle Events"
description: "Learn how to use flutter_animate's CallbackEffect and ListenEffect to trigger side effects, fire haptic feedback, and monitor animation progress values without touching AnimationController directly."
date: 2026-06-28
tags: [flutter_animate, animation, Flutter, Android, iOS]
comments: true
share: true
---

![Developer staring at code on a dark monitor with glowing terminal output](https://images.unsplash.com/photo-1517694712202-14dd9538aa97?w=800&q=80)

Most animation side-effect problems come down to the same question: how do you fire haptic feedback exactly when a card lands, play a sound cue mid-animation, or update state when an animation crosses the halfway point? With a raw `AnimationController` you'd add a `StatusListener` or an `addListener` callback. With `flutter_animate`, `CallbackEffect` and `ListenEffect` handle both cases inline, without exposing the controller at all.

Short answer: `CallbackEffect` fires a one-shot callback at a specific point in the animation timeline. `ListenEffect` streams the current animation value continuously. Know which you need before you reach for either.

## CallbackEffect — One-Shot Side Effects

`CallbackEffect` triggers a callback exactly once when the animation reaches the effect's position in the chain.

```dart
widget
  .animate()
  .fadeIn(duration: 300.ms)
  .scale(duration: 200.ms)
  .callback(callback: (_) {
    HapticFeedback.lightImpact();
  });
```

The callback fires after both `fadeIn` and `scale` complete — because it's chained after them. The `_` parameter is the `AnimationController` value at that moment (a `double` between 0 and 1), which you can ignore if you don't need it.

### Positioning the callback mid-chain

The callback fires at whatever point in the chain it appears. Put it between effects to fire mid-animation:

```dart
widget
  .animate()
  .slideX(begin: -1, end: 0, duration: 400.ms)   // slide in
  .callback(callback: (_) {
    // fires when slide finishes, before the scale
    audioPlayer.play(AssetSource('whoosh.mp3'));
  })
  .scale(begin: const Offset(0.9, 0.9), duration: 200.ms);  // then scale
```

This is the pattern I use for UI confirmation sounds — play the sound exactly when the element is in position, not before or after.

### Delay within callback

If you need the callback slightly after the preceding effect, use the `delay` parameter:

```dart
widget
  .animate()
  .fadeIn(duration: 500.ms)
  .callback(
    delay: 100.ms,
    callback: (_) => setState(() => _isVisible = true),
  );
```

The callback fires 600ms from the start (500ms fade + 100ms delay). Useful when you want to update state slightly after the animation visually settles — avoids the frame where the widget is animating in but state hasn't caught up yet.

![Animation timeline diagram with callback markers at specific points](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## The mounted Check Problem

Here's the thing nobody mentions in the docs: `CallbackEffect` callbacks can fire after the widget is disposed if the animation is long or the user navigates away. This crashes.

```dart
// Dangerous: fires after dispose
.callback(callback: (_) => setState(() => _done = true))

// Safe: check mounted first
.callback(callback: (_) {
  if (mounted) setState(() => _done = true);
})
```

Got this wrong the first time. Debugger showed a `setState called after dispose()` exception that only happened during fast navigation in tests. The fix is obvious in hindsight.

## ListenEffect — Continuous Value Monitoring

`ListenEffect` streams the animation value continuously — every frame, from 0.0 to 1.0. The callback receives the current progress value.

```dart
widget
  .animate()
  .custom(
    duration: 1.seconds,
    builder: (context, value, child) => child!,  // just driving time
  )
  .listen(callback: (value) {
    // value goes from 0.0 to 1.0 over 1 second
    _progressNotifier.value = value;
  });
```

The typical use case: drive a separate piece of UI from the animation progress. A progress bar, a counter, a parallax layer — anything that needs to track the animation value but isn't the animated widget itself.

### Practical Example: Synced Progress Indicator

Suppose you have a card that slides in while a separate progress bar fills in sync:

```dart
class SyncedCardEntry extends StatefulWidget {
  const SyncedCardEntry({super.key});

  @override
  State<SyncedCardEntry> createState() => _SyncedCardEntryState();
}

class _SyncedCardEntryState extends State<SyncedCardEntry> {
  final _progress = ValueNotifier<double>(0);

  @override
  void dispose() {
    _progress.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Card(
          child: const Padding(
            padding: EdgeInsets.all(16),
            child: Text('Loading complete'),
          ),
        )
        .animate()
        .slideY(begin: 0.3, end: 0, duration: 600.ms, curve: Curves.easeOut)
        .fadeIn(duration: 600.ms)
        .listen(callback: (value) {
          _progress.value = value;
        }),

        const SizedBox(height: 12),

        ValueListenableBuilder<double>(
          valueListenable: _progress,
          builder: (_, val, __) => LinearProgressIndicator(value: val),
        ),
      ],
    );
  }
}
```

The `ListenEffect` is the glue — it keeps the progress bar in sync with the card animation without any timer or manual controller management.

### Crossing a Threshold Once

`ListenEffect` fires every frame, so if you want to trigger something once when the animation passes 50%, you need to guard it yourself:

```dart
bool _halfFired = false;

widget.animate()
  .custom(duration: 1.seconds, builder: (_, __, child) => child!)
  .listen(callback: (value) {
    if (!_halfFired && value >= 0.5) {
      _halfFired = true;
      HapticFeedback.selectionClick();
    }
  });
```

Yes, it's a bit manual. `CallbackEffect` with a specific `delay` is cleaner if you know the exact timing. But if the timing depends on a curve (non-linear progress), `ListenEffect` with a threshold guard is the only way.

## Combining Both in One Chain

Nothing stops you from using both in a single chain:

```dart
widget
  .animate()
  .scale(
    begin: const Offset(0.8, 0.8),
    end: const Offset(1.0, 1.0),
    duration: 400.ms,
    curve: Curves.elasticOut,
  )
  .callback(callback: (_) {
    if (mounted) setState(() => _popped = true);
    HapticFeedback.mediumImpact();
  })
  .listen(callback: (value) {
    // post-scale progress, if you chain more effects after callback
    _postAnimProgress.value = value;
  });
```

The order matters: `callback` fires when the `scale` finishes; `listen` after that tracks whatever follows.

## When to Use Which

| Situation | Use |
|---|---|
| Fire haptic feedback when animation completes | `CallbackEffect` |
| Play a sound mid-animation | `CallbackEffect` with `delay` |
| Update state after animation settles | `CallbackEffect` (with `mounted` check) |
| Drive a separate progress indicator | `ListenEffect` |
| Trigger something at 50% progress | `ListenEffect` with threshold guard |
| Log animation timing for debugging | `ListenEffect` → `debugPrint` |

---

Next up: `FollowPathEffect` in flutter_animate — animating widgets along a custom `Path` for map markers, game characters, and tutorial highlights.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [CallbackEffect API docs](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/CallbackEffect-class.html)
- [ListenEffect API docs](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ListenEffect-class.html)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
