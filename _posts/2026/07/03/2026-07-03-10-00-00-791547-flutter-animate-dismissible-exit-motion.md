---
layout: post
title: "flutter_animate Dismissible - smooth exit motion without premature removal"
description: "Learn how to combine flutter_animate with Dismissible so swipe actions feel polished without removing list rows too early."
date: 2026-07-03
tags: [flutter_animate, ListView, animation, state_management]
comments: true
share: true
---

![Flutter swipe to dismiss list item animation](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

The cleanest way to use `flutter_animate` with `Dismissible` is to let `Dismissible` own the swipe gesture, then let your state decide when the row disappears. If the item is removed from the list inside `onDismissed` without any guard, the row can vanish before the visual exit reads as intentional.

The failure I kept seeing was subtle. A task row slid away correctly, but the list compacted instantly and the neighboring rows jumped into place. Adding `.fadeOut()` directly around the row did not fix it, because the widget had already been removed from the data source. The animation was attached to a subtree Flutter no longer had a reason to keep.

## Keep dismissal as a state transition

Treat the swipe as a request, not as the final delete. Store a pending id, render that row in an exit state, and remove it after the animation duration. This keeps the UI honest: the user's gesture starts the action, the animation explains the change, and the list only rebuilds after the row has finished leaving.

Here is the shape I use for a small inbox or task list:

```dart
class AnimatedDismissList extends StatefulWidget {
  const AnimatedDismissList({super.key});

  @override
  State<AnimatedDismissList> createState() => _AnimatedDismissListState();
}

class _AnimatedDismissListState extends State<AnimatedDismissList> {
  final items = <TaskItem>[
    TaskItem('a', 'Review crash report'),
    TaskItem('b', 'Ship settings copy'),
    TaskItem('c', 'Check sync logs'),
  ];

  final exiting = <String>{};

  Future<bool> _requestDismiss(TaskItem item) async {
    setState(() => exiting.add(item.id));

    await Future<void>.delayed(260.ms);

    if (!mounted) return false;
    setState(() {
      items.removeWhere((current) => current.id == item.id);
      exiting.remove(item.id);
    });

    return false;
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        final item = items[index];
        final isExiting = exiting.contains(item.id);

        return Dismissible(
          key: ValueKey(item.id),
          direction: DismissDirection.endToStart,
          confirmDismiss: (_) => _requestDismiss(item),
          background: const ColoredBox(color: Color(0xffd64545)),
          child: TaskTile(item: item)
              .animate(target: isExiting ? 1 : 0)
              .fade(end: 0, duration: 180.ms)
              .slideX(end: -0.08, duration: 220.ms, curve: Curves.easeOutCubic)
              .scaleXY(end: 0.98, duration: 220.ms),
        );
      },
    );
  }
}
```

The important detail is `confirmDismiss`. Returning `false` tells `Dismissible` not to perform its own permanent removal. That looks backward at first, but it gives the screen one reliable source of truth. Your model removes the item after `flutter_animate` has had time to show the exit.

## Avoid index keys

Use a stable id for the `Dismissible` key. An index key works until the first deletion, then Flutter may recycle the wrong row and the animation can appear on a neighboring item. That is especially confusing in lists with checkboxes, text fields, or cached network thumbnails.

I also keep the animation small. A deleted row should not perform a dramatic card trick. A short fade, a tiny horizontal slide, and a slight scale change are enough to make the action legible without slowing down repeated cleanup work.

## Watch undo flows

If the screen supports undo, delay the server delete but remove the row locally after the exit animation. Reinsert it with the same id when the undo action runs. Do not leave the row half transparent while a snackbar timer counts down; that creates a disabled-looking item that still occupies layout space.

For destructive actions, the practical rule is simple: gesture first, visual exit second, data removal third. `flutter_animate` works best here when it explains a state change instead of fighting `Dismissible` for ownership of the row lifecycle.
