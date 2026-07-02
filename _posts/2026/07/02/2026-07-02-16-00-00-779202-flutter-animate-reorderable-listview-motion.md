---
layout: post
title: "flutter_animate ReorderableListView Motion - polish drag reordering without broken state"
description: "Learn how to use flutter_animate with ReorderableListView, keeping drag feedback polished while avoiding unwanted row replay."
date: 2026-07-02
tags: [flutter_animate, ListView, animation, performance]
comments: true
share: true
---

![Flutter reorderable list drag motion](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

The best `flutter_animate` setup for `ReorderableListView` is not to animate every row on every reorder. Animate the dragged feedback and small state changes, but keep the actual row identity stable. That gives the list a polished feel without making items flash, replay, or lose local state after a drag.

The bug I hit was simple: I wrapped each row with `.animate().fadeIn().moveY()` and used the index as the key. It looked fine until I dragged the second row below the fifth. Flutter reused slots, `flutter_animate` saw a changed subtree, and several rows replayed like they had just loaded. In a settings screen, that kind of motion feels like data changed, not like a reorder happened.

## Keep row keys tied to data

`ReorderableListView` needs stable keys. The key should come from the item id, not from the current index. The animation can still use the index for tiny visual variation, but the widget identity must follow the data object.

Here is the basic structure I use for a reorderable task list:

```dart
class TaskItem {
  const TaskItem({required this.id, required this.title});

  final String id;
  final String title;
}

class AnimatedReorderList extends StatefulWidget {
  const AnimatedReorderList({super.key});

  @override
  State<AnimatedReorderList> createState() => _AnimatedReorderListState();
}

class _AnimatedReorderListState extends State<AnimatedReorderList> {
  final tasks = <TaskItem>[
    const TaskItem(id: 'inbox', title: 'Clean inbox'),
    const TaskItem(id: 'build', title: 'Ship profile screen'),
    const TaskItem(id: 'qa', title: 'Check animation jank'),
  ];

  void _reorder(int oldIndex, int newIndex) {
    setState(() {
      if (newIndex > oldIndex) newIndex -= 1;
      final moved = tasks.removeAt(oldIndex);
      tasks.insert(newIndex, moved);
    });
  }

  @override
  Widget build(BuildContext context) {
    return ReorderableListView.builder(
      itemCount: tasks.length,
      onReorder: _reorder,
      proxyDecorator: _dragProxy,
      itemBuilder: (context, index) {
        final task = tasks[index];

        return _TaskRow(
          key: ValueKey(task.id),
          task: task,
        );
      },
    );
  }
}
```

## Animate the drag proxy

The clean place for `flutter_animate` is `proxyDecorator`. That widget is only used while the item is being dragged, so the animation describes the drag state instead of replaying the whole list.

This proxy lifts the row slightly and adds a soft shadow while the user drags:

```dart
Widget _dragProxy(
  Widget child,
  int index,
  Animation<double> animation,
) {
  return AnimatedBuilder(
    animation: animation,
    builder: (context, _) {
      final t = Curves.easeOut.transform(animation.value);

      return Material(
        elevation: 2 + (10 * t),
        borderRadius: BorderRadius.circular(12),
        child: child
            .animate(target: t)
            .scale(
              begin: const Offset(1, 1),
              end: const Offset(1.03, 1.03),
              duration: 120.ms,
            )
            .fade(begin: 0.92, end: 1),
      );
    },
  );
}
```

I keep this short on purpose. Reordering is a direct manipulation gesture, so slow decorative motion fights the finger. A tiny scale and shadow change is enough to show that the row is floating.

## Avoid replaying settled rows

The row itself should usually be boring. If it needs an entry animation, apply it only when the item is created, not every time the list order changes. One practical pattern is to pass a flag from the creation flow and remove it after the first frame, but most reorderable settings screens do not need that.

```dart
class _TaskRow extends StatelessWidget {
  const _TaskRow({
    super.key,
    required this.task,
  });

  final TaskItem task;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(task.title),
      leading: const Icon(Icons.drag_handle),
      trailing: const Icon(Icons.chevron_right),
    );
  }
}
```

If you want feedback after a successful reorder, animate a small handle color, check icon, or status chip. Do not fade the whole row out and back in. The data did not disappear; only its position changed.

## Watch the edge cases

Index keys are the main footgun. They make animation bugs look random because everything works until the first move. Heavy shadows are the second one. A dragged proxy with blur, opacity, and scale can trigger extra compositing work, especially in a long list. Test it in profile mode on a physical Android device before shipping it broadly.

Nested scroll views are also awkward. If the reorderable list is inside another scrollable, animation might be blamed for gesture conflicts it did not cause. Fix the scroll layout first, then add motion.

## Short checklist

- Use `ValueKey(item.id)`, not `ValueKey(index)`.
- Put drag lift motion in `proxyDecorator`.
- Keep settled rows stable during reorder.
- Animate small feedback, not the whole list.
- Profile heavy proxy effects before using them in production.

`flutter_animate` works well with `ReorderableListView` when it is treated as feedback for the gesture. The list order should feel solid, the dragged row should feel lifted, and nothing else should pretend it just entered the screen.
