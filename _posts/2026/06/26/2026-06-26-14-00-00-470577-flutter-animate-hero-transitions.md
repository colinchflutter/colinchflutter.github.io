---
layout: post
title: "flutter_animate + Hero Transitions — Entrance Animations That Hand Off to Flutter's Hero Widget"
description: "How to coordinate flutter_animate entrance effects with Flutter's Hero widget. Covers animating destination content after Hero lands, avoiding double-animation conflicts, and a practical card-to-detail pattern."
date: 2026-06-26
tags: [flutter_animate, animation, Flutter, navigation]
comments: true
share: true
---

![Flutter Hero transition card to detail page animation](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

The Hero widget handles the shared-element flight between screens. flutter_animate handles entrance effects. Combining them is mostly straightforward — except for one timing trap that burns everyone the first time.

## The Setup: What Each One Does

Hero owns the widget during the flight. While the Hero is flying, your widget is inside the Hero's overlay, and any `.animate()` you put on it runs *at the same time as the flight*. That's usually not what you want — you want the Hero to land first, *then* your entrance animation plays.

The mistake:

```dart
// Destination page — DON'T do this
Hero(
  tag: 'product-image',
  child: ProductImage()
    .animate()
    .fadeIn(duration: 400.ms),  // fights the hero flight
)
```

The fade competes with the Hero flight. The image simultaneously fades in while flying across the screen — looks wrong on every device.

## Pattern 1: Animate the Destination Content, Not the Hero

The fix is simple: don't animate the Hero widget itself. Animate the *surrounding content* on the destination page, with a small delay to let the Hero finish landing.

Typical Hero flight duration is around 300ms. Add a 50ms buffer:

```dart
class ProductDetailPage extends StatelessWidget {
  const ProductDetailPage({super.key, required this.product});
  final Product product;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Hero itself — no animate()
          Hero(
            tag: 'product-${product.id}',
            child: ProductImage(product: product),
          ),

          // Everything below animates in after the hero lands
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(product.name, style: Theme.of(context).textTheme.headlineMedium),
                const SizedBox(height: 8),
                Text(product.description),
                const SizedBox(height: 16),
                ElevatedButton(
                  onPressed: () {},
                  child: const Text('Add to Cart'),
                ),
              ],
            ),
          )
          .animate(delay: 350.ms)
          .fadeIn(duration: 300.ms)
          .slideY(begin: 0.08, end: 0, duration: 300.ms, curve: Curves.easeOut),
        ],
      ),
    );
  }
}
```

The `delay: 350.ms` is the key. Hero's default flight is roughly 300ms (`kThemeAnimationDuration`). The content below fades in just as the hero settles — feels like a coordinated reveal.

## Pattern 2: Staggered Detail Content

If the destination page has multiple sections, stagger them for a more polished feel:

```dart
Column(
  children: [
    Hero(
      tag: 'card-${item.id}',
      child: ItemCard(item: item),
    ),
    const SizedBox(height: 16),

    Text(item.title, style: Theme.of(context).textTheme.titleLarge)
      .animate(delay: 320.ms)
      .fadeIn(duration: 250.ms)
      .slideX(begin: -0.05, end: 0),

    const SizedBox(height: 8),

    Text(item.body)
      .animate(delay: 400.ms)
      .fadeIn(duration: 250.ms),

    const SizedBox(height: 16),

    ActionButtonRow(item: item)
      .animate(delay: 480.ms)
      .fadeIn(duration: 250.ms)
      .slideY(begin: 0.06, end: 0),
  ],
)
```

80ms gaps between elements. Fine enough to feel staggered, fast enough not to feel slow.

![Flutter staggered detail page entrance animation sequence](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

## Pattern 3: Animating the Exit — Back Navigation

Here's something people miss: flutter_animate plays entrance animations on push. On pop (back), it doesn't automatically reverse them — the route just disappears.

If you want the content to animate out before the Hero flies back, use `autoplay: false` with a controller and trigger it manually before `Navigator.pop`:

```dart
class _ProductDetailState extends State<ProductDetailPage> {
  late AnimationController _controller;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        leading: BackButton(
          onPressed: () async {
            // Reverse the entrance animation, then pop
            await _controller.reverse();
            if (mounted) Navigator.pop(context);
          },
        ),
      ),
      body: Column(
        children: [
          Hero(tag: 'product-${widget.product.id}', child: ProductImage(product: widget.product)),
          ProductContent(product: widget.product)
            .animate(
              autoPlay: false,
              onInit: (controller) {
                _controller = controller;
                controller.forward();  // trigger the entrance
              },
            )
            .fadeIn(delay: 350.ms, duration: 300.ms)
            .slideY(begin: 0.08, end: 0, delay: 350.ms, duration: 300.ms),
        ],
      ),
    );
  }
}
```

`controller.reverse()` plays the animation backward — slide down and fade out — then the pop happens, and the Hero flies back. Clean.

Worth knowing: `await _controller.reverse()` completes when the animation finishes. On slow devices, this might feel like a hesitation. Keep the reverse duration short — 200ms max.

## The Hero Duration Mismatch Problem

This one took me a while to track down. If you customize the Hero flight duration using `createRectTween` or wrap routes in a custom `PageTransitionsTheme`, the default 350ms delay on your content animations won't match anymore.

Short answer: hardcode the delay to match your actual Hero duration.

```dart
// If your hero flight is 250ms:
.animate(delay: 280.ms)  // 250ms flight + 30ms buffer

// If your hero flight is 400ms:
.animate(delay: 430.ms)
```

There's no API to query the Hero duration from the destination widget directly. If you change the transition duration, update the delays manually.

## What Not to Do

**Don't animate scale or position on the Hero child.** Hero controls exactly where and how big the widget is during flight. If you add `.scale()` or `.slideY()` directly to the Hero child, you get competing transforms — the hero tries to fly it to position X while flutter_animate tries to move it to position Y. The result is jitter or the widget flying to the wrong place.

**Don't use `.animate()` with `key:` on the Hero child.** flutter_animate uses keys internally to restart animations on rebuild. A key conflict between your animation and the Hero widget's state causes the animation to restart mid-flight.

If you genuinely need to animate the hero child itself, do it *after* the flight completes — use the `flightShuttleBuilder` callback to show a static version during flight, then animate on arrival.

---

Next up: `flutter_animate` with `ValueNotifier` and `Animate.target` — driving animation state directly from business logic without `StatefulWidget`.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [Hero widget Flutter docs](https://docs.flutter.dev/ui/animations/hero-animations)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
