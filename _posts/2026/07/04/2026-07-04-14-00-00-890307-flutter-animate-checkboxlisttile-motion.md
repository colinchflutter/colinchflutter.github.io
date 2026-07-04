---
layout: post
title: "flutter_animate CheckboxListTile Motion - settings toggles without noisy list replays"
description: "Learn how to use flutter_animate with CheckboxListTile so changed settings feel responsive without replaying the whole list."
date: 2026-07-04
tags: [flutter_animate, animation, performance, ListView]
comments: true
share: true
---
![A settings screen with toggle controls on a mobile device](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1200&q=80)
This image is a reminder to keep settings screens readable: motion should confirm the changed row, not decorate every option.

`flutter_animate` works best with `CheckboxListTile` when the animation is tied to the setting that changed. The common mistake is wrapping the whole `ListView` in an entrance effect. It looks fine with 4 rows, then becomes distracting when the screen has 18 notification, privacy, and sync options.

The clean pattern is to keep the checkbox state boring and stable, then animate a small visual hint around the active row. I usually animate background color, scale, or a short fade on the supporting text. I do not animate the checkbox itself unless the app has a custom control, because Material already gives the check mark its own state transition.

| Area | Better motion | Trap |
| --- | --- | --- |
| Changed row | quick tint or scale | replaying every tile |
| Subtitle | fade in updated detail | moving layout height |
| Checkbox | let Material handle it | stacking extra bounce |
| Long settings list | keyed row animation | list-level entrance |

Here is the version I use when a settings row needs feedback after a tap.

```dart
class SettingsToggleTile extends StatelessWidget {
  const SettingsToggleTile({
    super.key,
    required this.title,
    required this.value,
    required this.onChanged,
  });

  final String title;
  final bool value;
  final ValueChanged<bool> onChanged;

  @override
  Widget build(BuildContext context) {
    return CheckboxListTile(
      key: ValueKey('$title-$value'),
      title: Text(title),
      subtitle: Text(value ? 'Enabled for this device' : 'Disabled'),
      value: value,
      onChanged: (next) => onChanged(next ?? false),
    )
        .animate(key: ValueKey('motion-$title-$value'))
        .fadeIn(duration: 120.ms)
        .scale(
          begin: const Offset(.985, .985),
          end: const Offset(1, 1),
          duration: 140.ms,
          curve: Curves.easeOut,
        )
        .tint(
          color: value
              ? Theme.of(context).colorScheme.primary.withOpacity(.08)
              : Colors.transparent,
          duration: 180.ms,
        );
  }
}
```

The two keys matter. The tile key keeps Flutter from confusing one row with another when the list rebuilds. The animation key changes only when this setting changes, so a parent rebuild from search, theme, or account refresh does not replay unrelated rows. That was the bug I hit first: toggling one setting made every visible option pulse because the animation lived above the builder.

If the row writes to disk or a remote profile, keep the optimistic toggle separate from the save status. A tiny trailing progress indicator is clearer than shaking the whole row for 700 ms.

```dart
trailing: isSaving
    ? const SizedBox.square(
        dimension: 18,
        child: CircularProgressIndicator(strokeWidth: 2),
      )
    : null,
```

The main rule is simple: `CheckboxListTile` owns selection, `flutter_animate` owns confirmation. When those jobs stay separate, settings screens feel responsive without making users re-scan the entire list after every tap.
