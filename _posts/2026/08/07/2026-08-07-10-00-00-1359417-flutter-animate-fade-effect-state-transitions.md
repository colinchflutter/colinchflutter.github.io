---
layout: post
title: "flutter_animate FadeEffect - Animate Flutter State Changes Without Layout Jumps"
description: "Learn how to use flutter_animate FadeEffect for loading, success, and error states while keeping Flutter layout stable with target-driven animations."
date: 2026-08-07
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![Flutter animation curves and state transition illustration](/assets/images/flutter-animate-curves-motion.png)

The practical use for `flutter_animate`'s `FadeEffect` is not just making a widget appear. It is switching between loading, success, and error content without changing the space that the surrounding layout expects. The child can become visually transparent while its layout box stays stable, which prevents the small vertical jumps that make async screens feel unfinished.

## Why a simple opacity change is easy to get wrong

I first used `AnimatedOpacity` for an async status card and assumed the problem was solved. The fade looked fine, but the card was still removed from the tree when the request returned an error. The next widget moved upward immediately, so the animation hid the content but did not preserve the layout.

`FadeEffect` has the same rendering rule: opacity changes during painting, not during layout. That makes it useful for a fixed state slot, but it does not magically reserve space if the widget is conditionally removed with `if`.

| Requirement | Recommended pattern | Reason |
| --- | --- | --- |
| Fade a fixed card in or out | `FadeEffect` | Keeps the card's layout box in place |
| Replace loading text with result text | Stable parent + `target` | State changes replay the same effect chain |
| Remove a finished widget | Keep it mounted until the fade ends | Unmounting immediately cancels the visual transition |
| Hide a required error message | Do not use opacity alone | Invisible content is still inaccessible to users |

The distinction matters in a `Column`: an invisible child still occupies its measured height, while a missing child does not.

## A target-driven async status card

For a status card, make the animation target represent whether the content should be visible. The effect itself can remain unchanged while `setState` moves the target between `0.0` and `1.0`.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

enum LoadState { loading, success, error }

class ProfileStatus extends StatefulWidget {
  const ProfileStatus({super.key});

  @override
  State<ProfileStatus> createState() => _ProfileStatusState();
}

class _ProfileStatusState extends State<ProfileStatus> {
  LoadState _state = LoadState.loading;

  Future<void> _finishRequest() async {
    await Future<void>.delayed(const Duration(milliseconds: 900));
    if (!mounted) return;
    setState(() => _state = LoadState.success);
  }

  @override
  void initState() {
    super.initState();
    _finishRequest();
  }

  @override
  Widget build(BuildContext context) {
    final isVisible = _state != LoadState.loading;

    return Column(
      children: [
        _LoadingCard(visible: !isVisible),
        _ResultCard(state: _state, visible: isVisible),
      ],
    );
  }
}

class _LoadingCard extends StatelessWidget {
  const _LoadingCard({required this.visible});

  final bool visible;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: const SizedBox(
        height: 96,
        child: Center(child: CircularProgressIndicator()),
      ),
    )
        .animate(target: visible ? 0.0 : 1.0)
        .fade(begin: 1.0, end: 0.0, duration: 220.ms);
  }
}

class _ResultCard extends StatelessWidget {
  const _ResultCard({required this.state, required this.visible});

  final LoadState state;
  final bool visible;

  @override
  Widget build(BuildContext context) {
    final isError = state == LoadState.error;

    return Card(
      child: SizedBox(
        height: 96,
        child: ListTile(
          leading: Icon(
            isError ? Icons.error_outline : Icons.check_circle_outline,
            color: isError ? Colors.red : Colors.green,
          ),
          title: Text(isError ? 'Could not load profile' : 'Profile ready'),
          subtitle: Text(isError ? 'Try again in a moment.' : 'Your data is up to date.'),
        ),
      ),
    )
        .animate(target: visible ? 1.0 : 0.0)
        .fade(begin: 0.0, end: 1.0, duration: 260.ms)
        .slideY(begin: 0.04, end: 0.0, duration: 260.ms);
  }
}
```

The two cards above use fixed heights so the transition does not reflow the page. In a real screen, I usually put both states inside one `SizedBox` and use a `Stack` when the cards must occupy exactly the same position. A `Column` is fine when the loading and result slots are intentionally separate.

The important line is `animate(target: visible ? 1.0 : 0.0)`. Setting the same target twice does not replay anything. If a user taps “retry” while the result is already visible, toggle a separate animation token or rebuild with a changing `ValueKey`; simply assigning `true` to an already-true boolean gives `flutter_animate` no new transition to run.

## Fade only the visual layer, not the interaction contract

An opacity of `0.0` does not necessarily mean “gone.” The widget can still participate in hit testing, semantics, and focus depending on how it is composed. That is useful for preserving layout, but dangerous for buttons and form fields.

```dart
Widget buildSaveButton({required bool saving}) {
  final button = FilledButton(
    onPressed: saving ? null : _save,
    child: Text(saving ? 'Saving…' : 'Save changes'),
  );

  return IgnorePointer(
    ignoring: saving,
    child: button
        .animate(target: saving ? 0.0 : 1.0)
        .fade(begin: 1.0, end: 0.0, duration: 180.ms),
  );
}
```

For status text, an invisible error is a usability bug. Prefer `Semantics`, an `AnimatedSwitcher`, or a visible fallback when the message is required for the task. Use `FadeEffect` for decoration and state transitions where the meaning is still exposed elsewhere.

## A few values worth checking before shipping

- Use 180–300 ms for a local state change. Longer fades make a button feel disconnected from the tap that caused it.
- Keep the child mounted while the fade runs. Conditional removal defeats the layout-stability benefit.
- Give async result cards stable keys when they appear in a list. Otherwise Flutter may reuse an element and animate the previous row's state.
- Test the first frame. A `FadeEffect(begin: 0.0, end: 1.0)` starts transparent, so a missing initial frame or an immediate screenshot can look like a blank widget.
- Respect reduced-motion settings for decorative fades, and never use opacity to hide a required success or error result.

The useful mental model is simple: `FadeEffect` controls visibility on the paint layer, while Flutter still controls layout, hit testing, and semantics. Keep those responsibilities explicit, and loading-to-result transitions stay smooth without surprising the rest of the screen.
