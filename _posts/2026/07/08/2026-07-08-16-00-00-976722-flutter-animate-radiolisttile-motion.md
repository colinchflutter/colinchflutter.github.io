---
layout: post
title: "flutter_animate RadioListTile Motion - single-choice feedback without list jitter"
description: "Use flutter_animate with Flutter RadioListTile to show selected feedback without replaying every option in the list."
date: 2026-07-08
tags: [flutter_animate, animation, state_management, Flutter, performance]
comments: true
share: true
---
![flutter_animate RadioListTile single choice motion](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)
The useful detail in this image is the dashboard choice pattern: one selected option should feel confirmed without making the whole settings list move.

`flutter_animate` is a good fit for `RadioListTile` when the animation belongs to the selected row, not to the radio group container. The practical target is small feedback: a short background tint, a trailing check cue, or a label emphasis that confirms the new value. The trap I hit was animating every tile from the same parent builder. It looked fine with 2 options, then became distracting with 6 options because old and new rows replayed together after every tap.

## Animate the chosen row only

`RadioListTile` already gives you accessible single-choice behavior: one `groupValue`, one `value` per row, and an `onChanged` callback. Keep that contract boring. Add motion around the parts that are allowed to change visually, and key the animation with both the option value and its selected state.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

enum SyncMode { wifiOnly, cellular, manual }

class SyncModePicker extends StatefulWidget {
  const SyncModePicker({super.key});

  @override
  State<SyncModePicker> createState() => _SyncModePickerState();
}

class _SyncModePickerState extends State<SyncModePicker> {
  SyncMode _mode = SyncMode.wifiOnly;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: SyncMode.values.map((mode) {
        final selected = mode == _mode;

        return _SyncModeTile(
          mode: mode,
          selected: selected,
          groupValue: _mode,
          onChanged: (value) {
            if (value == null || value == _mode) return;
            setState(() => _mode = value);
          },
        );
      }).toList(),
    );
  }
}

class _SyncModeTile extends StatelessWidget {
  const _SyncModeTile({
    required this.mode,
    required this.selected,
    required this.groupValue,
    required this.onChanged,
  });

  final SyncMode mode;
  final bool selected;
  final SyncMode groupValue;
  final ValueChanged<SyncMode?> onChanged;

  @override
  Widget build(BuildContext context) {
    final label = switch (mode) {
      SyncMode.wifiOnly => 'Wi-Fi only',
      SyncMode.cellular => 'Wi-Fi and cellular',
      SyncMode.manual => 'Manual sync',
    };

    return AnimatedContainer(
      duration: 140.ms,
      curve: Curves.easeOutCubic,
      margin: const EdgeInsets.symmetric(vertical: 4),
      decoration: BoxDecoration(
        color: selected
            ? Theme.of(context).colorScheme.primaryContainer.withOpacity(.34)
            : Colors.transparent,
        borderRadius: BorderRadius.circular(8),
      ),
      child: RadioListTile<SyncMode>(
        value: mode,
        groupValue: groupValue,
        onChanged: onChanged,
        title: Text(label),
        subtitle: Text(_helperText(mode)),
        secondary: AnimatedSwitcher(
          duration: 160.ms,
          child: selected
              ? const Icon(Icons.check_circle, key: ValueKey('selected'))
              : const SizedBox(width: 24, key: ValueKey('empty')),
        )
            .animate(key: ValueKey('${mode.name}-$selected'))
            .fadeIn(duration: 100.ms)
            .scale(begin: const Offset(.92, .92), duration: 160.ms),
      ),
    );
  }

  String _helperText(SyncMode mode) {
    return switch (mode) {
      SyncMode.wifiOnly => 'Saves data on metered plans',
      SyncMode.cellular => 'Keeps files current everywhere',
      SyncMode.manual => 'Syncs only when you ask',
    };
  }
}
```

The row uses `AnimatedContainer` for the background because it changes between two stable paint states. The trailing icon uses `AnimatedSwitcher` plus `flutter_animate` because the selected marker appears and disappears. That split matters. If the entire `RadioListTile` slides or scales, the radio target moves under the pointer, and the list feels less reliable.

## Choose the smallest moving part

| Moving part | Use when | Risk |
| --- | --- | --- |
| Background tint | The selected row needs stronger contrast | Too much opacity can fight Material selected colors |
| Secondary icon | The option needs a clear confirmation cue | Width changes if the empty state is not sized |
| Subtitle text | The explanation changes with selection | Text height changes can push nearby rows |
| Whole tile | Short, isolated lists only | Replays focus, touch target, and neighboring content |

I usually start with the secondary icon and a faint background. It gives enough feedback for a settings screen without making the option labels jump. The important small fix is the `SizedBox(width: 24)` in the unselected branch. Without it, the row title shifts when the icon appears, which makes the animation look like a layout bug rather than feedback.

## Guard repeated taps

Radio rows are easy to tap repeatedly, especially on desktop or web. If `onChanged` calls `setState` even when the value is already selected, the keyed animation can replay with no actual state change. The early return in the sample prevents that. It also avoids extra rebuild work when a user clicks the active choice again.

For long preference screens, keep the group in a small widget instead of rebuilding the entire page. In my tests, the visible difference showed up around 8 to 10 settings rows: rebuilding the page made app bars, dividers, and unrelated toggles participate in the frame, even when only one radio value changed. Keeping the picker isolated made the animation feel tied to the tap.

## Practical checklist

- Keep `RadioListTile.value` and `groupValue` as the source of truth.
- Key the animated cue with both option identity and selected state.
- Reserve space for icons, badges, or labels that appear only when selected.
- Avoid slide or scale on the full tile unless the list has only a few rows.
- Skip the animation when the selected value is tapped again.

`flutter_animate` should clarify the single-choice decision, not decorate every rebuild. For `RadioListTile`, the best result usually comes from leaving the list geometry stable and animating one confirmation cue on the row that actually changed.
