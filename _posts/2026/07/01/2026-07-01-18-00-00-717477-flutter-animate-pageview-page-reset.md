---
layout: post
title: "flutter_animate PageView - replaying page animations without controller clutter"
description: "Learn how to replay flutter_animate effects inside PageView pages using keys, target values, and guarded page change state."
date: 2026-07-01
tags: [flutter_animate, PageView, animation, state_management]
comments: true
share: true
---
![Flutter PageView onboarding screen with animated cards](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

PageView animations should usually replay when a page becomes active, not every time Flutter rebuilds a child. The clean pattern is to keep the `PageController` boring, track the selected index, and let each page decide whether its `flutter_animate` chain should run.

The mistake I made the first time was putting `.animate()` directly on every card and expecting PageView to handle the timing. It looked fine on a fast swipe, then broke during slow drags: the next page started animating while it was still half off-screen. It felt like the UI was racing the gesture.

## The rebuild problem

`PageView.builder` is lazy, but nearby pages can stay alive. That is good for performance and bad for one-shot entrance effects. If page 1 already built once, returning to it may not replay the animation because the widget tree did not get recreated in the way you expected.

Use the page index as explicit animation state:

```dart
class FeaturePager extends StatefulWidget {
  const FeaturePager({super.key});

  @override
  State<FeaturePager> createState() => _FeaturePagerState();
}

class _FeaturePagerState extends State<FeaturePager> {
  final _controller = PageController();
  int _activePage = 0;

  final pages = const [
    ('Fast checkout', 'Save payment details safely.'),
    ('Smart alerts', 'Only show notifications that matter.'),
    ('Offline mode', 'Keep key screens usable without network.'),
  ];

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return PageView.builder(
      controller: _controller,
      itemCount: pages.length,
      onPageChanged: (index) => setState(() => _activePage = index),
      itemBuilder: (context, index) {
        final (title, body) = pages[index];

        return FeaturePage(
          key: ValueKey('feature-$index-$_activePage'),
          title: title,
          body: body,
          active: index == _activePage,
        );
      },
    );
  }
}
```

The key includes `_activePage`, so the active page gets a fresh animation subtree when the selected page changes. That is heavy-handed, but it is predictable and works well for onboarding, product tours, and small horizontal carousels.

## Animate only the visible page

The page widget should render normally when inactive. I do not hide it with opacity 0, because users can see neighboring pages during a drag.

```dart
class FeaturePage extends StatelessWidget {
  const FeaturePage({
    super.key,
    required this.title,
    required this.body,
    required this.active,
  });

  final String title;
  final String body;
  final bool active;

  @override
  Widget build(BuildContext context) {
    final content = Padding(
      padding: const EdgeInsets.all(32),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(title, style: Theme.of(context).textTheme.headlineMedium),
          const SizedBox(height: 12),
          Text(body, style: Theme.of(context).textTheme.bodyLarge),
        ],
      ),
    );

    if (!active) return content;

    return content
        .animate()
        .fadeIn(duration: 260.ms, curve: Curves.easeOut)
        .slideY(begin: 0.08, end: 0, duration: 360.ms, curve: Curves.easeOutCubic)
        .then(delay: 80.ms)
        .shimmer(duration: 650.ms, color: Colors.white30);
  }
}
```

This keeps inactive pages stable. The selected page gets the entrance chain, then a short shimmer after the layout has settled. In a real app I would keep the shimmer subtle or remove it entirely for pages with dense text.

## When keys are too much

For heavier pages, rebuilding the whole subtree can be wasteful. In that case use `target` instead of a changing key:

```dart
return content
    .animate(target: active ? 1 : 0)
    .fade(begin: active ? 0 : 1, end: 1, duration: 240.ms)
    .slideX(begin: active ? 0.06 : 0, end: 0, duration: 320.ms);
```

This does not reset internal widget state. That is better for forms, video thumbnails, or pages that hold scroll position. The tradeoff is that reverse transitions can run when a page becomes inactive, so keep the inactive values close to the final layout.

## Practical guardrails

Do not update `_activePage` from `PageController.page` on every scroll tick unless you really need gesture-progress animation. `onPageChanged` fires after the page settles, which is the right moment for replaying entrance effects.

Avoid long delays inside a PageView. A 500ms delay feels broken when the user swipes quickly through three pages. Keep entry motion under 400ms and let the page feel responsive before adding decoration.

If a page contains buttons, make sure the non-animated layout is already tappable and readable. Animation should decorate the state change, not become the only way the content appears.

## Quick takeaways

Use `onPageChanged` as the source of truth for page activation. Use a changing `ValueKey` when small pages should replay from scratch. Use `target` when the page owns expensive state. Keep inactive pages visible during drags, because PageView often shows more than one page while the gesture is in progress.
