---
layout: post
title: "flutter_animate SwapEffect: Mid-Animation Widget Replacement Done Right"
description: "Learn how flutter_animate's SwapEffect lets you replace a widget mid-animation — with real code for button state changes, icon swaps, and content reveals."
date: 2026-06-27
tags: [flutter_animate, animation, Flutter, state_management]
comments: true
share: true
---

![Widget transformation animation — morphing shapes on dark background](https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=800&q=80)

`SwapEffect` is probably the most underused effect in `flutter_animate`. It lets you swap the target widget for a completely different widget at a specific point in the animation timeline — without any `setState`, `AnimatedSwitcher`, or visibility hacks.

Short answer: if you need a widget to transform into a different widget mid-animation, `SwapEffect` is the cleanest way to do it.

## What SwapEffect actually does

Most `flutter_animate` effects tweak properties of the same widget over time — opacity, position, scale. `SwapEffect` is different. At a specified `delay`, it replaces the entire widget subtree with whatever you return from its `builder` callback.

```dart
Text("Loading...")
  .animate()
  .fadeIn(duration: 300.ms)
  .swap(
    duration: 400.ms,
    delay: 1200.ms,
    builder: (context, child) => Text("Done!"),
  )
  .fadeIn();
```

The `child` parameter in `builder` is the original widget (or the previous swap target). You can use it or ignore it. Most of the time you ignore it.

![Sequence diagram showing widget state changing over animation timeline](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## The pattern I actually use it for

Submit buttons. You know the pattern: button shows "Submit", you tap it, it spins, then it shows "Done ✓". With `flutter_animate` + `SwapEffect`, the whole thing is one chain:

```dart
class SubmitButton extends StatefulWidget {
  const SubmitButton({super.key});

  @override
  State<SubmitButton> createState() => _SubmitButtonState();
}

class _SubmitButtonState extends State<SubmitButton> {
  bool _submitted = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _submitted = true),
      child: _submitted
          ? ElevatedButton(
              onPressed: null,
              child: const Text("Submit"),
            )
              .animate()
              .fadeOut(duration: 150.ms)
              .swap(
                builder: (_, __) => const CircularProgressIndicator(),
              )
              .shake(duration: 0.ms, delay: 1500.ms) // placeholder delay
              .swap(
                delay: 1500.ms,
                builder: (_, __) => const Icon(Icons.check, color: Colors.green),
              )
              .scale(begin: const Offset(0.5, 0.5), delay: 1500.ms)
          : ElevatedButton(
              onPressed: () => setState(() => _submitted = true),
              child: const Text("Submit"),
            ),
    );
  }
}
```

The catch: `SwapEffect` fires once on mount. So if `_submitted` becomes `true`, the animation runs its full chain top to bottom — loading spinner → checkmark. No need to manually sequence anything.

## Swapping back to the original widget

The `builder` callback receives the *original* widget as `child`. Turns out this is useful when you want to animate out, do something, and then animate back in — essentially a "wipe and reload" pattern.

```dart
Container(color: Colors.blue, width: 100, height: 100)
  .animate(onComplete: (controller) => controller.repeat())
  .scaleXY(end: 1.3, duration: 600.ms)
  .swap(
    delay: 600.ms,
    builder: (context, child) {
      // child is the original Container
      return child!
          .animate()
          .scaleXY(begin: 1.3, end: 1.0, duration: 600.ms);
    },
  );
```

I spent a while wondering why `onComplete` wasn't retriggering the swap. The answer: `SwapEffect` replaces the widget once. If you need it to repeat, you have to handle repeat at the `Animate` level and rebuild the widget tree.

## Icon toggle pattern

This one comes up constantly — toggling between two icon states (bookmark/bookmarked, heart/hearted, etc.):

```dart
class FavoriteIcon extends StatefulWidget {
  const FavoriteIcon({super.key});

  @override
  State<FavoriteIcon> createState() => _FavoriteIconState();
}

class _FavoriteIconState extends State<FavoriteIcon> {
  bool _favorited = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _favorited = !_favorited),
      child: Icon(
        _favorited ? Icons.favorite : Icons.favorite_border,
        size: 32,
      )
          .animate(key: ValueKey(_favorited))
          .scaleXY(
            begin: 0.6,
            end: 1.0,
            curve: Curves.elasticOut,
            duration: 500.ms,
          ),
    );
  }
}
```

Note: I'm using `key: ValueKey(_favorited)` on `.animate()` here instead of `SwapEffect` — when the key changes, Flutter rebuilds the widget and restarts the animation. This is actually cleaner than `SwapEffect` for toggle patterns. Use `SwapEffect` when you want one continuous chain, not when you want the same animation to replay on every state change.

## Where SwapEffect beats AnimatedSwitcher

`AnimatedSwitcher` is the standard tool for swapping widgets, but it has quirks:

- Requires explicit `key` management on child widgets
- In/out animations run simultaneously by default (you need `transitionBuilder` to sequence them)
- Verbose setup for anything beyond a simple crossfade

`SwapEffect` wins when:
1. The swap is part of a larger animation chain (blur in → load → swap to result)
2. You want precise timing control (swap at `delay: 1500.ms` exactly)
3. You're already in an `flutter_animate` chain and don't want to break out to `AnimatedSwitcher`

`AnimatedSwitcher` wins when:
1. The widget can swap in either direction (add/remove from list)
2. You need the swap to respond to external state repeatedly
3. The in/out animation needs to be symmetric

## One thing that trips people up

`SwapEffect` replaces the widget *structurally*. The returned widget from `builder` doesn't inherit the effects applied to the original chain below the swap. So this doesn't do what you might expect:

```dart
// ❌ The fadeIn below the swap applies to the ORIGINAL widget,
// not to what SwapEffect returns
Icon(Icons.cloud)
  .animate()
  .swap(
    delay: 500.ms,
    builder: (_, __) => Icon(Icons.cloud_done),
  )
  .fadeIn(); // This fadeIn runs on the original Icon(Icons.cloud)
```

If you want effects on the swapped widget, apply them inside the `builder`:

```dart
// ✓ Correct
Icon(Icons.cloud)
  .animate()
  .fadeOut(duration: 300.ms)
  .swap(
    delay: 300.ms,
    builder: (_, __) => Icon(Icons.cloud_done)
        .animate()
        .fadeIn(duration: 300.ms)
        .scaleXY(begin: const Offset(0.8, 0.8)),
  );
```

---

Next up: `ColorEffect` — animating color overlays and blend modes with `flutter_animate` for highlight and tint effects.

**Reference:**
- [flutter_animate SwapEffect API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/SwapEffect-class.html)
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
