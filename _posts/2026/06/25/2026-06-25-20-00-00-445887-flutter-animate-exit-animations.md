---
layout: post
title: "flutter_animate Exit Animations — Animating Widgets Out Before They Disappear"
description: "Learn how to play exit animations in flutter_animate before a widget is removed from the tree. Covers the delayed-removal pattern, AnimatedSwitcher combo, and a dismissible notification card example."
date: 2026-06-25
tags: [flutter_animate, animation, AnimatedSwitcher, Flutter, state_management]
comments: true
share: true
---

# flutter_animate Exit Animations — Animating Widgets Out Before They Disappear

![Flutter widget exit animation dismiss card effect](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

Exit animations are the awkward stepchild of Flutter. Entrance effects are trivial — widget builds, you chain `.animate()`, done. But remove a widget from the tree and it's just *gone*, no chance for a fade-out or slide-away. flutter_animate doesn't solve this automatically. You have to be deliberate about it.

Here's the pattern that actually works.

## Why Exit Animations Are Hard

Flutter's widget tree is declarative. When `_visible == false`, the widget is gone — immediately, frame-perfect. There's no lifecycle hook between "I want this widget gone" and "it's actually gone."

```dart
// This does NOT give you an exit animation
if (_visible)
  Text("Hello").animate().fadeIn()
```

The moment `_visible` flips to `false`, Flutter removes the widget before any animation can run. You need to keep the widget alive long enough to play its exit, then remove it.

## The Delayed-Removal Pattern

The trick is to split the "I want this gone" signal from the "actually remove it" action.

```dart
class _ExitDemoState extends State<ExitDemo> {
  bool _visible = true;
  bool _removing = false;

  void _dismiss() {
    setState(() => _removing = true);
    // wait for animation to finish, then actually remove
    Future.delayed(const Duration(milliseconds: 400), () {
      if (mounted) setState(() => _visible = false);
    });
  }

  @override
  Widget build(BuildContext context) {
    if (!_visible) return const SizedBox.shrink();

    return GestureDetector(
      onTap: _dismiss,
      child: Container(
        color: Colors.blue,
        padding: const EdgeInsets.all(16),
        child: const Text("Tap to dismiss"),
      )
      .animate(target: _removing ? 1.0 : 0.0)
      .fadeOut(duration: 300.ms)
      .slideY(begin: 0, end: -0.3, duration: 300.ms, curve: Curves.easeIn),
    );
  }
}
```

`_removing = true` triggers the animation (target flips to 1.0). The `Future.delayed` removes the widget after the animation finishes. The 400ms delay gives the 300ms animation a small buffer — don't cut it too tight or you get a flicker.

## AnimatedSwitcher + flutter_animate

For cases where you're swapping content (show/hide a panel, switch between states), `AnimatedSwitcher` is cleaner because it handles the removal timing itself. You can inject flutter_animate effects as the transition builder.

```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 350),
  transitionBuilder: (child, animation) {
    return FadeTransition(
      opacity: animation,
      child: SlideTransition(
        position: Tween<Offset>(
          begin: const Offset(0, 0.15),
          end: Offset.zero,
        ).animate(CurvedAnimation(parent: animation, curve: Curves.easeOut)),
        child: child,
      ),
    );
  },
  child: _showPanel
      ? const _PanelContent(key: ValueKey('panel'))
      : const SizedBox.shrink(key: ValueKey('empty')),
)
```

The key here is `ValueKey` — AnimatedSwitcher detects the swap by key change, not by widget type. Forget the key and it won't animate.

## Dismissible Notification Card — Full Example

Here's something more realistic: a notification card that slides up and fades out when dismissed.

![Flutter dismissible notification card with exit animation](https://images.unsplash.com/photo-1611532736597-de2d4265fba3?w=800&q=80)

```dart
class NotificationCard extends StatefulWidget {
  final String message;
  final VoidCallback onDismissed;

  const NotificationCard({
    super.key,
    required this.message,
    required this.onDismissed,
  });

  @override
  State<NotificationCard> createState() => _NotificationCardState();
}

class _NotificationCardState extends State<NotificationCard> {
  bool _exiting = false;

  void _handleDismiss() {
    if (_exiting) return;
    setState(() => _exiting = true);
    Future.delayed(350.ms, widget.onDismissed);
  }

  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        title: Text(widget.message),
        trailing: IconButton(
          icon: const Icon(Icons.close),
          onPressed: _handleDismiss,
        ),
      ),
    )
    .animate(target: _exiting ? 1.0 : 0.0)
    .fade(begin: 1, end: 0, duration: 300.ms)
    .slideY(begin: 0, end: -0.2, duration: 300.ms, curve: Curves.easeIn);
  }
}
```

Usage in the parent:

```dart
class _ParentState extends State<Parent> {
  bool _showNotification = true;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (_showNotification)
          NotificationCard(
            message: "Update available",
            onDismissed: () => setState(() => _showNotification = false),
          ),
      ],
    );
  }
}
```

The parent doesn't care about animation timing — it just responds to `onDismissed`. Clean separation.

## The `onComplete` Callback Approach

flutter_animate's `Animate` widget has an `onComplete` callback. You can use this instead of `Future.delayed` — it fires exactly when the animation finishes.

```dart
.animate(
  target: _exiting ? 1.0 : 0.0,
  onComplete: (controller) {
    if (_exiting && mounted) {
      widget.onDismissed();
    }
  },
)
```

This is slightly more precise than a hardcoded delay, but watch out: `onComplete` fires for *both* forward and reverse completion. Check `_exiting` inside the callback or you'll call `onDismissed` on the entrance animation too.

## Watch Out: `target` vs `play`/`reverse`

Using `target` is the right call for exit animations because it's stateless from the animation's perspective — you just flip a value and it moves. If you were using `controller.reverse()` manually (from the [gesture-driven post]({% post_url 2026-06-25-18-00-00-433542-flutter-animate-gesture-driven-animations %})), you'd call `controller.reverse()` then wait for `controller.status == AnimationStatus.dismissed` before removing the widget.

Both approaches work. `target` is simpler for most exit cases.

---

Next up: looping animations with `repeat` — pulse effects, loading spinners, breathing indicators, and how to stop them cleanly.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
- [AnimatedSwitcher docs](https://api.flutter.dev/flutter/widgets/AnimatedSwitcher-class.html)
