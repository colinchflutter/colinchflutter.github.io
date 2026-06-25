---
layout: post
title: "flutter_animate ScrollAdapter — scroll-driven animations without a single AnimationController"
description: "Drive flutter_animate effects directly from a ScrollController using ScrollAdapter. Covers parallax headers, fade-in cards, and sticky nav transforms with working Dart code."
date: 2026-06-25
tags: [flutter_animate, animation, Flutter, ScrollPhysics]
comments: true
share: true
---
![Mobile app with smooth scroll-driven animations](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

Short answer: wrap your `Animate` widget's controller in a `ScrollAdapter`, hand it a `ScrollController`, and the animation position tracks scroll offset automatically. No `addListener`, no manual `controller.value = ...`, no `setState` on every frame.

I wasted a morning building this from scratch — a parallax header that faded and scaled as you scrolled. Dozens of lines of controller wiring. Then I found `ScrollAdapter` buried in the flutter_animate docs. Replaced it all with about 15 lines.

## How ScrollAdapter works

`flutter_animate` uses *adapters* to hand off control of the `AnimationController` to an external source. `ScrollAdapter` is the scroll-specific one. It maps a scroll range (in pixels) to the animation's 0.0–1.0 value.

```dart
import 'package:flutter_animate/flutter_animate.dart';

class ScrollDemoPage extends StatefulWidget {
  const ScrollDemoPage({super.key});

  @override
  State<ScrollDemoPage> createState() => _ScrollDemoPageState();
}

class _ScrollDemoPageState extends State<ScrollDemoPage> {
  final _scrollCtrl = ScrollController();

  @override
  void dispose() {
    _scrollCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        controller: _scrollCtrl,
        slivers: [
          SliverToBoxAdapter(
            child: _buildHero(),
          ),
          // ... rest of page
        ],
      ),
    );
  }

  Widget _buildHero() {
    return SizedBox(
      height: 300,
      child: Stack(
        fit: StackFit.expand,
        children: [
          Image.network(
            'https://images.unsplash.com/photo-1506905925346-21bda4d32df4?w=1200&q=80',
            fit: BoxFit.cover,
          )
              .animate(
                adapter: ScrollAdapter(_scrollCtrl, begin: 0, end: 300),
              )
              .fadeOut()
              .scaleXY(end: 1.15),
        ],
      ),
    );
  }
}
```

The `begin` and `end` parameters are scroll offsets in pixels. When `ScrollController.offset` is at `begin`, the animation sits at 0.0. At `end`, it's at 1.0. Everything between is interpolated linearly.

## Parallax header — the most common use case

The example above is a parallax header in 15 lines. As you scroll down 300px, the image fades out and scales up slightly — the classic "magazine app" look.

![Parallax scroll effect on mobile app header](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

The catch: `ScrollAdapter` drives the animation *forward* as you scroll down. If you want the reverse (element appears as you scroll), flip the begin/end values or use `end` smaller than `begin` — but honestly that direction is confusing. I prefer composing effects forward and accepting that "appearing" means "start when scroll offset hits X".

## Fade-in cards on scroll

Staggered fade-in as the user scrolls down a list is trickier than it looks, because `ScrollAdapter` is for a *single* controller position. Each card needs its own begin/end range.

The trick: compute begin/end from the card's approximate position in the list.

```dart
Widget _buildCard(int index, ScrollController scrollCtrl) {
  // rough pixel position — adjust cardHeight to match your layout
  const cardHeight = 120.0;
  const triggerOffset = 50.0;
  final begin = (index * cardHeight) - triggerOffset;
  final end = begin + cardHeight;

  return Card(
    child: ListTile(
      title: Text('Item $index'),
      subtitle: const Text('Scroll to reveal'),
    ),
  )
      .animate(
        adapter: ScrollAdapter(scrollCtrl, begin: begin, end: end),
      )
      .fadeIn()
      .slideY(begin: 0.2, end: 0);
}
```

Works fine until the card height varies or you have sections. In that case, you're better off switching to a `VisibilityDetector`-based approach and using regular `flutter_animate` triggers. `ScrollAdapter` shines for *continuous* effects (parallax, sticky transforms), not for discrete "triggered once" reveals.

## Sticky nav that shrinks on scroll

Another great use case: a floating header that compresses as you scroll past the fold.

```dart
class _StickyHeader extends StatelessWidget {
  const _StickyHeader({required this.scrollCtrl});

  final ScrollController scrollCtrl;

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 80,
      color: Colors.white,
      child: Row(
        children: [
          const SizedBox(width: 16),
          Text(
            'My App',
            style: Theme.of(context).textTheme.headlineMedium,
          )
              .animate(
                adapter: ScrollAdapter(scrollCtrl, begin: 0, end: 200),
              )
              .scaleXY(end: 0.7)
              .fadeOut(end: 0.7),
          const Spacer(),
          IconButton(
            icon: const Icon(Icons.menu),
            onPressed: () {},
          )
              .animate(
                adapter: ScrollAdapter(scrollCtrl, begin: 0, end: 100),
              )
              .fadeIn(begin: 0, end: 0.5),  // icon fades IN as title shrinks
        ],
      ),
    );
  }
}
```

Here the title shrinks and fades while a hamburger icon fades in — all driven by the same scroll position, no `setState` anywhere.

## The gotcha I hit: multiple adapters, one controller

You can attach multiple `Animate` widgets to the same `ScrollController`. That's fine. The issue I hit: if you call `_scrollCtrl.dispose()` before the adapters are cleaned up, you get a `ChangeNotifier was disposed` error on the next frame.

Fix: make sure the `Animate` widgets are inside the same `StatefulWidget` that owns the `ScrollController`, so they unmount before `dispose()` runs. Or use a `KeepAlive` carefully. Don't put the scroll controller in a parent widget and pass it down to children that might outlive it.

## When not to use ScrollAdapter

- **One-shot animations** (element appears once when scrolled into view) → use `VisibilityDetector` + `.animate(onPlay: ...)` instead
- **Very long lists** → calculating `begin`/`end` per item gets messy; `AnimatedList` or `SliverList` with stagger logic is cleaner
- **Gesture-driven animations** (drag, swipe) → look at `ValueNotifierAdapter` instead

---

Next up: `ValueNotifierAdapter` and `ChangeNotifierAdapter` — hooking flutter_animate to any state source, not just scroll position.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate adapters docs](https://pub.dev/packages/flutter_animate#-adapters)
