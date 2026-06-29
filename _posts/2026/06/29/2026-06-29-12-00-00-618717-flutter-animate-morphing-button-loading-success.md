---
layout: post
title: "flutter_animate Morphing Button — Loading Spinner to Success State"
description: "Build an animated submit button in Flutter that transitions from idle to loading spinner to success checkmark using flutter_animate — with a clean state machine, error state, and no extra animation controllers."
date: 2026-06-29
tags: [flutter_animate, animation, Flutter, state_management, performance]
comments: true
share: true
---

![Submit button on a mobile UI with glowing feedback state](https://images.unsplash.com/photo-1563986768494-4dee2763ff3f?w=800&q=80)

The animated submit button is one of those "should be simple" UI elements that turns into a rabbit hole fast. You want: idle → loading spinner → success checkmark (or error). The naive approach usually involves three separate widgets and a `setState` mess. Here's how to do it cleanly with `flutter_animate` — no extra `AnimationController`, no `TickerProvider` headaches.

## The state machine first

Before touching animation code, define the button states as an enum. This is non-negotiable — I've seen projects where three booleans (`_isLoading`, `_isSuccess`, `_hasError`) gradually become contradictory and impossible to reason about.

```dart
enum ButtonState { idle, loading, success, error }
```

The widget holds one `ButtonState` and derives everything from it. That's the whole trick.

## Basic structure

```dart
class MorphingButton extends StatefulWidget {
  final Future<void> Function() onPressed;
  final String label;

  const MorphingButton({
    super.key,
    required this.onPressed,
    required this.label,
  });

  @override
  State<MorphingButton> createState() => _MorphingButtonState();
}

class _MorphingButtonState extends State<MorphingButton> {
  ButtonState _state = ButtonState.idle;

  Future<void> _handlePress() async {
    if (_state != ButtonState.idle) return;
    setState(() => _state = ButtonState.loading);
    try {
      await widget.onPressed();
      setState(() => _state = ButtonState.success);
      // Auto-reset after 2 seconds
      await Future.delayed(const Duration(seconds: 2));
      if (mounted) setState(() => _state = ButtonState.idle);
    } catch (_) {
      setState(() => _state = ButtonState.error);
      await Future.delayed(const Duration(seconds: 2));
      if (mounted) setState(() => _state = ButtonState.idle);
    }
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handlePress,
      child: _ButtonContent(state: _state, label: widget.label),
    );
  }
}
```

The guard `if (_state != ButtonState.idle) return;` prevents double-taps during the loading phase. Simple and effective.

## The animated content widget

This is where flutter_animate does the heavy lifting. Each state renders a different child, and the `AnimatedSwitcher` + flutter_animate combo handles the transition.

```dart
class _ButtonContent extends StatelessWidget {
  final ButtonState state;
  final String label;

  const _ButtonContent({required this.state, required this.label});

  @override
  Widget build(BuildContext context) {
    final isLoading = state == ButtonState.loading;
    final isSuccess = state == ButtonState.success;
    final isError = state == ButtonState.error;

    final bgColor = isSuccess
        ? const Color(0xFF2ECC71)
        : isError
            ? const Color(0xFFE74C3C)
            : const Color(0xFF3498DB);

    return AnimatedContainer(
      duration: const Duration(milliseconds: 300),
      curve: Curves.easeInOut,
      width: (isLoading || isSuccess || isError) ? 56 : 200,
      height: 56,
      decoration: BoxDecoration(
        color: bgColor,
        borderRadius: BorderRadius.circular(28),
      ),
      child: ClipRRect(
        borderRadius: BorderRadius.circular(28),
        child: Center(
          child: AnimatedSwitcher(
            duration: const Duration(milliseconds: 250),
            child: _buildIcon(state, label),
          ),
        ),
      ),
    );
  }

  Widget _buildIcon(ButtonState state, String label) {
    switch (state) {
      case ButtonState.idle:
        return Text(
          label,
          key: const ValueKey('label'),
          style: const TextStyle(
            color: Colors.white,
            fontWeight: FontWeight.w600,
            fontSize: 16,
          ),
        );
      case ButtonState.loading:
        return const SizedBox(
          key: ValueKey('loading'),
          width: 24,
          height: 24,
          child: CircularProgressIndicator(
            color: Colors.white,
            strokeWidth: 2.5,
          ),
        );
      case ButtonState.success:
        return const Icon(
          Icons.check,
          key: ValueKey('success'),
          color: Colors.white,
          size: 28,
        )
            .animate()
            .scale(begin: const Offset(0, 0), duration: 300.ms, curve: Curves.elasticOut)
            .fadeIn(duration: 200.ms);
      case ButtonState.error:
        return const Icon(
          Icons.close,
          key: ValueKey('error'),
          color: Colors.white,
          size: 28,
        )
            .animate()
            .shake(hz: 4, offset: const Offset(4, 0), duration: 400.ms)
            .fadeIn(duration: 150.ms);
    }
  }
}
```

![Animated button states showing idle, loading spinner, green success, and red error](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

The `AnimatedContainer` handles the width morph (200px → 56px pill → back). flutter_animate handles the per-icon entrance. The two work independently, which means you can tune them separately without them fighting each other.

## Where I got stuck — the key trap

`AnimatedSwitcher` compares children by their `key`. If you forget to set unique keys, switching between the loading spinner and the success icon does nothing — both are `SizedBox` or `Icon` widgets that look the same to the framework.

```dart
// Wrong — AnimatedSwitcher sees no change
case ButtonState.loading:
  return const CircularProgressIndicator(color: Colors.white);
case ButtonState.success:
  return const Icon(Icons.check, color: Colors.white);

// Correct — key forces the switcher to rebuild
case ButtonState.loading:
  return const CircularProgressIndicator(
    key: ValueKey('loading'),  // <-- required
    color: Colors.white,
  );
case ButtonState.success:
  return const Icon(
    Icons.check,
    key: ValueKey('success'),  // <-- required
    color: Colors.white,
  );
```

Spent 20 minutes on this before spotting it. The keys have to be on the direct child of `AnimatedSwitcher`.

## Adding flutter_animate to the idle → loading transition

The `AnimatedContainer` width change is smooth, but the label text exit feels abrupt. Add a fade-out before the width collapses:

```dart
case ButtonState.idle:
  return Text(
    label,
    key: const ValueKey('label'),
    style: const TextStyle(
      color: Colors.white,
      fontWeight: FontWeight.w600,
      fontSize: 16,
    ),
  )
      .animate(key: ValueKey('label-anim-${state.name}'))
      .fadeIn(duration: 200.ms);
```

The re-keying on `state.name` forces flutter_animate to replay the entrance animation every time `_state` changes to idle. Without the key change, it only animates once on first build.

## Error state with shake

The `.shake()` effect on error is the reason flutter_animate earns its place here. Doing this manually means an `AnimationController`, a `Tween`, a `CurvedAnimation`, and 30 lines of boilerplate. With flutter_animate:

```dart
const Icon(Icons.close, color: Colors.white, size: 28)
    .animate()
    .shake(hz: 4, offset: const Offset(4, 0), duration: 400.ms)
    .fadeIn(duration: 150.ms);
```

`hz: 4` means 4 oscillations over the duration. `offset: Offset(4, 0)` is horizontal-only. Tweak `hz` between 3–6 for different feels — 3 is gentle, 6 starts to look frantic.

## Full usage example

```dart
MorphingButton(
  label: 'Submit',
  onPressed: () async {
    await Future.delayed(const Duration(seconds: 2)); // simulate API call
    // throw Exception('Server error'); // uncomment to test error state
  },
)
```

That's the entire consumer API. The state machine, animation, and auto-reset are encapsulated inside.

## Wrapping up

The pattern: `AnimatedContainer` for shape/size morphing, `AnimatedSwitcher` with keys for content swapping, and flutter_animate for per-icon entrance effects. They're separate concerns and don't step on each other.

Next up: building an animated bottom navigation bar where the active icon bounces and scales with flutter_animate — similar technique, but with persistent state across multiple icons.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [AnimatedSwitcher docs](https://api.flutter.dev/flutter/widgets/AnimatedSwitcher-class.html)
- [AnimatedContainer docs](https://api.flutter.dev/flutter/widgets/AnimatedContainer-class.html)
