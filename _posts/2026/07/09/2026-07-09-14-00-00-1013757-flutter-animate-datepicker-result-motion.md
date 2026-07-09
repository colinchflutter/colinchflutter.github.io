---
layout: post
title: "flutter_animate DatePicker Result Motion - clear date feedback without route noise"
description: "Use flutter_animate with Flutter showDatePicker to animate selected date feedback without fighting dialog routes or calendar layout."
date: 2026-07-09
tags: [flutter_animate, animation, widgets, UX]
comments: true
share: true
---

![flutter_animate DatePicker result feedback](https://images.unsplash.com/photo-1506784983877-45594efa4cbe?w=800&q=80)
Watch the chosen date summary and confirm action, not the calendar route itself.

`flutter_animate` works best with `showDatePicker` after the picker closes. Let Flutter own the dialog, focus trap, keyboard behavior, and platform transitions. Then animate the result row, calendar icon, helper copy, or submit button so the selected date feels acknowledged without making the date grid jump.

The mistake I hit first was wrapping the button that opens the picker in a repeating scale effect. It looked fine in a static demo, but after three quick edits the trigger became noisy and the selected value still felt unchanged. The useful motion belongs where state changed: the text that moved from "No date selected" to "Jul 18, 2026".

| Area | Good motion | Avoid |
| --- | --- | --- |
| Picker trigger | Tiny icon color or one-shot badge | Scaling the whole button on every tap |
| Selected date row | Fade + 6 px slide when value changes | Replaying the full form section |
| Validation hint | Short shake only after submit | Shaking while the picker is open |
| Confirm button | Enable-state fade or glow | Delaying the picker route close |

The pattern is simple: key the animated result by the selected date, not by the parent build. That keeps unrelated form rebuilds from replaying the date animation.

```dart
class BookingDateField extends StatefulWidget {
  const BookingDateField({super.key});

  @override
  State<BookingDateField> createState() => _BookingDateFieldState();
}

class _BookingDateFieldState extends State<BookingDateField> {
  DateTime? _date;

  Future<void> _pickDate() async {
    final picked = await showDatePicker(
      context: context,
      initialDate: _date ?? DateTime.now(),
      firstDate: DateTime(2026, 1, 1),
      lastDate: DateTime(2026, 12, 31),
    );

    if (picked == null || picked == _date) return;
    setState(() => _date = picked);
  }

  @override
  Widget build(BuildContext context) {
    final label = _date == null
        ? 'No date selected'
        : '${_date!.month}/${_date!.day}/${_date!.year}';

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        OutlinedButton.icon(
          onPressed: _pickDate,
          icon: const Icon(Icons.event),
          label: const Text('Choose date'),
        ),
        const SizedBox(height: 12),
        Row(
          key: ValueKey(label),
          children: [
            Icon(
              _date == null ? Icons.event_busy : Icons.event_available,
              color: _date == null ? Colors.grey : Colors.green,
            ),
            const SizedBox(width: 8),
            Text(label),
          ],
        )
            .animate()
            .fadeIn(duration: 140.ms)
            .slideY(begin: .18, end: 0, duration: 180.ms),
      ],
    );
  }
}
```

There are two details that keep this from feeling cheap. The animation is under 200 ms because the user already waited for a modal picker. The `ValueKey(label)` changes only when the date text changes, so theme updates, parent validation, and focus changes do not replay the row.

A date picker often sits inside a larger checkout or booking form, so the motion should answer one question: "Did the app accept the date I picked?" If the answer is visible in the summary row and the submit button state, the dialog can stay boring. That is usually the better trade.

Checklist for this pattern:

- Keep `showDatePicker` route transitions untouched.
- Animate the result, helper text, or enabled submit state.
- Key the animation with the selected value.
- Skip the animation when `picked == null` or the value did not change.
- Use short fade/slide effects instead of bouncing the whole form.
