---
layout: post
title: "flutter_animate Slider Value Motion - readable drag feedback without jitter"
description: "Use flutter_animate with Flutter Slider controls to show value changes clearly without jitter, rebuild noise, or distracting thumb motion."
date: 2026-07-07
tags: [flutter_animate, animation, performance, state_management]
comments: true
share: true
---

![Flutter slider value feedback on a mobile control panel](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1200&q=80)
The useful detail in this image is the compact control area: slider feedback should make the changed value easier to read, not make the whole panel feel busy.

`flutter_animate` is a good fit for `Slider` when the animated part is the value label, helper text, or affected preview. The trap is animating the thumb or the entire slider on every drag tick. A slider can emit dozens of updates in one second. If every update starts a new scale, slide, or fade sequence, the control feels late even when the state code is correct.

The cleaner pattern is to keep the `Slider` itself boring and animate only the output that benefits from attention. In my own tests, dragging from 20 to 80 felt clear when the number badge faded over 90 ms, but it felt broken when the whole row pulsed for each value change. That second version did not fail technically. It failed because the motion fought the user's finger.

| Slider change | Better animated target | Avoid |
|---|---|---|
| Live value drag | value badge opacity or color | scaling the thumb each tick |
| Committed value | preview card after `onChangeEnd` | replaying entrance motion while dragging |
| Disabled state | short fade to muted label | shaking unavailable controls |

Here is the shape I use for a settings slider. The active drag value updates immediately, while the heavier preview motion only runs after the user releases the thumb.

```dart
class BrightnessSlider extends StatefulWidget {
  const BrightnessSlider({super.key});

  @override
  State<BrightnessSlider> createState() => _BrightnessSliderState();
}

class _BrightnessSliderState extends State<BrightnessSlider> {
  double _value = 64;
  double _committed = 64;
  bool _dragging = false;

  @override
  Widget build(BuildContext context) {
    final rounded = _value.round();

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Row(
          children: [
            const Text('Brightness'),
            const Spacer(),
            AnimatedSwitcher(
              duration: 90.ms,
              child: Text(
                '$rounded%',
                key: ValueKey(rounded),
                style: Theme.of(context).textTheme.labelLarge,
              ).animate().fadeIn(duration: 90.ms),
            ),
          ],
        ),
        Slider(
          value: _value,
          min: 0,
          max: 100,
          divisions: 100,
          label: '$rounded%',
          onChangeStart: (_) => setState(() => _dragging = true),
          onChanged: (next) => setState(() => _value = next),
          onChangeEnd: (next) {
            setState(() {
              _dragging = false;
              _committed = next;
            });
          },
        ),
        _BrightnessPreview(
          value: _committed,
          dragging: _dragging,
        ),
      ],
    );
  }
}
```

The preview should react to the committed value, not every raw drag event. That keeps expensive child widgets from rebuilding like a progress ticker. It also gives the motion a clean meaning: the value has landed.

```dart
class _BrightnessPreview extends StatelessWidget {
  const _BrightnessPreview({
    required this.value,
    required this.dragging,
  });

  final double value;
  final bool dragging;

  @override
  Widget build(BuildContext context) {
    final opacity = 0.35 + (value / 100 * 0.65);

    return Container(
      height: 72,
      alignment: Alignment.center,
      decoration: BoxDecoration(
        color: Colors.amber.withOpacity(opacity),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(dragging ? 'Adjusting' : 'Saved ${value.round()}%'),
    )
        .animate(key: ValueKey(value.round()))
        .fadeIn(duration: 140.ms)
        .scale(
          begin: const Offset(.98, .98),
          end: const Offset(1, 1),
          duration: 140.ms,
          curve: Curves.easeOut,
        );
  }
}
```

The important key detail is intentional granularity. `ValueKey(value.round())` is acceptable for a divided slider because it limits animation restarts to visible value changes. For a continuous audio gain slider, I would not key on the raw double. I would either round to one decimal place or move the animation to `onChangeEnd`.

One more edge case: accessibility settings can make slider motion feel worse faster than button motion. If the app already has a reduced motion flag, branch the preview to a plain `AnimatedContainer` color change or skip `flutter_animate` for this control. A slider is already interactive feedback; animation is only useful when it clarifies the number, the committed state, or the preview result.

Short version: animate the readout, not the drag mechanics. Keep live feedback under 100 ms, reserve larger motion for `onChangeEnd`, and never key an animation from an unrounded continuous double unless jitter is the intended result.
