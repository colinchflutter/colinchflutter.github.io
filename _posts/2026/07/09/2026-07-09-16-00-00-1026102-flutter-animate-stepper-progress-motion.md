---
layout: post
title: "flutter_animate Stepper Progress Motion - form progress without layout jumps"
description: "Use flutter_animate with Flutter Stepper to animate progress feedback, active content, and validation states without layout jumps."
date: 2026-07-09
tags: [flutter_animate, animation, Flutter, UX]
comments: true
share: true
---

![flutter_animate Stepper progress feedback](https://images.unsplash.com/photo-1454165804606-c3d57bc86b40?w=800&q=80)
Watch the current step summary and controls, not the whole multi-step form.

`flutter_animate` works well with Flutter's `Stepper` when the motion is attached to progress feedback instead of the entire widget. Keep the step headers, connector lines, and form fields stable. Animate the active content panel, completion badge, validation hint, or continue button so the user sees what changed without losing their place in a long form.

The first version I tried wrapped the full `Stepper` in `animate().fadeIn().slideY()`. It looked harmless with 3 short steps. In a checkout form with address fields, every validation rebuild replayed the whole control and pushed the scroll position by a few pixels. The real fix was smaller: key the current step body and animate only the part that changes.

| Stepper area | Good motion | Avoid |
| --- | --- | --- |
| Step header | completed icon pop once | scaling every header on rebuild |
| Active content | 6-10 px slide + fade | animating inactive step content |
| Validation | short color/opacity cue | shaking while the user types |
| Controls | continue button enable fade | delaying step navigation |

Here is the compact pattern. The `Stepper` owns navigation, while `flutter_animate` decorates the active step content after the index changes.

```dart
class AnimatedCheckoutStepper extends StatefulWidget {
  const AnimatedCheckoutStepper({super.key});

  @override
  State<AnimatedCheckoutStepper> createState() =>
      _AnimatedCheckoutStepperState();
}

class _AnimatedCheckoutStepperState extends State<AnimatedCheckoutStepper> {
  int _step = 0;
  final _valid = <bool>[true, false, false];

  void _continue() {
    if (_step < 2) {
      setState(() => _step++);
    }
  }

  void _cancel() {
    if (_step > 0) {
      setState(() => _step--);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Stepper(
      currentStep: _step,
      onStepContinue: _valid[_step] ? _continue : null,
      onStepCancel: _cancel,
      controlsBuilder: (context, details) {
        return Row(
          children: [
            FilledButton(
              onPressed: details.onStepContinue,
              child: Text(_step == 2 ? 'Place order' : 'Continue'),
            )
                .animate(target: _valid[_step] ? 1 : 0)
                .fade(begin: .55, end: 1, duration: 120.ms),
            const SizedBox(width: 8),
            TextButton(
              onPressed: details.onStepCancel,
              child: const Text('Back'),
            ),
          ],
        );
      },
      steps: [
        _stepTile(0, 'Cart', 'Confirm items and quantity'),
        _stepTile(1, 'Delivery', 'Add address and delivery window'),
        _stepTile(2, 'Payment', 'Choose payment method'),
      ],
    );
  }

  Step _stepTile(int index, String title, String body) {
    final active = _step == index;
    final complete = index < _step;

    return Step(
      title: Text(title),
      isActive: active,
      state: complete ? StepState.complete : StepState.indexed,
      content: AnimatedSwitcher(
        duration: 180.ms,
        child: active
            ? _StepBody(
                key: ValueKey(index),
                body: body,
                valid: _valid[index],
              )
                  .animate()
                  .fadeIn(duration: 140.ms)
                  .slideY(begin: .08, end: 0, duration: 180.ms)
            : const SizedBox.shrink(),
      ),
    );
  }
}

class _StepBody extends StatelessWidget {
  const _StepBody({
    super.key,
    required this.body,
    required this.valid,
  });

  final String body;
  final bool valid;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(body),
        const SizedBox(height: 8),
        Text(
          valid ? 'Ready to continue' : 'Complete this section',
          style: TextStyle(
            color: valid ? Colors.green : Theme.of(context).colorScheme.error,
          ),
        )
            .animate(target: valid ? 1 : 0)
            .fade(begin: .65, end: 1, duration: 120.ms),
      ],
    );
  }
}
```

Two details matter more than the exact curve. Use `ValueKey(index)` so only a real step change replays the content animation. Keep the slide distance small; 8% of the child height is enough for a direction cue, while 30% starts to feel like a route transition inside the form.

For production forms, I also avoid animating after every keystroke. Validation can update live, but the motion should usually wait for blur, submit, or step change. That keeps accessibility settings and keyboard users from fighting a moving target.

Checklist I use before shipping:

- The focused input does not move while typing.
- The active step changes within about 180 ms.
- Back navigation does not replay completed badges endlessly.
- `onStepContinue` is disabled immediately when the step is invalid.
- The layout stays stable on narrow screens with `StepperType.vertical`.

The useful rule is simple: animate the evidence of progress, not the structure that holds the form. `Stepper` already explains sequence. `flutter_animate` should make the current decision feel acknowledged without making the whole workflow bounce.
