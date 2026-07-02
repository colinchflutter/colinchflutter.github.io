---
layout: post
title: "flutter_animate GridView Filter Motion - animating card grids without replaying everything"
description: "Use flutter_animate with GridView cards, filter changes, and stable keys so Flutter grid animations feel intentional instead of noisy."
date: 2026-07-02
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![Flutter GridView cards with subtle animated filter transitions](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

The clean pattern for animated `GridView` cards is to animate only new or newly visible items, keep keys stable, and avoid replaying the whole grid every time a filter chip changes. `flutter_animate` makes the entry motion easy, but the real work is deciding what should not move.

The mistake I made on a dashboard screen was treating every filter result as a fresh page. I changed the selected category, rebuilt the grid, and each card faded in from below again. It looked polished for about two taps. After that it felt like the UI was hiding the data from the user.

For a card grid, start with stable model IDs and a small entrance effect. Do not key by index. The index changes when sorting or filtering, so Flutter thinks familiar cards are different widgets.

```dart
class MetricCard {
  const MetricCard({
    required this.id,
    required this.title,
    required this.value,
    required this.group,
  });

  final String id;
  final String title;
  final String value;
  final String group;
}
```

Here is the baseline grid. The animation is intentionally short: a tiny vertical slide, fade, and scale. Large card movement inside a grid quickly becomes visual noise.

```dart
GridView.builder(
  padding: const EdgeInsets.all(16),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    mainAxisSpacing: 12,
    crossAxisSpacing: 12,
    childAspectRatio: 1.35,
  ),
  itemCount: visibleCards.length,
  itemBuilder: (context, index) {
    final card = visibleCards[index];

    return DashboardCard(
      key: ValueKey(card.id),
      title: card.title,
      value: card.value,
    )
        .animate()
        .fadeIn(duration: 180.ms, curve: Curves.easeOut)
        .slideY(begin: 0.04, end: 0, duration: 220.ms)
        .scale(begin: const Offset(0.98, 0.98), duration: 220.ms);
  },
)
```

This is fine for initial page load. It is not fine for every filter change, because every rebuilt child replays. The fix is to separate "this card exists" from "this card just entered the current result set".

I usually keep a small set of IDs that were already visible before the filter changed:

```dart
Set<String> _previousVisibleIds = {};
String _selectedGroup = 'all';

void selectGroup(String group) {
  setState(() {
    _previousVisibleIds = visibleCards.map((card) => card.id).toSet();
    _selectedGroup = group;
  });
}
```

Then animate only cards that were not in the previous result. Existing cards stay stable, while newly revealed cards get motion.

```dart
Widget buildGridCard(MetricCard card) {
  final isNewToResult = !_previousVisibleIds.contains(card.id);

  final child = DashboardCard(
    key: ValueKey(card.id),
    title: card.title,
    value: card.value,
  );

  if (!isNewToResult) return child;

  return child
      .animate(key: ValueKey('enter-${card.id}-$_selectedGroup'))
      .fadeIn(duration: 160.ms)
      .slideY(begin: 0.05, end: 0, duration: 220.ms)
      .scale(begin: const Offset(0.97, 0.97), duration: 220.ms);
}
```

The separate animate key matters. The card itself keeps `ValueKey(card.id)`, so Flutter preserves the identity of the content. The animation wrapper gets a key that changes only when a card enters through a specific filter state. That gives you replay control without pretending the data changed.

For a search box, I use an even quieter rule. If the user is typing, skip entry animation entirely. Search results update too quickly for card choreography to read well.

```dart
final shouldAnimateEntries =
    query.isEmpty && !_isDraggingScrollView && visibleCards.length <= 20;
```

That last limit is pragmatic. Animating 60 cards in a grid does not help anyone, and on a low-end Android phone it can cause missed frames if the card has charts, images, or shadows. If the grid is large, animate the first screenful only or just fade the empty/loading state.

One more trap: do not combine this with aggressive `AnimatedSwitcher` wrapping the entire `GridView`. That fades the full scrollable area while each card also animates. The result is double motion and sometimes a lost scroll position. If the filter changes the dataset, keep the grid widget stable and animate the children that deserve attention.

Use `flutter_animate` on `GridView` when it clarifies what just appeared. Stable IDs prevent accidental replays, a previous-visible set keeps existing cards calm, and short fade/slide/scale effects are enough. The grid should feel responsive first and animated second.
