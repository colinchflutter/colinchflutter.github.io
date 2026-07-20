---
layout: post
title: "flutter_animate DragTarget - Animate Flutter Drop Zones Without Drag Jitter"
description: "Use flutter_animate with Flutter DragTarget to highlight valid drop zones while keeping drag feedback, acceptance, and layout state predictable."
date: 2026-07-21
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

![Flutter DragTarget with flutter_animate drop zone feedback](https://colinchflutter.github.io/assets/images/flutter-animate-dragtarget-drop-zone-feedback.png)

*The important visual cue is the target zone, not a second animation fighting the card being dragged.*

The cleanest way to animate a Flutter `DragTarget` with `flutter_animate` is to keep the drag gesture in `Draggable`, keep acceptance in `DragTarget`, and animate only the target's visual response. I initially animated the dragged card and the destination at the same time. It looked impressive in a static demo, but the pointer feedback became hard to read once the card crossed several targets.

## Give each part one responsibility

`DragTarget` already tells you when a compatible item enters, leaves, and gets accepted. That state is enough to drive `flutter_animate` with `target`. There is no need for a second `AnimationController` or a rebuild of the whole board.

| Part | Owner | What changes |
|---|---|---|
| Pointer movement | `Draggable` | The feedback widget follows the pointer |
| Valid target detection | `DragTarget` | `isHovering` becomes true or false |
| Drop result | `onAcceptWithDetails` | The model moves the item |
| Target emphasis | `flutter_animate` | A short scale cue confirms the valid zone |

Here is a small target that accepts task IDs and gives immediate hover feedback:

```dart
class TaskDropZone extends StatefulWidget {
  const TaskDropZone({super.key, required this.onDrop});

  final ValueChanged<String> onDrop;

  @override
  State<TaskDropZone> createState() => _TaskDropZoneState();
}

class _TaskDropZoneState extends State<TaskDropZone> {
  bool _isHovering = false;

  @override
  Widget build(BuildContext context) {
    return DragTarget<String>(
      onWillAcceptWithDetails: (details) {
        final accepted = details.data.isNotEmpty;
        if (_isHovering != accepted) {
          setState(() => _isHovering = accepted);
        }
        return accepted;
      },
      onLeave: (_) {
        if (_isHovering) setState(() => _isHovering = false);
      },
      onAcceptWithDetails: (details) {
        setState(() => _isHovering = false);
        widget.onDrop(details.data);
      },
      builder: (context, candidates, rejected) {
        return AnimatedContainer(
          duration: 160.ms,
          padding: const EdgeInsets.all(20),
          decoration: BoxDecoration(
            color: _isHovering
                ? Colors.blue.withValues(alpha: 0.16)
                : Colors.transparent,
            border: Border.all(
              color: _isHovering ? Colors.blue : Colors.grey.shade400,
              width: _isHovering ? 2 : 1,
            ),
            borderRadius: BorderRadius.circular(16),
          ),
          child: const Icon(Icons.download),
        )
            .animate(target: _isHovering ? 1 : 0)
            .scale(
              begin: const Offset(1, 1),
              end: const Offset(1.04, 1.04),
              duration: 160.ms,
            );
      },
    );
  }
}
```

The `AnimatedContainer` handles the stable border and color transition. `flutter_animate` adds a small scale change, which is enough to make the destination discoverable without changing the board's layout. Because `target` moves between `0` and `1`, the same effect reverses when the card leaves the zone.

## Do not use candidate lists as your only state

The `candidates` argument in the builder is useful for inspecting active drags, but it is not always a good visual state source. A pointer can leave the target between frames, and a rejected draggable can still cause builder activity. Updating `_isHovering` in `onWillAcceptWithDetails` and clearing it in `onLeave` makes the intended behavior explicit.

There is another trap in the callback order. Do not remove the task from the source list inside `onWillAcceptWithDetails`; that callback only answers whether the target can accept it. Move the model in `onAcceptWithDetails`, after the drop is real. Otherwise a canceled drag can delete data even though no item was accepted.

For a real board, use an immutable task ID rather than the displayed title:

```dart
Draggable<String>(
  data: task.id,
  feedback: Material(
    color: Colors.transparent,
    child: TaskCard(task: task),
  ),
  childWhenDragging: TaskCard(task: task, dimmed: true),
  child: TaskCard(task: task),
)
```

The stable ID keeps the source row and the drop result connected even when another task has the same title. Keep the feedback widget lightweight too. A large subtree with shadows, images, and its own looping animation can make drag movement feel laggy on older phones.

## Short checklist

- Let `Draggable` own pointer motion and `DragTarget` own acceptance.
- Use `animate(target: ...)` for hover state instead of restarting an entrance animation.
- Change the data model only in `onAcceptWithDetails`.
- Keep the target's scale small so it does not disturb surrounding layout.
- Test accepted, rejected, canceled, and rapid re-entry paths.

`flutter_animate` is most useful here as a feedback layer. The drop zone should acknowledge a valid destination quickly, then get out of the way while Flutter continues to manage the gesture and the board's layout.
