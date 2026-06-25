---
layout: post
title: "flutter_animate staggered list animations — one line per item, no controllers"
description: "flutter_animate makes staggered list animations trivial: multiply the index by a delay, chain your effects, done. No AnimationControllers, no Tickers, no managing 20 separate animation states."
date: 2026-06-24
tags: [flutter_animate, animation, ListView, Flutter, Dart]
comments: true
share: true
---
![Flutter list animation with staggered fade-in effect](https://images.unsplash.com/photo-1611532736597-de2d4265fba3?w=800&q=80)

Staggered list animations look impressive and feel polished. And before `flutter_animate`, they were genuinely annoying to implement — you'd manage an `AnimationController` per item, or use a single controller with `Interval` curves for each one, or reach for `flutter_staggered_animations` just to avoid writing the boilerplate. With `flutter_animate` 4.x, the whole thing collapses to one line per item.

## The problem with vanilla staggered animations

Here's what a staggered 5-item list looks like without any helper package:

```dart
class _MyListState extends State<MyList> with TickerProviderStateMixin {
  late final List<AnimationController> _controllers;
  late final List<Animation<double>> _animations;

  @override
  void initState() {
    super.initState();
    _controllers = List.generate(5, (i) => AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 400),
    ));
    _animations = _controllers.map((c) =>
      CurvedAnimation(parent: c, curve: Curves.easeOut)).toList();

    for (var i = 0; i < _controllers.length; i++) {
      Future.delayed(Duration(milliseconds: i * 80), () {
        if (mounted) _controllers[i].forward();
      });
    }
  }

  @override
  void dispose() {
    for (final c in _controllers) c.dispose();
    super.dispose();
  }
  // ...
}
```

That's 20+ lines before a single widget fades in. For a `ListView.builder` with dynamic item count, it gets worse — you can't pre-create controllers because you don't know the count at build time.

## The flutter_animate version

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(items[index]),
    )
    .animate(delay: Duration(milliseconds: index * 80))
    .fadeIn(duration: 300.ms)
    .slideY(begin: 0.2, end: 0, duration: 300.ms, curve: Curves.easeOut);
  },
);
```

That's the whole thing. Each item delays itself by `index * 80ms`, then fades in and slides up simultaneously. No `StatefulWidget`, no `dispose`, no `TickerProviderStateMixin`.

## How the delay actually works

The `delay` parameter on `.animate()` is not a sleep — it holds the widget at its initial state (opacity 0 if you have `fadeIn`, original position if you have `slideY`) and starts the animation chain after the delay elapses. The widget is rendered immediately; only the animation starts late.

This matters because `ListView.builder` builds items lazily. Item 10 doesn't exist in the widget tree until you scroll to it. When it appears, its `delay` clock starts from zero — so a 10th item with `delay: 800.ms` will wait 800ms from the moment it's *built*, not from the moment the list was first rendered.

For most use cases, that's exactly what you want: items that appear mid-scroll still animate in with a slight delay feel. If you want all 10 items to animate strictly in sequence only on first load, you'll need a different approach (covered below).

## Realistic example: a card feed

```dart
class AnimatedFeed extends StatelessWidget {
  final List<FeedItem> items;
  const AnimatedFeed({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      padding: const EdgeInsets.all(16),
      itemCount: items.length,
      itemBuilder: (context, index) {
        return Padding(
          padding: const EdgeInsets.only(bottom: 12),
          child: _FeedCard(item: items[index]),
        )
        .animate(delay: Duration(milliseconds: index * 60))
        .fadeIn(duration: 350.ms)
        .slideY(
          begin: 0.15,
          end: 0,
          duration: 350.ms,
          curve: Curves.easeOutCubic,
        );
      },
    );
  }
}
```

60ms between items is a good starting point. Go below 40ms and the stagger becomes invisible. Above 120ms and users start to feel like the list is loading slowly.

![Staggered card list animation sequence illustration](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## First-load only: capping the delay

The lazy-build behavior is usually fine, but it means items scrolled to late get the same delay treatment as items on initial load. If you want the stagger only on the first N items (say, the ones visible above the fold), cap the delay:

```dart
final delay = Duration(milliseconds: index.clamp(0, 8) * 70);

return YourCard(item: items[index])
  .animate(delay: delay)
  .fadeIn()
  .slideY(begin: 0.1);
```

Items beyond index 8 all get the same `8 * 70 = 560ms` delay, so they don't animate in with any stagger. Adjust the cap to however many items fit on screen.

## Combining effects: shimmer then reveal

Here's a pattern I've used for loading states — shimmer while fetching, then animate the real content in:

```dart
// While loading: shimmer effect
if (isLoading) {
  return SkeletonCard()
    .animate(onPlay: (c) => c.repeat())
    .shimmer(duration: 1200.ms, color: Colors.white24);
}

// Loaded: staggered reveal
return RealCard(item: items[index])
  .animate(delay: Duration(milliseconds: index * 70))
  .fadeIn(duration: 400.ms)
  .scale(begin: const Offset(0.95, 0.95), end: Offset.zero, duration: 400.ms);
```

The `scale` here is a subtle grow-in effect. `begin: Offset(0.95, 0.95)` means start at 95% size — enough to feel alive without being cartoonish.

## Watch out: keys in ListView.builder

If your list items can reorder or the list rebuilds frequently, add keys:

```dart
return YourCard(key: ValueKey(items[index].id), item: items[index])
  .animate(delay: Duration(milliseconds: index * 70))
  .fadeIn();
```

Without a key, Flutter may reuse the widget slot and the animation won't replay when an item moves. With the key, a moved or newly inserted item gets a fresh `Animate` widget and replays from scratch. This is the right behavior for inserts (new items pop in with the animation), but might be surprising for reorders.

## Performance note

Each `.animate()` creates an `AnimationController` internally. For a list with hundreds of items, you're creating hundreds of controllers simultaneously — `ListView.builder` limits the active ones to what's on screen, but as you scroll, old controllers are disposed and new ones created. I haven't hit any jank with this pattern up to ~500 items on a mid-range Android (Pixel 6a), but if you're seeing frame drops, profile with `flutter run --profile` before optimizing.

---

Next up: driving flutter_animate from state changes — using `animate(target:)` to build toggle animations and step-by-step reveals without touching a controller directly.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter staggered animations cookbook](https://docs.flutter.dev/cookbook/effects/staggered-menu-animation)
