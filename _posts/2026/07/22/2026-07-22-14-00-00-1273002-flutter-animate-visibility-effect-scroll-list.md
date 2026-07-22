---
layout: post
title: "flutter_animate VisibilityEffect - Hide Flutter Widgets Without Scroll Jumps"
description: "Learn how flutter_animate VisibilityEffect controls widget visibility with maintained layout, fade and slide coordination, and stable keys for scrollable Flutter lists."
date: 2026-07-22
tags: [flutter_animate, animation, Flutter, performance, ListView]
comments: true
share: true
---

![Flutter list cards using flutter_animate visibility transitions](/assets/images/flutter-animate-animate-list-stable-keys.png)

*The useful detail is that a hidden card can keep its list slot while its visual entrance finishes.*

`flutter_animate`'s `VisibilityEffect` is for a different job than `FadeEffect`. It toggles Flutter's `Visibility` widget at a timeline point, so the child can become hidden or visible without writing a separate boolean wrapper. I find it most useful for optional list rows, permission-dependent controls, and content that should stop participating in semantics or hit testing after an exit sequence.

The trap is treating it like an opacity animation. A widget can be fully transparent and still occupy space, receive semantics, or intercept interaction. `VisibilityEffect` changes that behavior. Decide whether the layout slot should remain, then set `maintain` deliberately.

## VisibilityEffect compared with the usual choices

The visual result may look similar, but the layout and accessibility behavior are not the same.

| Need | Use | What changes |
| --- | --- | --- |
| Keep the widget in the layout and fade its pixels | `FadeEffect` | Opacity changes; the child still exists |
| Hide or show at a timeline boundary | `VisibilityEffect` | Flutter `Visibility` controls participation |
| Animate the size of a disappearing child | `AnimatedSize` | Parent layout follows the changing size |
| Replace one state with another | `CrossfadeEffect` | Incoming and outgoing children overlap |

`VisibilityEffect` does not interpolate visibility. It evaluates a boolean at the beginning and end of its interval. That makes it a good boundary effect: run a fade or slide, then switch the child off once the animation has finished.

## A visible entrance that keeps its list slot

For an item entering a list, keeping the slot during the short entrance avoids a one-frame collapse while neighboring rows are being laid out. The `maintain` flag applies the relevant `Visibility` maintenance properties, including size, animation, state, semantics, and interactivity.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class TaskTile extends StatelessWidget {
  const TaskTile({super.key, required this.task});

  final Task task;

  @override
  Widget build(BuildContext context) {
    return Animate(
      key: ValueKey(task.id),
      child: ListTile(
        leading: CircleAvatar(child: Text(task.title[0])),
        title: Text(task.title),
        subtitle: Text(task.project),
      ),
      effects: [
        const VisibilityEffect(maintain: true),
        FadeEffect(duration: 220.ms, curve: Curves.easeOut),
        SlideEffect(
          begin: const Offset(0, 0.08),
          end: Offset.zero,
          duration: 220.ms,
          curve: Curves.easeOutCubic,
        ),
      ],
    );
  }
}
```

The key belongs on the animated wrapper and must represent the item identity. If the list reuses one `Animate` state for a different task, the controller can already be complete and the new row will appear without an entrance. This is the same stable-key issue that shows up with `AnimateList`, but it is easier to miss when the rows come from a builder.

For a long `ListView`, I would not set `maintain: true` automatically. It keeps more of the hidden subtree alive than a plain hidden widget. Use it when preserving the row's footprint during the transition matters; allow the item to disappear completely when it has already been removed from the data model.

## Coordinating an exit before removal

The clean exit sequence has two separate moments: visual motion first, structural removal second. The parent owns the removal, while the child reports when its animation completes.

```dart
class DismissibleTask extends StatefulWidget {
  const DismissibleTask({super.key, required this.onRemoved});

  final VoidCallback onRemoved;

  @override
  State<DismissibleTask> createState() => _DismissibleTaskState();
}

class _DismissibleTaskState extends State<DismissibleTask> {
  var _removing = false;

  void _remove() {
    if (_removing) return;
    setState(() => _removing = true);
  }

  @override
  Widget build(BuildContext context) {
    return Animate(
      key: ValueKey(_removing),
      child: ListTile(
        title: const Text('Archive this task'),
        trailing: IconButton(
          onPressed: _remove,
          icon: const Icon(Icons.archive_outlined),
        ),
      ),
      effects: _removing
          ? [
              FadeEffect(begin: 1, end: 0, duration: 180.ms),
              SlideEffect(
                begin: Offset.zero,
                end: const Offset(-0.12, 0),
                duration: 180.ms,
              ),
              const VisibilityEffect(
                delay: Duration(milliseconds: 180),
                duration: Duration.zero,
                end: false,
              ),
            ]
          : const [],
      onComplete: (_) {
        if (_removing) widget.onRemoved();
      },
    );
  }
}
```

Here `VisibilityEffect` is scheduled at the end of the fade. The parent should remove the item in `onRemoved`, not immediately inside `_remove`. If the data source removes it first, the widget leaves the tree and there is no opportunity to see the exit animation.

The `ValueKey(_removing)` forces a fresh `Animate` state when the exit branch is selected. Without that key, the new effect list is attached to an existing animation state and the result can depend on whether the previous controller has already completed. I would also guard the callback because a double tap or a parent rebuild should not remove the same row twice.

## The maintenance decision that caused my layout bug

I initially used `maintain: true` on a disappearing row and expected it to collapse after `VisibilityEffect` reached `false`. It did not. The row became visually hidden but continued reserving its height, leaving a blank gap in the list.

| Exit behavior | `maintain` choice | Result |
| --- | --- | --- |
| Parent removes the item after the animation | Omit it | Hidden child can stop occupying the slot |
| Another widget will fill the slot | `true` | Layout remains stable during the transition |
| Preserve state while temporarily hidden | `true` | Useful for tabs or cached panels |
| Hide a control from users and assistive tech | Omit it | Do not preserve interactivity or semantics accidentally |

If the row should shrink instead of disappearing in place, pair the animation with `AnimatedSize` owned by the parent. `VisibilityEffect` answers “should this child participate?”; it is not a replacement for a size transition.

## Practical rules

- Use `FadeEffect` when only pixels should change; use `VisibilityEffect` for a participation boundary.
- Put a stable identity key on the animated subtree when list data or visibility state changes.
- Set `maintain` only when preserving layout or state is intentional.
- Let the parent own data removal and call it after `onComplete`.
- Keep exit effects short. A 160–220ms window was enough for a task row; longer timings made fast scrolling feel sticky.

`VisibilityEffect` is small, but it solves a real boundary problem in Flutter UI: the difference between looking gone and actually being gone. Use it after the visual work, choose the maintenance behavior explicitly, and the animation will not fight the list's layout or interaction model.

### References

- [flutter_animate API documentation](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/)
- [Flutter Visibility documentation](https://api.flutter.dev/flutter/widgets/Visibility-class.html)
