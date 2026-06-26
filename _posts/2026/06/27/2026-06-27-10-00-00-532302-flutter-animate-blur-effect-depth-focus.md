---
layout: post
title: "flutter_animate BlurEffect: Animating Depth and Focus in Flutter UI"
description: "Learn how to use flutter_animate's BlurEffect to create focus-reveal modals, frosted-glass entrances, and directional blur animations with working code examples."
date: 2026-06-27
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

# flutter_animate BlurEffect: Animating Depth and Focus in Flutter UI

![Blurred bokeh lights with sharp center focus — depth effect photography](https://images.unsplash.com/photo-1518655048521-f130df041f66?w=800&q=80)

`flutter_animate` ships a `BlurEffect` that runs a `BackdropFilter`-style blur through the animation timeline. You use it as `.blurXY()`, `.blurX()`, or `.blurY()`, and it just works — no `AnimationController`, no `TweenAnimationBuilder`, no manual `ImageFilter.blur()` wiring.

Short answer: blur in, blur out, directional blur, and focus-reveal — all under 10 lines each.

## What BlurEffect actually does

Under the hood, `BlurEffect` wraps your widget in an `ImageFilter` applied via `BackdropFilter` during the animation. It interpolates `sigmaX` and `sigmaY` from a start value to an end value over the effect's duration.

The defaults are:
- `begin`: `Offset(4.0, 4.0)` — starts blurred
- `end`: `Offset(0.0, 0.0)` — ends sharp

So `.blurXY()` with no arguments gives you "sharp reveal from blur" — blurred at `t=0`, in focus at `t=duration`. That's the most useful pattern.

```dart
// Basic focus reveal
Text('Hello world')
  .animate()
  .blurXY(); // blurs in from 4.0, clears to 0 over 300ms
```

## The three blur methods

### `.blurXY()` — uniform blur

Both axes animate together. The most common usage.

```dart
Container(
  width: 200,
  height: 200,
  color: Colors.blue,
)
  .animate()
  .blurXY(begin: 8.0, end: 0.0, duration: 400.ms);
```

### `.blurX()` and `.blurY()` — directional blur

Useful for motion blur that implies direction. A card sliding in from the left blurs horizontally as it enters, then snaps into focus.

```dart
Card(child: content)
  .animate()
  .slideX(begin: -0.3, end: 0.0, duration: 350.ms)
  .blurX(begin: 6.0, end: 0.0, duration: 350.ms);
```

Horizontal motion → horizontal blur. Much more convincing than uniform blur for slide-in patterns.

![Motion blur visualization showing directional blur on a sliding UI card](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## Focus-reveal pattern

This is probably the most practical use case. You show a dialog or bottom sheet, and the background content blurs as the overlay appears. The effect gives the app a depth layer that feels intentional rather than flat.

Here's a pattern that blurs content when an overlay is shown, then removes the blur when dismissed:

```dart
class _FocusRevealState extends State<FocusRevealPage> {
  bool _showOverlay = false;

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        // Background content — blurs when overlay is active
        Animate(
          target: _showOverlay ? 1.0 : 0.0,
          effects: [
            BlurEffect(
              begin: Offset.zero,
              end: const Offset(8.0, 8.0),
              duration: 300.ms,
            ),
          ],
          child: _buildMainContent(),
        ),

        // Overlay
        if (_showOverlay)
          GestureDetector(
            onTap: () => setState(() => _showOverlay = false),
            child: Container(
              color: Colors.black45,
              alignment: Alignment.center,
              child: _buildModal()
                  .animate()
                  .fadeIn(duration: 250.ms)
                  .scale(begin: const Offset(0.9, 0.9)),
            ),
          ),
      ],
    );
  }
}
```

The trick here is using `Animate` with a `target` rather than auto-playing. When `_showOverlay` is `true`, `target` is `1.0` and the blur animates forward. When it's `false`, it animates back to `0.0`. No duplicate animation controllers needed.

## Frosted-glass modal entrance

A frosted card that blurs into clarity on appear. Works well for bottom sheets and cards.

```dart
Widget _buildFrostedCard() {
  return ClipRRect(
    borderRadius: BorderRadius.circular(16),
    child: BackdropFilter(
      filter: ImageFilter.blur(sigmaX: 10, sigmaY: 10),
      child: Container(
        color: Colors.white.withOpacity(0.2),
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text('Confirm action', style: TextStyle(fontSize: 20)),
            const SizedBox(height: 16),
            const Text('Are you sure you want to proceed?'),
          ],
        ),
      ),
    ),
  )
      .animate()
      .fadeIn(duration: 300.ms)
      .blurXY(begin: 12.0, end: 0.0, duration: 300.ms)
      .scale(begin: const Offset(0.95, 0.95), end: const Offset(1.0, 1.0));
}
```

The `blurXY` starts at `12.0` — visibly blurry — and clears as the card fades in and scales up. The three effects run simultaneously (same duration, no `.then()` needed), which is what creates the cohesive "materializing from nothing" feel.

## The performance gotcha with blur

Here's what I wasted an afternoon on: `BlurEffect` is expensive on older devices.

`BackdropFilter` (which `BlurEffect` uses internally) requires a compositing layer. Each animated frame reads pixels behind the widget and applies a Gaussian blur. On a mid-range device, a single blurred widget animating at 60fps is fine. Three or four concurrent blur animations on a list screen is not.

The fix is to not animate blur on every list item. Instead:

```dart
// ❌ Avoid: blur on every item in a list
ListView.builder(
  itemBuilder: (context, index) => ListTile(...)
      .animate()
      .blurXY(), // 50 items = 50 concurrent BackdropFilter operations
);

// ✅ Better: blur the whole list container once
ListView.builder(
  itemBuilder: (context, index) => ListTile(...),
)
  .animate()
  .blurXY(duration: 300.ms);
```

The container blur applies one compositing layer. Individual item blurs apply N layers. Same visual result on entrance, massively different GPU cost.

Also worth noting: `blurXY(end: 0.0)` actually removes the `BackdropFilter` layer once the animation finishes (because sigma = 0 is a no-op). So there's no performance penalty on idle state — the cost is only during the animation itself.

## Combining blur with saturation for a dramatic reveal

flutter_animate also has a `SaturationEffect`. Combining desaturated + blurred → full color + sharp is a nice content-reveal pattern:

```dart
Image.network(imageUrl)
  .animate()
  .blurXY(begin: 10.0, end: 0.0, duration: 600.ms, curve: Curves.easeOut)
  .saturate(begin: 0.0, end: 1.0, duration: 600.ms);
```

The image loads as a gray blur and materializes into sharp, full-color as it "arrives". Works especially well on image cards in a feed layout.

---

Next up: color tint animations with `flutter_animate`'s `TintEffect` — useful for status feedback (success green flash, error red pulse) without touching your theme.

**References:**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [BackdropFilter Flutter docs](https://api.flutter.dev/flutter/widgets/BackdropFilter-class.html)
