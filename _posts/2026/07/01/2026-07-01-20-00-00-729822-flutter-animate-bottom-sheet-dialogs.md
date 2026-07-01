---
layout: post
title: "flutter_animate Bottom Sheets - entry and exit motion for overlays"
description: "Use flutter_animate inside Flutter bottom sheets and dialogs without fighting route transitions, rebuilds, or early Navigator.pop calls."
date: 2026-07-01
tags: [flutter_animate, animation, navigation, Flutter]
comments: true
share: true
---
![Flutter bottom sheet overlay with animated controls](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

`flutter_animate` works well inside bottom sheets and dialogs when you treat the overlay route as the container and animate the content, not the route itself. Let Flutter open the sheet, then use `flutter_animate` for the controls, title, error state, and close affordance.

The trap is trying to replace `showModalBottomSheet`'s transition with a widget animation. That usually creates double motion: the route slides up while the child also slides up from too far away. I have shipped that once in a filter sheet, and it felt cheap because the sheet arrived after its own contents had already started moving.

## The overlay problem

Bottom sheets and dialogs are not normal widgets in the current page tree. They live in a route managed by `Navigator`. That route owns the barrier, focus, back button behavior, safe area, and the default entrance transition.

So the clean boundary is simple: Flutter owns the overlay shell, `flutter_animate` owns the content inside it. This keeps the animation chain local and avoids custom route code for every small modal.

## Animating sheet content

This is the pattern I use for a settings or filter sheet. The route slides the sheet into place. The content fades and settles after the sheet exists.

```dart
Future<void> openFilterSheet(BuildContext context) {
  return showModalBottomSheet<void>(
    context: context,
    isScrollControlled: true,
    useSafeArea: true,
    builder: (_) => const _FilterSheet(),
  );
}

class _FilterSheet extends StatelessWidget {
  const _FilterSheet();

  @override
  Widget build(BuildContext context) {
    const options = ['Open now', 'Has delivery', 'Top rated'];

    return Padding(
      padding: const EdgeInsets.fromLTRB(20, 16, 20, 24),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'Filters',
            style: TextStyle(fontSize: 22, fontWeight: FontWeight.w700),
          )
              .animate()
              .fadeIn(duration: 180.ms)
              .slideY(begin: 0.18, end: 0, curve: Curves.easeOutCubic),
          const SizedBox(height: 12),
          ...List.generate(options.length, (index) {
            final label = options[index];
            return CheckboxListTile(
              value: false,
              onChanged: (_) {},
              title: Text(label),
              contentPadding: EdgeInsets.zero,
            )
                .animate(delay: (index * 45).ms)
                .fadeIn(duration: 180.ms)
                .slideX(begin: -0.04, end: 0);
          }),
        ],
      ),
    );
  }
}
```

Keep the stagger tight. A modal is already interrupting the user, so slow decorative timing feels like friction. For most sheets, `35.ms` to `60.ms` between rows is enough to make the hierarchy readable without making the user wait.

## Handling close motion

Entrance is easy because the widget is being built. Exit is different. If you call `Navigator.pop(context)`, the route starts closing immediately and your child animation may never be seen.

When the close animation matters, hold a local `_closing` flag, reverse the content with `target`, wait for the short duration, then pop.

```dart
class ConfirmDialogBody extends StatefulWidget {
  const ConfirmDialogBody({super.key});

  @override
  State<ConfirmDialogBody> createState() => _ConfirmDialogBodyState();
}

class _ConfirmDialogBodyState extends State<ConfirmDialogBody> {
  var _closing = false;

  Future<void> _close() async {
    if (_closing) return;
    setState(() => _closing = true);
    await Future<void>.delayed(160.ms);
    if (mounted) Navigator.of(context).pop();
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('Discard draft?'),
      content: const Text('Unsaved changes will be lost.'),
      actions: [
        TextButton(onPressed: _close, child: const Text('Cancel')),
        FilledButton(onPressed: _close, child: const Text('Discard')),
      ],
    )
        .animate(target: _closing ? 0 : 1)
        .fade(begin: 0, end: 1, duration: 160.ms)
        .scale(begin: const Offset(0.96, 0.96), end: const Offset(1, 1));
  }
}
```

This pattern is only worth it for dialogs where the close state communicates something: confirmation, destructive action, success, or failed validation. For a plain menu sheet, the default route exit is enough.

## Avoiding double animation

Do not put a large `.slideY(begin: 1)` on the whole bottom sheet content. `showModalBottomSheet` already comes from the bottom, so the child motion should be subtle: `0.08`, `0.12`, maybe `0.18` for a title. The route does the travel; the child does the settling.

Also avoid long delays inside dialogs. If the primary button appears 500ms after the dialog, keyboard and screen reader users can end up with focus timing that feels broken. Animate visual emphasis, not availability. The controls should be built immediately.

## Practical rule

Use `flutter_animate` inside overlays for content choreography, not route ownership. Let `showModalBottomSheet` and `showDialog` manage barriers and navigation. Animate titles, rows, buttons, and validation states with short durations. Delay `Navigator.pop` only when the exit animation carries meaning; otherwise close the overlay immediately and keep the interface fast.
