---
layout: post
title: "flutter_animate TextField Validation Motion - form feedback without noisy shaking"
description: "Use flutter_animate with Flutter TextField validation to show focus, error, and submit feedback without distracting form motion."
date: 2026-07-03
tags: [flutter_animate, animation, Flutter, UI, state_management]
comments: true
share: true
---

![Flutter TextField validation motion on a mobile form](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

This image points at the real problem: form feedback has to be visible while the keyboard, cursor, and validation text are already competing for attention.

The best use of `flutter_animate` in a Flutter `TextField` form is not a dramatic shake on every invalid input. Use motion to mark the field that changed state, keep the `Form` and `TextFormField` validation logic normal, and animate only the error row, helper row, or submit result. The form stays predictable, while the user still gets a clear signal.

The mistake I made first was adding a shake effect to the entire field whenever validation failed. It worked for one empty email field. It felt bad once the screen had 5 fields, the keyboard was open, and a password manager banner shifted the viewport. The field moved, the error text appeared, and the scroll position adjusted at the same time. That was three movements for one problem.

## Pick the motion target

The field border, label, error text, and submit button all have different jobs. Animating all of them makes the form look nervous. I usually pick one motion target per state change.

| Form state | Better target | Motion budget |
| --- | --- | --- |
| Focus gained | helper text or suffix icon | 120-180 ms fade |
| Validation failed | error row | 160-220 ms fade and tiny slide |
| Submit in progress | button child | spinner swap, no bounce |
| Submit succeeded | inline success row | 180 ms fade and scale from 0.98 |

That split keeps validation readable. The `TextField` itself should remain easy to type into; the animated layer should explain what changed.

## Keep validation ordinary

Start with a regular `Form`. The key detail is that animation state follows validation state, not the other way around.

```dart
final formKey = GlobalKey<FormState>();
final emailController = TextEditingController();
String? emailError;

void validateEmail() {
  final value = emailController.text.trim();

  setState(() {
    if (value.isEmpty) {
      emailError = 'Email is required';
    } else if (!value.contains('@')) {
      emailError = 'Enter a valid email address';
    } else {
      emailError = null;
    }
  });
}
```

I avoid hiding this logic inside the animation chain. Validation needs to be testable without frames, timers, or controller lifecycle concerns. The animation can be deleted later and the form should still work.

## Animate the error row

Instead of shaking the whole `TextField`, render an error row below it when the message exists. A small vertical reveal is enough.

```dart
Column(
  crossAxisAlignment: CrossAxisAlignment.start,
  children: [
    TextField(
      controller: emailController,
      keyboardType: TextInputType.emailAddress,
      decoration: InputDecoration(
        labelText: 'Email',
        suffixIcon: emailError == null
            ? const Icon(Icons.check_circle_outline)
            : const Icon(Icons.error_outline),
      ),
      onChanged: (_) => validateEmail(),
    ),
    if (emailError != null)
      Padding(
        padding: const EdgeInsets.only(top: 6),
        child: Text(
          emailError!,
          style: const TextStyle(color: Colors.redAccent),
        )
            .animate(key: ValueKey(emailError))
            .fadeIn(duration: 160.ms)
            .slideY(begin: -0.18, end: 0, curve: Curves.easeOut),
      ),
  ],
);
```

The `ValueKey(emailError)` matters. Without it, changing from `Email is required` to `Enter a valid email address` can reuse the same child and skip the entrance. That is fine for some apps, but in dense forms I want the new message to visibly replace the old one.

## Do not replay on every keystroke

There is one trap in the previous snippet: `onChanged` can validate on every character. If the message stays the same, the key stays the same, so the animation does not replay. If you key by `emailController.text`, the error would animate on every keystroke and become distracting.

For submit-only validation, the state is even cleaner.

```dart
bool submitted = false;

void submit() {
  setState(() => submitted = true);
  validateEmail();

  if (emailError != null) return;

  // Continue with the real submit action.
}
```

Then show error motion only after the user tries to submit. That avoids punishing someone while they are still typing the first character.

## Use focus motion carefully

Focus feedback should be quieter than error feedback. A suffix icon or helper text fade is usually enough.

```dart
Focus(
  onFocusChange: (focused) => setState(() => isEmailFocused = focused),
  child: TextField(
    controller: emailController,
    decoration: InputDecoration(
      labelText: 'Email',
      helperText: isEmailFocused ? 'Use the address for receipts' : null,
    ),
  ),
);
```

If the helper text appears and disappears often, reserve its vertical space with a fixed-height wrapper. Otherwise the field below it will jump every time focus changes, and the animation will make the layout shift more obvious.

## Practical checklist

- Keep `TextField` input behavior and `Form` validation independent from animation.
- Animate the error or helper row, not the whole field container.
- Key error animations by the message or field id, not by the current text.
- Reserve space for helper text when focus changes can happen repeatedly.
- Avoid horizontal shake on multi-field mobile forms unless the error has no text.

`flutter_animate` works best here as a feedback layer. The validation model decides what is wrong, Flutter renders a stable input, and the animation gives the changed message just enough movement to be noticed without turning typing into a moving target.
