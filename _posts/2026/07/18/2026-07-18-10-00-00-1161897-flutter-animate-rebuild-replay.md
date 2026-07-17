---
layout: post
title: "flutter_animate Rebuilds - Stop Flutter Animations from Replaying on Every State Change"
description: "Fix flutter_animate animations that replay during Flutter rebuilds by separating animation identity, state targets, and list item keys with practical examples."
date: 2026-07-18
tags: [flutter_animate, animation, state_management, performance, Flutter]
comments: true
share: true
---

![Flutter app dashboard with animated cards and changing state](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

The most confusing `flutter_animate` bug is not a crash. It is a card that fades in again whenever a counter changes, a list item that slides from the side after every rebuild, or a loading animation that restarts when an unrelated provider emits. The fix is usually not a new curve. It is giving the animation a stable identity and choosing the right trigger.

## Why a rebuild can replay an animation

Flutter rebuilds widgets freely. A rebuild is not supposed to mean “create a new visual object,” but an animation can restart when the `Animate` widget receives a new configuration or is replaced by a new element.

These three situations look similar but need different fixes:

| Situation | What actually changes | Better fix |
| --- | --- | --- |
| Parent calls `setState` | The widget rebuilds, but the item is still the same | Keep the animation configuration stable |
| An item is inserted or removed | The element identity changes | Use a stable `Key` and an explicit target |
| The visual state changes | The animation should run again | Drive it with `target` |

For example, this is easy to write in a dashboard:

```dart
Widget build(BuildContext context) {
  return Text('$_count')
      .animate()
      .fadeIn(duration: 350.ms)
      .moveY(begin: 12, end: 0);
}
```

If the `Text` is rebuilt as part of a larger state update, the result may look like the number is entering the screen repeatedly. The problem becomes more obvious when the animation is inside a conditional branch or a builder that creates a new subtree after every state notification.

## Separate entrance animation from state animation

An entrance effect should normally run once when the element enters the tree. A state effect should run when a value changes. Combining both in one unconditional chain makes the intent unclear.

For a permanent dashboard card, keep the entrance animation on a stable child and animate the changing value separately:

```dart
class ScoreCard extends StatelessWidget {
  const ScoreCard({required this.score, super.key});

  final int score;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Row(
          children: [
            const Icon(Icons.insights),
            const SizedBox(width: 12),
            Text('$score')
                .animate(key: ValueKey(score))
                .fadeIn(duration: 180.ms)
                .scale(begin: const Offset(.92, .92), end: const Offset(1, 1)),
          ],
        ),
      ),
    ).animate().fadeIn(duration: 350.ms).moveY(begin: 16, end: 0);
  }
}
```

The card's entrance animation is not tied to `score`. The number has a new key when the score changes, so only that small element gets a new identity. This is useful for a count-up-like “value changed” cue, but it is not the same as preserving an animation through a rebuild.

If the number should not be replaced at all, use `target` instead:

```dart
Text('$score')
    .animate(target: score.toDouble())
    .tint(
      color: score > previousScore ? Colors.green : Colors.orange,
      duration: 220.ms,
    )
    .scale(
      begin: const Offset(1, 1),
      end: const Offset(1.08, 1.08),
      curve: Curves.easeOut,
    );
```

The target is meaningful only when it changes. A parent rebuild with the same target does not express a new animation event.

## The key trap in dynamic lists

List animations become unreliable when items use their array index as identity. Imagine a list containing `A`, `B`, and `C`. If `A` is removed, Flutter may reuse the element that used to display `B` for the new first row. A row-level entrance effect can then appear to replay on the wrong item.

Use the domain identifier as the key:

```dart
ListView.builder(
  itemCount: tasks.length,
  itemBuilder: (context, index) {
    final task = tasks[index];

    return TaskTile(
      key: ValueKey(task.id),
      task: task,
    );
  },
);
```

Then keep the effect inside `TaskTile`:

```dart
class TaskTile extends StatelessWidget {
  const TaskTile({required this.task, super.key});

  final Task task;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(task.title),
      trailing: Icon(task.done ? Icons.check : Icons.circle_outlined),
    ).animate().fadeIn(duration: 250.ms).slideX(begin: .08, end: 0);
  }
}
```

A stable key does not magically make every list animation perfect. It tells Flutter which logical task owns the existing element. That distinction matters when a state-management layer rebuilds the list after sorting, filtering, or pagination.

## Preventing an effect from replaying after a provider update

When a provider or `ValueListenableBuilder` updates, try to rebuild the smallest subtree possible. The animation does not need to be moved into a stateful widget just because the parent rebuilds.

This pattern keeps the animated shell outside the frequently changing builder:

```dart
class ProfileHeader extends StatelessWidget {
  const ProfileHeader({required this.name, required this.points, super.key});

  final String name;
  final int points;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(name),
        PointsValue(points: points),
      ],
    ).animate().fadeIn(duration: 300.ms);
  }
}
```

`ProfileHeader` may rebuild when `points` changes, but its element remains the same. If the entrance effect still visibly restarts, inspect the parent first: changing a `Key`, switching between different widget types, or conditionally removing the subtree is more significant than the rebuild itself.

## A practical debugging checklist

When an animation unexpectedly replays, I check the tree in this order:

1. Is the animated widget being removed and inserted, or merely rebuilt?
2. Did a parent key change?
3. Is a list using `ValueKey(index)` instead of a stable model ID?
4. Is `target` changing on every build because it is derived from a new object?
5. Is an `Animate` chain being created in two mutually exclusive branches?

For a quick test, add a `debugPrint` in the widget's `build` method and another in a stateful child's `initState`. A build log proves that the widget rebuilt; `initState` proves that the element was recreated. Those are different events, and treating them as the same leads to unnecessary controller code.

The useful rule is simple: use an unconditional `.animate()` chain for a stable entrance, `target` for a deliberate state transition, and stable keys for collections. Once those responsibilities are separated, `flutter_animate` can replay when the UI actually changed instead of whenever Flutter did its normal rebuild work.

![Stable keys keep animated list rows attached to the correct data](https://images.unsplash.com/photo-1556075798-4825dfaaf498?w=1200&q=80)

The important detail is not the animation curve; it is whether the same logical widget still owns the animation after the state update.

