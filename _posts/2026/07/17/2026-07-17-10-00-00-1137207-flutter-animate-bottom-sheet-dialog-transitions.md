---
layout: post
title: "flutter_animate Bottom Sheets and Dialogs - Animate Overlay Entry and Exit Safely"
description: "Use flutter_animate with Flutter bottom sheets and dialogs to animate overlay content without fighting route dismissal or creating duplicate controllers."
date: 2026-07-17
tags: [flutter_animate, animation, navigation, Flutter, Android, iOS]
comments: true
share: true
---

![flutter_animate bottom sheet and dialog transition]({{ site.baseurl }}/assets/images/flutter-animate-bottom-sheet-dialog-transitions.png)

This image highlights the important boundary: a bottom sheet or dialog is an overlay route, so its content animation and its route transition should not compete for ownership.

`flutter_animate` is a good fit for animating the content inside a bottom sheet or dialog. It is not a replacement for the route that positions, dismisses, and restores that overlay. The practical pattern is to let Flutter own the route transition, then use `Animate` for the sheet body, status card, or dialog controls. Trying to make an inner widget reverse after `Navigator.pop()` is the trap: the route removes that widget before its reverse animation can be seen.

## Animate the sheet content, not the route

For a standard modal sheet, `showModalBottomSheet` already provides the slide-from-bottom behavior. Add a small stagger to the content so the visual hierarchy is clear without making the whole sheet feel slow:

```dart
Future<void> showUploadSheet(BuildContext context) {
  return showModalBottomSheet<void>(
    context: context,
    isScrollControlled: true,
    builder: (context) {
      return SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Animate(
                effects: [
                  fadeIn(duration: 220.ms),
                  slideY(begin: .12, end: 0, duration: 280.ms),
                ],
                child: const _UploadHeader(),
              ),
              12.verticalSpace,
              Animate(
                delay: 90.ms,
                effects: [fadeIn(), slideY(begin: .08, end: 0)],
                child: const _UploadActions(),
              ),
            ],
          ),
        ),
      );
    },
  );
}
```

The route moves the sheet as one unit, while the two `Animate` widgets reveal the header and actions after they are already in the correct route. That separation prevents a common visual bug where the sheet slides upward and its children slide upward again, making the overlay travel twice as far.

| Layer | Owner | Good responsibility |
| --- | --- | --- |
| Overlay route | Flutter | Position, barrier, drag, dismissal |
| Sheet or dialog content | `flutter_animate` | Fade, scale, stagger, emphasis |
| Business state | Your model | Loading, success, error, retry |

## Dialogs need the same boundary

The same rule applies to `showDialog`. Animate the dialog card, not a full-screen `Material` wrapper. A short scale and fade works better than a large translation because the barrier already communicates that the user entered a new layer:

```dart
showDialog<void>(
  context: context,
  builder: (context) {
    return AlertDialog(
      content: Animate(
        effects: [
          fadeIn(duration: 160.ms),
          scale(begin: const Offset(.96, .96), end: const Offset(1, 1)),
        ],
        child: const _DeleteAccountContent(),
      ),
    );
  },
);
```

There is a subtle lifecycle detail here. The dialog route owns the barrier and starts its own exit transition when `Navigator.pop` is called. The inner `Animate` is not guaranteed to remain mounted long enough to reverse. If the exit motion is part of the product interaction, delay the pop until a local closing state has finished:

```dart
class _DialogBody extends StatefulWidget {
  const _DialogBody();

  @override
  State<_DialogBody> createState() => _DialogBodyState();
}

class _DialogBodyState extends State<_DialogBody> {
  bool _closing = false;

  Future<void> _close() async {
    setState(() => _closing = true);
    await Future<void>.delayed(180.ms);
    if (mounted) Navigator.of(context).pop();
  }

  @override
  Widget build(BuildContext context) {
    return Animate(
      target: _closing ? 0 : 1,
      effects: [fade(begin: 0, end: 1), scale(begin: .96, end: 1)],
      child: TextButton(onPressed: _close, child: const Text('Cancel')),
    );
  }
}
```

This is deliberately a small, explicit handoff. Keep the real operation state outside `_closing`; closing only describes the visual exit. If deletion fails, the dialog should not pretend that the business state succeeded merely because its animation completed.

## Three checks that prevent overlay motion bugs

- Do not wrap both the route and its entire child tree in the same slide effect.
- Keep entrance delays below the route transition duration, otherwise the sheet appears empty before its content arrives.
- Test back-button, barrier-tap, drag dismissal, and programmatic `pop`; they do not all pass through the same button callback.

`flutter_animate` works best around overlays when it owns local visual hierarchy and Flutter continues to own route lifecycle. That division gives bottom sheets and dialogs motion that feels intentional, while dismissal remains compatible with focus, accessibility, gestures, and navigation.
