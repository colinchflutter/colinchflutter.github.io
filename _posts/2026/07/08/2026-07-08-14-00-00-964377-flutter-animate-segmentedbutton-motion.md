---
layout: post
title: "flutter_animate SegmentedButton Motion - selected states without control jitter"
description: "Use flutter_animate with Flutter SegmentedButton to highlight selected states without replaying the whole control."
date: 2026-07-08
tags: [flutter_animate, animation, state_management, Flutter, performance]
comments: true
share: true
---
![flutter_animate SegmentedButton selection motion](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
The useful detail in this image is the grouped choice surface: a segmented control should make the active option obvious without making nearby options jump.

`flutter_animate` works well with `SegmentedButton` when the motion is tied to the committed selection, not to every child in the control. Keep the button group layout stable, then animate a small selected-state detail such as the icon, label weight, or an adjacent summary. The mistake I hit was animating each segment as it rebuilt. With 3 segments it looked playful. With 5 segments and keyboard focus, it became noisy fast.

## Keep the segments stable

`SegmentedButton` already handles hit testing, disabled states, focus traversal, and selected styling. If the whole control fades or slides on every selection, the user sees movement in options they did not touch. A better pattern is to give the selected value a keyed, short animation while the rest of the row stays still.

```dart
enum ReportRange { day, week, month }

class ReportRangePicker extends StatefulWidget {
  const ReportRangePicker({super.key});

  @override
  State<ReportRangePicker> createState() => _ReportRangePickerState();
}

class _ReportRangePickerState extends State<ReportRangePicker> {
  ReportRange _range = ReportRange.week;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        SegmentedButton<ReportRange>(
          segments: const [
            ButtonSegment(
              value: ReportRange.day,
              label: Text('Day'),
              icon: Icon(Icons.today),
            ),
            ButtonSegment(
              value: ReportRange.week,
              label: Text('Week'),
              icon: Icon(Icons.view_week),
            ),
            ButtonSegment(
              value: ReportRange.month,
              label: Text('Month'),
              icon: Icon(Icons.calendar_month),
            ),
          ],
          selected: {_range},
          onSelectionChanged: (values) {
            setState(() => _range = values.first);
          },
        ),
        const SizedBox(height: 12),
        AnimatedSwitcher(
          duration: 180.ms,
          switchInCurve: Curves.easeOutCubic,
          switchOutCurve: Curves.easeInCubic,
          child: Text(
            'Showing ${_range.name} metrics',
            key: ValueKey(_range),
          )
              .animate(key: ValueKey(_range))
              .fadeIn(duration: 120.ms)
              .slideY(begin: .18, end: 0, duration: 180.ms),
        ),
      ],
    );
  }
}
```

The important part is the key. `ValueKey(_range)` makes the summary replay only when the selected range changes. The `SegmentedButton` itself is not animated, so width, focus ring, and pointer target positions stay predictable.

| Choice | Animate | Reason |
| --- | --- | --- |
| Whole `SegmentedButton` | No | It shifts every option after one tap. |
| Selected icon only | Sometimes | Good for compact filters, but keep it under 160 ms. |
| Summary text below | Yes | It explains the new state without touching the control. |
| Parent form section | No | It makes unrelated settings feel changed. |

## Avoid per-segment replays

It is tempting to wrap every `ButtonSegment` label in `.animate()`. That usually creates two problems. The first is timing: all labels replay when Flutter rebuilds the segment list. The second is measurement: scale or slide effects can make text look misaligned beside icons, especially on desktop where hover and focus states are visible.

For a dense toolbar, I use a smaller cue outside the control:

```dart
Row(
  children: [
    const Icon(Icons.check_circle, size: 18)
        .animate(key: ValueKey(_range))
        .scaleXY(begin: .82, end: 1, duration: 140.ms)
        .fadeIn(duration: 100.ms),
    const SizedBox(width: 8),
    Text('${_range.name} selected'),
  ],
)
```

This keeps the visual feedback close to the segmented choice, but it does not fight Material's own selected segment painting.

## Short checklist

- Use `SegmentedButton` for the selection mechanics.
- Animate a keyed summary, badge, or confirmation icon.
- Keep selection feedback around 120-180 ms.
- Do not animate every segment on rebuild.
- Test with keyboard focus and at least one long label before calling it done.

The practical rule is simple: the selected state can move, but the choice targets should not. That keeps `flutter_animate` helpful instead of turning a precise control into a moving one.
