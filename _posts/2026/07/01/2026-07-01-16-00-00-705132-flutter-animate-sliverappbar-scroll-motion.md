---
layout: post
title: "flutter_animate SliverAppBar Motion — Collapsing Headers That Track Scroll"
description: "Build a collapsing SliverAppBar animation in Flutter with flutter_animate, ScrollAdapter, and scroll-synced title, image, and action effects."
date: 2026-07-01
tags: [flutter_animate, animation, performance, ListView]
comments: true
share: true
---

![Flutter mobile app with collapsing image header and scroll-synced toolbar](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

Short answer: use `SliverAppBar` for the layout, but let `flutter_animate` own the visual transitions. The clean pattern is a single `ScrollController`, a few `ScrollAdapter` instances, and effects that finish before the toolbar becomes pinned.

I used to put this logic inside `FlexibleSpaceBar` with opacity calculations in a `LayoutBuilder`. It works, but it gets ugly fast. The title fades at one threshold, the hero image scales at another, the action button needs a tiny slide, and suddenly the header has five hand-written interpolation blocks. `flutter_animate` keeps those transitions readable.

## The setup

This pattern assumes a pinned `SliverAppBar` with an expanded image area. The scroll range between `0` and `220` pixels becomes the animation timeline.

```dart
class ProductPage extends StatefulWidget {
  const ProductPage({super.key});

  @override
  State<ProductPage> createState() => _ProductPageState();
}

class _ProductPageState extends State<ProductPage> {
  final _scroll = ScrollController();

  @override
  void dispose() {
    _scroll.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      controller: _scroll,
      slivers: [
        SliverAppBar(
          pinned: true,
          expandedHeight: 280,
          title: _CollapsedTitle(scroll: _scroll),
          flexibleSpace: FlexibleSpaceBar(
            background: _HeaderImage(scroll: _scroll),
          ),
          actions: [
            _SaveButton(scroll: _scroll),
          ],
        ),
        SliverList.builder(
          itemCount: 24,
          itemBuilder: (_, index) => ListTile(
            title: Text('Detail row $index'),
          ),
        ),
      ],
    );
  }
}
```

The important part is ownership. The `CustomScrollView`, header widgets, and adapters all live under the same state object that owns `_scroll`. That avoids the disposed-controller bug I hit when a header child outlived the parent scroll view.

## Fade the collapsed title in late

The toolbar title should not appear while the expanded image still has room. Make it fade in near the end of the collapse.

```dart
class _CollapsedTitle extends StatelessWidget {
  const _CollapsedTitle({required this.scroll});

  final ScrollController scroll;

  @override
  Widget build(BuildContext context) {
    return const Text('Trail Pack')
        .animate(
          adapter: ScrollAdapter(scroll, begin: 150, end: 220),
        )
        .fadeIn(duration: 1.ms)
        .slideY(begin: .25, end: 0, duration: 1.ms);
  }
}
```

The `1.ms` duration looks strange, but with `ScrollAdapter` the scroll offset drives progress. The duration is just required by the effect API; the adapter decides where the animation sits.

## Scale the image away from the toolbar

For the background, keep the effect subtle. A huge zoom makes the app feel like a demo instead of a product screen.

```dart
class _HeaderImage extends StatelessWidget {
  const _HeaderImage({required this.scroll});

  final ScrollController scroll;

  @override
  Widget build(BuildContext context) {
    return Image.network(
      'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?w=1200&q=80',
      fit: BoxFit.cover,
    )
        .animate(
          adapter: ScrollAdapter(scroll, begin: 0, end: 220),
        )
        .scale(begin: const Offset(1, 1), end: const Offset(1.08, 1.08))
        .fadeOut(begin: 0, end: .35);
  }
}
```

This is related to the earlier `ScrollAdapter` pattern, but the Sliver version has a tighter constraint: the animation must complete before the app bar hits its pinned height. If it continues after that, the toolbar feels like it is still moving while the layout has already stopped.

## Keep actions readable

Header action buttons are easy to overdo. I usually fade them slightly darker during collapse, then stop.

```dart
class _SaveButton extends StatelessWidget {
  const _SaveButton({required this.scroll});

  final ScrollController scroll;

  @override
  Widget build(BuildContext context) {
    return IconButton(
      icon: const Icon(Icons.bookmark_border),
      onPressed: () {},
    )
        .animate(
          adapter: ScrollAdapter(scroll, begin: 40, end: 180),
        )
        .slideX(begin: .2, end: 0)
        .fadeIn(begin: .4, end: 1);
  }
}
```

The mistake is animating everything at once. Image scale can start immediately, title fade should start late, and actions should settle somewhere in the middle. Those staggered thresholds make the header feel intentional without adding manual listener code.

## Things to watch

Do not rebuild the whole `CustomScrollView` on every scroll tick. `ScrollAdapter` listens to the controller for you, so there is no reason to call `setState` from `_scroll.addListener`.

Do not use massive images in the flexible space. The animation can be smooth and still drop frames if the header image is oversized. Resize server-side or request a width close to what the device actually needs.

Also test the pinned boundary on small phones. A title that looks perfect on a tall simulator can collide with action buttons when text scale is high. Keep the collapsed title short, or move the longer label into the body.

## Quick recap

`SliverAppBar` should handle the collapsing layout. `flutter_animate` should handle the motion layered on top of it. Use one `ScrollController`, give each header piece its own `ScrollAdapter` range, and finish the effects before the pinned toolbar takes over. That keeps the code readable and the scroll interaction predictable.

Related pattern: [driving flutter_animate from scroll offset]({% post_url 2026-06-25-10-00-00-408852-flutter-animate-scroll-adapter %}).
