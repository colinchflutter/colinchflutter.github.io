---
layout: post
title: "flutter_animate ReorderableListView - Animate Drop Feedback Without Drag Jitter"
description: "Use flutter_animate with Flutter ReorderableListView to highlight the row that moved while keeping drag feedback, keys, and list layout stable."
date: 2026-07-19
tags: [flutter_animate, animation, ListView, performance, Flutter]
comments: true
share: true
---

![Flutter ReorderableListView with a highlighted dropped card](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=1200&q=80)

The cleanest `flutter_animate` pattern for `ReorderableListView` is to animate the row after the drop, not the row while it is being dragged. Flutter already owns the drag proxy, placeholder, gesture arena, and final layout. `flutter_animate` only needs to confirm which item moved. That boundary prevents the list from wobbling when a user drags quickly.

## The problem with animating the whole row

It is tempting to wrap every row in `.animate().scale().fadeIn()`. A reorder then triggers several changes at once: one item becomes a proxy, another gets a gap, and the list changes its order. If the animation also changes size or position, Flutter has to reconcile two motion systems.

The result is usually acceptable with four items and unpleasant with twenty. The dragged card jumps, the row under the pointer changes height, or every item replays when `setState` stores the new order.

| Responsibility | Best owner | Why |
|---|---|---|
| Drag proxy and placeholder | `ReorderableListView` | It understands pointer movement and list geometry |
| Persisting the new order | App state | The order is business data, not visual state |
| Confirming the moved row | `flutter_animate` | A small visual response is enough |
| Insert/remove animation | `AnimatedList` or a diffing layer | Reordering is not insertion |

The useful signal is the item ID that was dropped. Do not use the old or new index as the animation identity. Indexes change during every reorder, while an ID stays attached to the same item.

## A keyed reorderable list

This example keeps the list data separate from the temporary highlight state. The `ValueKey` belongs to the row's data identity, and the animation target belongs to the last moved ID.

```dart
class Task {
  const Task(this.id, this.title);

  final String id;
  final String title;
}

class ReorderableTasks extends StatefulWidget {
  const ReorderableTasks({super.key});

  @override
  State<ReorderableTasks> createState() => _ReorderableTasksState();
}

class _ReorderableTasksState extends State<ReorderableTasks> {
  final tasks = <Task>[
    const Task('plan', 'Plan the release'),
    const Task('test', 'Run widget tests'),
    const Task('review', 'Review the pull request'),
    const Task('ship', 'Publish the build'),
  ];

  String? lastMovedId;

  void _reorder(int oldIndex, int newIndex) {
    if (newIndex > oldIndex) newIndex -= 1;

    setState(() {
      final moved = tasks.removeAt(oldIndex);
      tasks.insert(newIndex, moved);
      lastMovedId = moved.id;
    });
  }

  @override
  Widget build(BuildContext context) {
    return ReorderableListView.builder(
      itemCount: tasks.length,
      onReorder: _reorder,
      itemBuilder: (context, index) {
        final task = tasks[index];
        final isMoved = task.id == lastMovedId;

        return ListTile(
          key: ValueKey(task.id),
          title: Text(task.title),
          trailing: const Icon(Icons.drag_handle),
        ).animate(
          target: isMoved ? 1 : 0,
          effects: const [
            FadeEffect(begin: 0.72, end: 1, duration: 180.ms),
            ScaleEffect(begin: Offset(0.98, 0.98), end: Offset(1, 1)),
          ],
        );
      },
    );
  }
}
```

The important detail is that `lastMovedId` changes only after `onReorder` finishes. During the drag, the row keeps its normal size and the built-in reorder animation remains in charge. After the drop, the moved row gets a short target transition.

## Why the key matters

Without `ValueKey(task.id)`, Flutter can reuse an element by position. After moving item A from position 0 to position 3, the animation state that belonged to position 0 may appear on item B. That is the classic “the wrong row flashed” bug.

The key should identify the record, not the current index:

```dart
// Stable across reordering.
ValueKey(task.id)

// Wrong for data-driven reorderable rows.
ValueKey(index)
```

There is another small decision here: `target` is derived from a boolean, so ordinary parent rebuilds do not restart the effect. If the same row is moved twice, the target must go back to `0` before it can transition to `1` again. A simple way to handle that is to clear the highlight after the short effect.

```dart
void _reorder(int oldIndex, int newIndex) {
  if (newIndex > oldIndex) newIndex -= 1;

  final moved = tasks.removeAt(oldIndex);
  tasks.insert(newIndex, moved);

  setState(() => lastMovedId = moved.id);

  Future<void>.delayed(const Duration(milliseconds: 260), () {
    if (!mounted || lastMovedId != moved.id) return;
    setState(() => lastMovedId = null);
  });
}
```

For production code, I would keep the delayed cleanup in a cancellable timer if reorders can happen repeatedly. The guard prevents an old cleanup callback from clearing the highlight for a newer drop.

## Keep the drag proxy visually quiet

`proxyDecorator` is a good place for a temporary elevation or color change, but it is not a good place for a looping `flutter_animate` chain. The proxy is created and moved every time the pointer changes. An animation that starts there can restart many times during one drag.

```dart
proxyDecorator: (child, index, animation) {
  return AnimatedBuilder(
    animation: animation,
    builder: (context, child) => Material(
      elevation: 4 + animation.value * 6,
      borderRadius: BorderRadius.circular(12),
      child: child,
    ),
    child: child,
  );
},
```

Use `flutter_animate` for the settled state and Flutter's `animation` value for the live drag state. The separation is especially noticeable on a low-end Android device, where repainting several animated rows during a drag can turn a simple reorder into dropped frames.

## Practical checks

- Drag the first item to the last position and verify that only that item is highlighted.
- Drag an item up and down twice before the 260 ms cleanup completes.
- Trigger an unrelated parent rebuild while the list is visible; no row should replay.
- Use real IDs and stable keys when the list is loaded from a database or API.
- Keep row height fixed during the drop cue. Animate color, opacity, or a tiny scale rather than layout dimensions.
- For reduced-motion settings, skip the effect or reduce it to a quick color change.

`ReorderableListView` should own movement, the state layer should own order, and `flutter_animate` should own the small confirmation cue. Once those responsibilities stay separate, reordering feels responsive instead of animated for animation's sake.
