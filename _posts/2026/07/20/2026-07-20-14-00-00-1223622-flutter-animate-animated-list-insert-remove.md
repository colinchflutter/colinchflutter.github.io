---
layout: post
title: "flutter_animate AnimatedList - Smooth Insert and Remove Animations Without List Jumps"
description: "Combine Flutter AnimatedList with flutter_animate to animate inserted and removed rows while keeping list height, keys, and data updates synchronized."
date: 2026-07-20
tags: [flutter_animate, animation, ListView, performance, Flutter]
comments: true
share: true
---

![Flutter AnimatedList with flutter_animate insert and remove motion](https://colinchflutter.github.io/assets/images/flutter-animate-animated-list-insert-remove.png)

*The useful boundary is visible here: AnimatedList changes the layout, while flutter_animate adds visual emphasis to the card entering or leaving.*

The reliable way to animate `AnimatedList` with `flutter_animate` is to split the job. Let `AnimatedList` control the row's size and position, then use `flutter_animate` for opacity and translation. If both systems animate height, a fast insert followed by a delete makes the list jump or leaves a blank gap.

## Keep data and the list state in the same order

The most common bug is removing the model first and then trying to read it inside `removeItem`. The removed row is built after the data mutation, so the old object is already gone. Save it before changing `_items`.

| Responsibility | Owner | Typical effect |
|---|---|---|
| Row height during insert/remove | `AnimatedList` | `SizeTransition` |
| Card opacity and offset | `flutter_animate` | `fadeIn`, `slideX`, `fadeOut` |
| Item identity | `ValueKey` | Stable state during rebuilds |
| Source of truth | `_items` | Updated exactly once per action |

Here is a compact list that handles both directions:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class TaskList extends StatefulWidget {
  const TaskList({super.key});

  @override
  State<TaskList> createState() => _TaskListState();
}

class _TaskListState extends State<TaskList> {
  final _listKey = GlobalKey<AnimatedListState>();
  final _items = <String>['Write tests', 'Review pull request'];

  void _addTask() {
    final index = _items.length;
    final task = 'Task ${index + 1}';
    _items.insert(index, task);
    _listKey.currentState?.insertItem(
      index,
      duration: 360.ms,
    );
  }

  void _removeTask(int index) {
    final removed = _items[index]; // Capture before removing it.
    _items.removeAt(index);
    _listKey.currentState?.removeItem(
      index,
      (context, animation) => SizeTransition(
        sizeFactor: animation,
        child: _taskTile(removed)
            .animate()
            .fadeOut(duration: 220.ms)
            .slideX(begin: 0, end: -0.08, duration: 220.ms),
      ),
      duration: 360.ms,
    );
  }

  Widget _taskTile(String task) => Card(
        child: ListTile(
          key: ValueKey(task),
          title: Text(task),
          trailing: IconButton(
            icon: const Icon(Icons.close),
            onPressed: () => _removeTask(_items.indexOf(task)),
          ),
        ),
      );

  @override
  Widget build(BuildContext context) => Scaffold(
        floatingActionButton: FloatingActionButton(
          onPressed: _addTask,
          child: const Icon(Icons.add),
        ),
        body: AnimatedList(
          key: _listKey,
          initialItemCount: _items.length,
          padding: const EdgeInsets.all(16),
          itemBuilder: (context, index, animation) => SizeTransition(
            sizeFactor: animation,
            child: KeyedSubtree(
              key: ValueKey(_items[index]),
              child: _taskTile(_items[index])
                  .animate()
                  .fadeIn(duration: 260.ms)
                  .slideX(begin: 0.08, end: 0, duration: 260.ms),
            ),
          ),
        ),
      );
}
```

The insert path updates `_items` before calling `insertItem`, because `itemBuilder` reads the new index immediately. The remove path does the reverse visually: capture the old value, update the list, and give `removeItem` a builder for that captured row.

## Why the two animations should not share the same job

`SizeTransition` receives the animation that `AnimatedList` manages. It shrinks the layout slot, so the rows below move naturally. The `fadeOut` and `slideX` chain only changes pixels inside that slot. On insertion, the same arrangement prevents a scale effect from changing the row's measured height while the list is expanding.

There is one practical edge case in the sample: using the displayed text as a key is fine only while task names are unique. A real model should expose an immutable `id`; otherwise removing one duplicate label can target the wrong row through `indexOf`.

## Short checklist

- Capture the removed model before `_items.removeAt`.
- Call `insertItem` after inserting the model, and `removeItem` after removing it.
- Give layout motion to `SizeTransition`, not to a second height-changing effect.
- Use stable IDs for keys and callbacks when rows can reorder.
- Test rapid add/remove taps; race conditions show up there first.

`AnimatedList` owns structure, and `flutter_animate` owns attention. Keeping that contract small makes dynamic lists feel deliberate instead of animated twice.
