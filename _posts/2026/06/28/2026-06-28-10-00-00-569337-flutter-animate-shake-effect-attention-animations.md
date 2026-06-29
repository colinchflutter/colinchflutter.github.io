---
layout: post
title: "flutter_animate ShakeEffect: Attention-Seeker Animations for Error States and Notifications"
description: "How to use flutter_animate's ShakeEffect and related attention animations to signal errors, draw focus to UI elements, and build effective onboarding nudges — with working code examples."
date: 2026-06-28
tags: [flutter_animate, animation, Flutter, Android, iOS]
comments: true
share: true
---

![Red alert notification badge glowing on dark background](https://images.unsplash.com/photo-1563986768609-322da13575f3?w=800&q=80)

Shake animations are one of those things that look trivial until you try to implement them from scratch. With `AnimationController` and `Tween` you end up writing thirty lines of boilerplate for a two-frame jiggle. `flutter_animate`'s `ShakeEffect` handles this in one chained call — and it's more flexible than the name suggests.

Short answer: `.shake()` for horizontal jitter, combine with `.scale()` or `.fade()` for richer attention effects, and keep durations under 400ms or users think the UI is broken.

## What ShakeEffect Actually Does

`ShakeEffect` applies a rapid oscillating translation to the widget. The default is horizontal (left-right), but you can control the axis, frequency, and rotation.

```dart
// Default: horizontal shake, 3 oscillations, 300ms
widget.animate().shake()

// Explicit parameters
widget.animate().shake(
  hz: 4,                  // oscillations per second
  offset: const Offset(4, 0),  // max displacement in logical pixels
  rotation: 0.05,         // slight rotation per oscillation (radians)
  duration: 400.ms,
  curve: Curves.easeInOut,
)
```

The `hz` parameter is the key control — `3` feels like a gentle nudge, `8` feels like a real error. Default is `3`.

One thing that caught me off guard: `ShakeEffect` doesn't loop by default. It plays once and stops. If you want a pulsing attention effect, you need `.animate(onPlay: (c) => c.repeat())` or the `.loop()` modifier.

## Error State: Form Validation Shake

The most common use case. User submits a form with missing fields, the input shakes to indicate the problem.

```dart
class ShakingTextField extends StatefulWidget {
  final TextEditingController controller;
  final String hint;
  const ShakingTextField({super.key, required this.controller, required this.hint});

  @override
  State<ShakingTextField> createState() => _ShakingTextFieldState();
}

class _ShakingTextFieldState extends State<ShakingTextField> {
  bool _hasError = false;
  final _key = GlobalKey();

  void triggerShake() {
    setState(() => _hasError = true);
    Future.delayed(500.ms, () {
      if (mounted) setState(() => _hasError = false);
    });
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: widget.controller,
      decoration: InputDecoration(
        hintText: widget.hint,
        border: OutlineInputBorder(
          borderSide: BorderSide(
            color: _hasError ? Colors.red : Colors.grey,
          ),
        ),
      ),
    )
    .animate(
      key: ValueKey(_hasError),
      target: _hasError ? 1.0 : 0.0,
    )
    .shake(
      hz: 5,
      offset: const Offset(6, 0),
      duration: 400.ms,
    );
  }
}
```

The `ValueKey(_hasError)` trick is important — it forces `flutter_animate` to rebuild the animation when `_hasError` flips, which re-triggers the shake on subsequent validation failures. Without the key, the second submission doesn't shake because the animation state is the same.

![Flutter form validation error shake animation example](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## Notification Badge: Continuous Attention Pulse

For a notification badge or unread indicator, you want the animation to keep running — not a one-shot shake but a slow, repeating pulse that says "hey, look at me."

Combining `shake` with `scale` works well here:

```dart
class PulsingBadge extends StatelessWidget {
  final int count;
  const PulsingBadge({super.key, required this.count});

  @override
  Widget build(BuildContext context) {
    if (count == 0) return const SizedBox.shrink();

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
      decoration: BoxDecoration(
        color: Colors.red,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(
        '$count',
        style: const TextStyle(color: Colors.white, fontSize: 11),
      ),
    )
    .animate(onPlay: (c) => c.repeat(reverse: true))
    .scale(
      begin: const Offset(1.0, 1.0),
      end: const Offset(1.15, 1.15),
      duration: 700.ms,
      curve: Curves.easeInOut,
    );
  }
}
```

I tried adding `.shake()` to this too — it ends up looking frantic. Scale pulse alone is calmer and less annoying for a badge that stays on screen for seconds or minutes.

## Onboarding Nudge: "Tap Here" Arrow

When you want to draw attention to a specific button during onboarding, a bounce is more readable than a shake.

`flutter_animate` doesn't have a dedicated `BounceEffect`, but combining `slide` and `fade` gives the same result:

```dart
Widget buildOnboardingArrow() {
  return const Icon(Icons.arrow_downward, size: 32, color: Colors.blue)
      .animate(onPlay: (c) => c.repeat(reverse: true))
      .slideY(
        begin: 0,
        end: 0.3,
        duration: 600.ms,
        curve: Curves.easeInOut,
      )
      .fadeIn(begin: 0.6);
}
```

The `begin: 0.6` on `fadeIn` keeps the arrow visible even at the top of the bounce — a fully transparent arrow at the start of each cycle is jarring. Spent more time than I should have on that tweak.

## Combining Shake With Color: Error Flash + Shake

Pairing `ShakeEffect` with `TintEffect` makes error feedback much clearer — the widget shakes *and* flashes red at the same time.

```dart
Widget buildErrorIcon({required bool hasError}) {
  return const Icon(Icons.warning_rounded, size: 28)
      .animate(
        key: ValueKey(hasError),
        target: hasError ? 1.0 : 0.0,
      )
      .shake(
        hz: 6,
        offset: const Offset(5, 0),
        duration: 350.ms,
      )
      .tint(
        color: Colors.red,
        end: 0.7,
        duration: 350.ms,
      );
}
```

Both effects run in parallel since they're chained without `.then()`. The visual result: the icon shakes and turns red simultaneously, then snaps back to normal. The tint going to `0.7` makes it noticeably red without fully washing out the icon.

## Rotation Shake: The "Wrong Answer" Effect

Games and quiz apps often use a rotation-based shake rather than a translation shake. Set `offset` to zero and use `rotation` only:

```dart
widget.animate().shake(
  hz: 4,
  offset: Offset.zero,
  rotation: 0.08,   // about 4.5 degrees
  duration: 400.ms,
)
```

This reads as "no" rather than "error" — which is exactly what you want for wrong answer feedback in a quiz. Discovered this when a product designer pointed out that horizontal jitter felt "technical" while rotation felt "human."

## Triggering Programmatically With AnimationController

Sometimes you need to trigger the shake from external logic — like after an API call fails. Pass a controller via the `controller` parameter:

```dart
class _MyWidgetState extends State<MyWidget> {
  final _controller = AnimationController(vsync: this, duration: 400.ms);

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Future<void> _submitForm() async {
    final success = await apiCall();
    if (!success && mounted) {
      _controller.forward(from: 0);
    }
  }

  @override
  Widget build(BuildContext context) {
    return submitButton
        .animate(controller: _controller, autoPlay: false)
        .shake(hz: 5, duration: 400.ms);
  }
}
```

`autoPlay: false` stops the animation from playing on mount. `_controller.forward(from: 0)` resets and plays it from scratch each time — necessary if you want the shake to work on repeated failures.

## Duration and Frequency: The Numbers That Matter

After shipping a few different shake animations, here's what I've settled on:

| Use case | `hz` | `duration` | `offset` |
|---|---|---|---|
| Subtle nudge | 3 | 300ms | `(3, 0)` |
| Form validation error | 5 | 400ms | `(6, 0)` |
| Critical error | 7 | 350ms | `(8, 0)` |
| Attention pulse (looping) | 2 | 800ms | `(2, 0)` |

Going above `hz: 8` looks like a glitch, not an animation. Going above `500ms` makes users think the app hung. The sweet spot for most cases is `hz: 4–6` at `300–400ms`.

---

Next up: `CallbackEffect` and `ListenEffect` in flutter_animate — how to hook into animation lifecycle events to trigger side effects without managing `AnimationController` directly.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [ShakeEffect API docs](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ShakeEffect-class.html)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
