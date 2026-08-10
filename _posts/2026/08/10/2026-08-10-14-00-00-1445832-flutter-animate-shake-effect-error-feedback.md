---
layout: post
title: "flutter_animate ShakeEffect - Add Flutter Error Feedback Without Manual Tween Code"
description: "Learn how to use flutter_animate ShakeEffect for Flutter form errors, including timing, axis control, replay keys, and accessible motion limits."
date: 2026-08-10
tags: [flutter_animate, animation, Flutter, performance, accessibility]
comments: true
share: true
---

![Flutter form error feedback using flutter_animate ShakeEffect](/assets/images/flutter-animate-shake-effect-error-feedback.png)

`flutter_animate`'s `ShakeEffect` is a compact way to tell a Flutter user that an action was rejected. It works well for an invalid form field, a wrong PIN, or a failed drag target because the widget moves briefly and returns to its original position. The useful part is not making something shake. It is keeping the movement short, replayable, and small enough that it communicates an error instead of feeling like a broken layout.

## The smallest useful ShakeEffect

The effect can be added to any widget that already exists in the tree. This example shakes the whole form when validation fails:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class LoginForm extends StatefulWidget {
  const LoginForm({super.key});

  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  int _shakeVersion = 0;

  void _submit() {
    if (_formKey.currentState?.validate() ?? false) return;

    setState(() => _shakeVersion++);
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            validator: (value) =>
                value == null || value.isEmpty ? 'Email is required' : null,
          ),
          const SizedBox(height: 16),
          ElevatedButton(onPressed: _submit, child: const Text('Sign in')),
        ],
      ),
    ).animate(key: ValueKey(_shakeVersion)).shake(
          hz: 4,
          offset: const Offset(8, 0),
          duration: 450.ms,
        );
  }
}
```

`hz` controls how many side-to-side oscillations happen during the effect. `offset` controls the maximum displacement, not the final position. The widget settles back automatically, so there is no controller to reset after validation.

## Why the ValueKey matters

I initially called `setState` after validation and expected the same chained animation to replay. It did not. The widget rebuilt, but its animation state was still considered the same animation. Incrementing `_shakeVersion` changes the `ValueKey`, which gives the animated widget a fresh identity and starts the effect again.

This distinction matters when the user presses Submit twice. Without a changing key, the first error may animate correctly and every later press may appear to do nothing. If the field itself should shake, put the key and effect on that field instead of the entire `Form`:

```dart
TextFormField(
  validator: (value) => value == null || value.isEmpty ? 'Required' : null,
).animate(key: ValueKey('email-$_shakeVersion')).shake(
      hz: 4,
      offset: const Offset(8, 0),
      duration: 450.ms,
    );
```

Use the same key strategy consistently for the installed `flutter_animate` version. The chained `shake` call is enough for most cases; a direct `ShakeEffect` is useful when building a reusable effect list with other `Effect` objects.

## Choosing motion that supports the message

| Situation | Starting values | Reason |
| --- | --- | --- |
| Inline field error | `hz: 3`, `offset: 6` | Keeps nearby labels stable |
| Login or PIN rejection | `hz: 4`, `offset: 8` | Noticeable without looking playful |
| Drop-zone rejection | `hz: 5`, `offset: 10` | Faster feedback for a gesture |

Avoid shaking a large page or a full navigation shell. A moving parent can make the error text, focus ring, and keyboard feel disconnected. Anchor the effect to the smallest widget that owns the failed action.

## Two traps worth checking

First, do not use a long repeat count as a substitute for a larger error message. Repeated movement becomes noisy, especially on iOS where users may already have motion sensitivity enabled. Respect `MediaQuery.disableAnimations` and provide visible validation text as the primary explanation.

Second, a shake does not reserve extra layout space. If an ancestor clips its child, a large horizontal offset can be cut off at the edge. Test the effect inside dialogs and narrow list tiles, not only in a wide emulator screen.

`ShakeEffect` is best treated as a momentary signal. Pair it with a stable error label, focus the invalid field, and keep the animation below half a second. That combination gives the user three things the animation alone cannot: what failed, where to fix it, and confirmation that the UI has finished reacting.
