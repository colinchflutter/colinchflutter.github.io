---
layout: post
title: "flutter_animate Gesture-Driven Animations — Controlling Effects with GestureDetector"
description: "Learn how to tie flutter_animate effects directly to user gestures using Animate.controller. Includes a swipe-to-reveal card example with manual AnimationController progress control."
date: 2026-06-25
tags: [flutter_animate, animation, GestureDetector, AnimationController, Flutter]
comments: true
share: true
---

# flutter_animate Gesture-Driven Animations — Controlling Effects with GestureDetector

![Flutter gesture-driven animation interactive card swipe](https://images.unsplash.com/photo-1555421689-491a97ff2040?w=800&q=80)

Most flutter_animate usage is fire-and-forget — widget appears, animation plays, done. But sometimes you need the animation to *follow* the user's finger. Drag progress controls opacity, swipe distance controls a card flip, pan gesture drives a parallax shift. Turns out flutter_animate handles this cleanly through `Animate.controller`.

## The Problem with Auto-Playing Animations

The default `.animate()` chain starts playing immediately when the widget builds. That's great for entrance effects, but useless when you want something like:

- A card that flips only as far as the user has dragged
- A delete row that slides open proportionally to the swipe distance
- A pull-to-refresh indicator that grows with drag progress

The missing piece is `AnimationController` — specifically, setting its `value` manually instead of calling `forward()`.

## How Animate.controller Works

Every `Animate` widget can accept an external `AnimationController`. When you provide one, flutter_animate stops driving the animation itself and just maps the controller's value (0.0 → 1.0) to your effect chain.

```dart
final _controller = AnimationController(vsync: this, duration: 400.ms);

Widget build(BuildContext context) {
  return MyCard()
    .animate(controller: _controller)
    .flipV(end: 1.0)
    .fadeOut(begin: 0.7);
}
```

Now instead of calling `_controller.forward()`, you set `_controller.value` directly from gesture callbacks.

## Building a Swipe-to-Flip Card

Here's a full working example — a card that flips vertically as you drag it up, snapping to full flip on release.

```dart
class FlipCard extends StatefulWidget {
  const FlipCard({super.key});
  @override
  State<FlipCard> createState() => _FlipCardState();
}

class _FlipCardState extends State<FlipCard>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  double _dragStartY = 0;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: 300.ms);
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _onPanStart(DragStartDetails d) {
    _dragStartY = d.globalPosition.dy;
  }

  void _onPanUpdate(DragUpdateDetails d) {
    final dragDelta = _dragStartY - d.globalPosition.dy;
    // 200px drag = full flip
    final progress = (dragDelta / 200).clamp(0.0, 1.0);
    _ctrl.value = progress;
  }

  void _onPanEnd(DragEndDetails d) {
    // snap: if past halfway, complete the flip
    if (_ctrl.value > 0.5) {
      _ctrl.forward();
    } else {
      _ctrl.reverse();
    }
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onPanStart: _onPanStart,
      onPanUpdate: _onPanUpdate,
      onPanEnd: _onPanEnd,
      child: Container(
        width: 280,
        height: 180,
        decoration: BoxDecoration(
          color: Colors.indigo,
          borderRadius: BorderRadius.circular(16),
        ),
        child: const Center(
          child: Text(
            'Drag up to flip',
            style: TextStyle(color: Colors.white, fontSize: 18),
          ),
        ),
      )
          .animate(controller: _ctrl)
          .flipV(end: 1.0, curve: Curves.easeOut)
          .tint(color: Colors.purple, end: 0.4),
    );
  }
}
```

The key line is `_ctrl.value = progress` — you're not animating to a value, you're *setting* it directly. flutter_animate reflects whatever the controller's current value is.

![Flutter AnimationController value gesture mapping diagram](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

## Chaining Effects Still Works

One thing I wasn't sure about: do `.then()` chains work correctly when driving the controller manually?

Short answer: yes, but with a quirk. `ThenEffect` adds a delay offset internally. When you scrub the controller to 0.6, flutter_animate figures out which effects are active at that point in the timeline and renders them correctly. So this still works as expected:

```dart
widget
  .animate(controller: _ctrl)
  .slideY(end: -0.3, duration: 200.ms)
  .then()  // starts after slideY
  .fadeOut(duration: 200.ms)
```

Scrubbing to 0.5 shows the widget mid-slide. Scrubbing to 0.8 shows it partially faded. Works fine.

## The Gotchas

**1. Don't forget `vsync`.**  
`AnimationController` needs a `TickerProvider`. Use `SingleTickerProviderStateMixin` on your State class, or `TickerProviderStateMixin` if you have multiple controllers.

**2. Clamp gesture values.**  
Raw drag deltas can easily go negative or exceed 1.0. `_ctrl.value` throws if you set it outside [0.0, 1.0]. Always clamp:
```dart
_ctrl.value = (rawProgress).clamp(0.0, 1.0);
```

**3. Duration matters for snap animations.**  
When you call `_ctrl.forward()` or `_ctrl.reverse()` at the end of a gesture, it animates from the current value to 1.0 or 0.0 over the full `duration`. If `duration` is 2 seconds and the user is at 0.9, it still takes 2 seconds to finish. Use a shorter snap duration or calculate remaining time:
```dart
final remainingMs = ((1.0 - _ctrl.value) * 300).round();
_ctrl.animateTo(1.0, duration: Duration(milliseconds: remainingMs));
```

**4. `autoPlay` is ignored when you pass a controller.**  
Setting `autoPlay: false` on `.animate()` has no effect once you pass a controller — the controller is in full charge. Don't confuse yourself by leaving `autoPlay: false` in the chain.

## When to Use This vs AnimatedBuilder

flutter_animate's gesture-driven mode is cleanest when:
- You already have an effect chain you like and just want gesture control
- The animation has a clear start (0.0) and end (1.0) state

Reach for raw `AnimatedBuilder` + `AnimationController` when:
- You need to animate properties that flutter_animate doesn't cover
- The animation logic is complex enough that the declarative chain becomes confusing

For most swipe/drag UI, the flutter_animate approach saves a lot of boilerplate.

---

Next up: combining flutter_animate with `AnimatedSwitcher` for animated widget-swap transitions — the pattern that replaces most `showDialog`/`Navigator.push` entrance animations.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [AnimationController API docs](https://api.flutter.dev/flutter/animation/AnimationController-class.html)
- [flutter_animate GitHub — controller examples](https://github.com/gskinner/flutter_animate)
