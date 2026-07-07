---
layout: post
title: "flutter_animate SwitchListTile Motion - setting changes without row jitter"
description: "Use flutter_animate with Flutter SwitchListTile to show setting changes without replaying rows, shifting labels, or breaking semantics."
date: 2026-07-08
tags: [flutter_animate, animation, state_management, Flutter, performance]
comments: true
share: true
---
![flutter_animate SwitchListTile settings motion](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
The useful detail in this image is the settings surface: switch motion should confirm a changed preference without making the whole row feel like it moved.

`flutter_animate` works best with `SwitchListTile` when the animation is attached to the small piece of UI that changed, not to the full tile. Keep the switch, tap target, semantics, and row height owned by Flutter. Animate a helper line, trailing status text, or a tiny confirmation pulse beside the label.

The mistake I made was animating the entire tile after each toggle. It looked good with one setting. With 9 settings in a dense screen, every toggle caused text, icon, and switch alignment to replay. Worse, a remote preference refresh looked like the user tapped three switches at once. The fix was smaller: state changes get a 180 ms cue, row creation gets no repeat animation.

## Attach motion to state, not build

`SwitchListTile` already animates the thumb and track. Adding another full-row slide usually creates duplicate motion. The better pattern is to key a small child by the boolean value and leave the list structure stable.

```dart
class AnimatedSettingSwitch extends StatelessWidget {
  const AnimatedSettingSwitch({
    super.key,
    required this.title,
    required this.value,
    required this.onChanged,
    this.subtitle,
  });

  final String title;
  final String? subtitle;
  final bool value;
  final ValueChanged<bool> onChanged;

  @override
  Widget build(BuildContext context) {
    final color = value
        ? Theme.of(context).colorScheme.primary
        : Theme.of(context).colorScheme.onSurfaceVariant;

    return SwitchListTile(
      value: value,
      onChanged: onChanged,
      title: Row(
        children: [
          Expanded(child: Text(title)),
          AnimatedStatusDot(enabled: value, color: color),
        ],
      ),
      subtitle: subtitle == null
          ? null
          : Text(subtitle!, maxLines: 2, overflow: TextOverflow.ellipsis),
    );
  }
}
```

Here is the small animated piece. The `ValueKey` is deliberate: it restarts only when the setting value changes, not when the parent list rebuilds.

```dart
class AnimatedStatusDot extends StatelessWidget {
  const AnimatedStatusDot({
    super.key,
    required this.enabled,
    required this.color,
  });

  final bool enabled;
  final Color color;

  @override
  Widget build(BuildContext context) {
    return Container(
      key: ValueKey(enabled),
      width: 10,
      height: 10,
      decoration: BoxDecoration(
        color: color,
        shape: BoxShape.circle,
      ),
    )
        .animate()
        .scale(
          begin: const Offset(0.72, 0.72),
          end: const Offset(1, 1),
          duration: 140.ms,
          curve: Curves.easeOutCubic,
        )
        .fadeIn(duration: 100.ms);
  }
}
```

## What should animate

The rule I use is boring but reliable: animate confirmation, not control ownership.

| Target | Good use | Risk |
| --- | --- | --- |
| Status dot | Confirms on/off change | Low, fixed size |
| Helper text | Explains a changed mode | Medium, can shift height |
| Whole tile | Rare first-load reveal | High, replays on rebuild |
| Switch itself | Usually unnecessary | Can fight built-in motion |

If the helper text changes from "Off" to "Syncs over Wi-Fi only", reserve enough vertical space or keep the text outside the animated widget. Animating height in a settings list is where jitter usually starts.

## Handling saved and pending states

Real settings often have three states: current value, pending network save, and failed save. Do not use the switch animation as the only signal. Keep `onChanged` disabled while saving if duplicate taps would cause bad writes, then animate a small trailing state.

```dart
SwitchListTile(
  value: notificationsEnabled,
  onChanged: saving ? null : (next) => saveNotifications(next),
  title: Row(
    children: [
      const Expanded(child: Text('Notifications')),
      if (saving)
        const SizedBox.square(
          dimension: 14,
          child: CircularProgressIndicator(strokeWidth: 2),
        ).animate().fadeIn(duration: 120.ms)
      else
        AnimatedStatusDot(
          enabled: notificationsEnabled,
          color: Theme.of(context).colorScheme.primary,
        ),
    ],
  ),
  subtitle: const Text('Alerts for direct messages and mentions'),
);
```

I avoid shaking the row on save failure. A failed switch write is not a form validation error. Revert the value, show a compact error line or snackbar, and let the status element do one short color pulse. That gives enough feedback without making the settings page feel unstable.

## Keep rebuilds cheap

In a long settings screen, put each setting value close to the row that needs it. With `Provider`, `Riverpod`, `Bloc`, or plain `ValueListenableBuilder`, the goal is the same: toggling one row should not replay visual cues for every row.

```dart
ValueListenableBuilder<bool>(
  valueListenable: wifiOnlySync,
  builder: (context, value, _) {
    return AnimatedSettingSwitch(
      title: 'Wi-Fi only sync',
      subtitle: 'Avoid mobile data while uploads are pending',
      value: value,
      onChanged: (next) => wifiOnlySync.value = next,
    );
  },
);
```

Two details matter in production. Use stable keys for animated indicators, and keep their width fixed. If a trailing label changes from `Off` to `Enabled`, place it in a `SizedBox(width: 72)` or the whole title row will breathe sideways.

## Short checklist

- Keep `SwitchListTile` responsible for tap behavior and semantics.
- Animate only the status cue, helper text, or save indicator.
- Key animated children by the setting value, not by list index.
- Avoid row-level slide or shake on normal toggles.
- Reserve width and height for changing labels.

The final screen feels calmer because the user sees exactly what changed. The switch still behaves like a Flutter switch, the row stays readable, and `flutter_animate` adds confirmation instead of taking over the control.
