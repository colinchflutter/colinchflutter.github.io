---
layout: post
title: "flutter_animate SliverList Motion - scroll polish without replaying every row"
description: "Use flutter_animate with SliverList rows without replaying animations on every scroll rebuild, with practical keys, intervals, and motion limits."
date: 2026-07-03
tags: [flutter_animate, animation, performance, scrolling, Flutter]
comments: true
share: true
---
![flutter_animate SliverList scroll motion](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=800&q=80)

Look at this image as a reminder that scrolling UI is already moving; row animation should support the list, not compete with it.

`flutter_animate` works well with `SliverList`, but the trap is replaying row entrance motion every time Flutter rebuilds visible children. The list feels polished for the first three seconds, then annoying as rows fade in again after a filter change, tab switch, or parent rebuild. For production lists, I treat sliver motion as a one-time hint, not a permanent decoration.

The first mistake I made was wrapping every row with the same `.animate().fadeIn().slideY()`. It looked fine in a short demo. In a 120-row activity log, fast scrolling caused rows to shimmer as if the data was unstable. The bug was not `flutter_animate`; it was my lifecycle assumption.

| List case | Motion choice | Why |
|---|---|---|
| First page load | short fade + 8px slide | Helps users see new content arrive |
| Infinite append | animate only new items | Avoids old rows replaying |
| Filter change | animate group container | Rows should not all pop again |
| Pull refresh | use refresh indicator feedback | Scroll position is already context |

The safer pattern is to keep stable keys and limit the delay.

```dart
SliverList.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    final item = items[index];

    return ActivityRow(
      key: ValueKey(item.id),
      item: item,
    )
        .animate(delay: (index.clamp(0, 8) * 35).ms)
        .fadeIn(duration: 180.ms)
        .slideY(begin: .05, end: 0, curve: Curves.easeOutCubic);
  },
)
```

The `clamp(0, 8)` matters. Without it, row 80 waits almost three seconds before appearing, which feels broken rather than elegant. I also avoid large slide distances in slivers. A `begin: .2` slide can overlap neighboring rows during quick scrolls, especially when row heights vary.

For appended data, I prefer marking new IDs instead of animating the whole list.

```dart
final isNew = newlyInsertedIds.contains(item.id);

return ActivityRow(key: ValueKey(item.id), item: item)
    .animate(target: isNew ? 1 : 0)
    .fadeIn(duration: 160.ms)
    .scaleXY(begin: .98, end: 1);
```

The edge case is pagination. If the API returns the same item with a changed timestamp, a weak key like `ValueKey(index)` makes Flutter treat it as a different row after sorting. Use the domain ID. If the backend has no stable ID, create one from fields that actually identify the record, not from its current position.

A short checklist:

- Keep row keys stable across sorting, filtering, and pagination.
- Cap stagger delays so deep rows never wait visibly.
- Animate appended rows, not every existing row.
- Use small movement; slivers already have scroll motion.
- Test with 100+ rows, not only the pretty five-row sample.

Motion in a `SliverList` should make state changes easier to read. Once it starts making old data look new, the animation is working against the user.
