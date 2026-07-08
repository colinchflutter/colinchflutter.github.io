---
layout: post
title: "flutter_animate Tooltip Target Motion - helpful hints without hover noise"
description: "Use flutter_animate with Flutter Tooltip targets to show focus and hover feedback without animating the overlay itself."
date: 2026-07-09
tags: [flutter_animate, animation, Flutter, Web, Desktop]
comments: true
share: true
---

![Flutter Tooltip target motion with flutter_animate](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

This image is a reminder that tooltip motion belongs near the control the user is inspecting, not in a detached overlay.

`flutter_animate` is useful around `Tooltip` when it animates the trigger state, not the tooltip bubble. The practical pattern is to make the icon, help chip, or small action target react to hover and keyboard focus while Flutter keeps the semantic tooltip behavior intact.

The trap is trying to rebuild a custom tooltip overlay just to get a fade or slide. I have tried that for dense desktop toolbars. It worked for the mouse path, then broke keyboard discoverability and made long-press behavior on mobile inconsistent. A tooltip is already timing-sensitive; adding another animated layer often creates a hint that arrives late or follows the pointer too aggressively.

| Layer | Good job | Bad job |
| --- | --- | --- |
| `Tooltip` | message, wait duration, semantics | decorative motion |
| Target widget | hover, focus, pressed feedback | replacing accessibility behavior |
| `flutter_animate` | subtle scale, tint, glow | moving the overlay bubble |

Here is a compact target wrapper for icon buttons in a desktop or web toolbar. The animation is tied to local interaction state, so hovering one control does not replay the whole row.

```dart
class AnimatedTooltipTarget extends StatefulWidget {
  const AnimatedTooltipTarget({
    super.key,
    required this.message,
    required this.icon,
    required this.onPressed,
  });

  final String message;
  final IconData icon;
  final VoidCallback onPressed;

  @override
  State<AnimatedTooltipTarget> createState() => _AnimatedTooltipTargetState();
}

class _AnimatedTooltipTargetState extends State<AnimatedTooltipTarget> {
  bool _active = false;

  void _setActive(bool value) {
    if (_active != value) setState(() => _active = value);
  }

  @override
  Widget build(BuildContext context) {
    final color = Theme.of(context).colorScheme.primary;

    return FocusableActionDetector(
      onShowHoverHighlight: _setActive,
      onShowFocusHighlight: _setActive,
      child: Tooltip(
        message: widget.message,
        waitDuration: const Duration(milliseconds: 500),
        child: IconButton(
          icon: Icon(widget.icon),
          onPressed: widget.onPressed,
        )
            .animate(target: _active ? 1 : 0)
            .scale(
              begin: const Offset(1, 1),
              end: const Offset(1.08, 1.08),
              duration: 120.ms,
              curve: Curves.easeOut,
            )
            .tint(color: color.withOpacity(0.10), duration: 120.ms),
      ),
    );
  }
}
```

The important detail is `animate(target: _active ? 1 : 0)`. It gives the target two stable visual states instead of replaying an entrance animation on every parent rebuild. In a toolbar with 8 to 12 controls, that difference is visible. The user should see which item is inspectable, not a wave of icons moving because one provider emitted a new value.

I keep the scale under about 8 percent. Larger values make compact controls feel like they are stealing layout space, even when the actual box size stays fixed. For destructive actions, I usually skip scale and use a mild tint only; a bouncing delete icon reads like encouragement, which is the wrong signal.

For touch-heavy screens, this pattern should be quieter. `Tooltip` can still support long press, but hover feedback does not matter on most phones. In that case, gate the animated wrapper behind platform or pointer-mode checks and leave the plain `Tooltip` for mobile rows.

```dart
Widget buildToolbarButton(BuildContext context) {
  final isDesktopWidth = MediaQuery.sizeOf(context).width >= 720;

  if (!isDesktopWidth) {
    return Tooltip(
      message: 'Refresh data',
      child: IconButton(
        icon: const Icon(Icons.refresh),
        onPressed: refreshData,
      ),
    );
  }

  return AnimatedTooltipTarget(
    message: 'Refresh data',
    icon: Icons.refresh,
    onPressed: refreshData,
  );
}
```

The rule I use is simple: let `Tooltip` own the hint, let `IconButton` own the action, and let `flutter_animate` mark the target as active. That keeps the UI responsive on desktop without sacrificing keyboard focus, screen reader labels, or mobile long-press behavior.
