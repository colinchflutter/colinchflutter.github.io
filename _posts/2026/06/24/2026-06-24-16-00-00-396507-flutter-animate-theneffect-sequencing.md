---
layout: post
title: "flutter_animate ThenEffect — chaining animations without the delay math headache"
description: "Use ThenEffect to chain flutter_animate effects sequentially. No timer arithmetic, no AnimationControllers — just clean sequential animations that stay in sync when durations change."
date: 2026-06-24
tags: [flutter_animate, animation, Flutter, Dart]
comments: true
share: true
---

# flutter_animate ThenEffect — chaining animations without the delay math headache

![Sequential UI animation transitions on a mobile device](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

Short answer: `ThenEffect` is a zero-visual marker that pins the timeline position. Any effect after it starts from that marker, not from zero. When you change one duration, everything downstream shifts automatically — no recalculating delays by hand.

## The delay arithmetic trap

Before I landed on `ThenEffect`, chaining effects looked like this:

```dart
MyWidget()
  .animate()
  .fadeIn(duration: 300.ms)
  .scale(delay: 300.ms, duration: 400.ms)
  .slideY(delay: 700.ms, duration: 500.ms)  // 300+400, kept in my head
```

This works until you change the fade duration. Now `scale` is wrong, and so is `slideY`. I walked into this on an onboarding screen — changed the fade from 300ms to 500ms and the rest of the sequence ran out of order for an hour before I caught it.

## How ThenEffect fixes this

`then()` snaps the cursor to wherever the previous effect ended. The next effect's `delay` is relative to that snap point, not to `t=0`.

```dart
MyWidget()
  .animate()
  .fadeIn(duration: 300.ms)
  .then()                         // cursor: t=300ms
  .scale(duration: 400.ms)       // runs from t=300ms to t=700ms
  .then()                         // cursor: t=700ms
  .slideY(duration: 500.ms)      // runs from t=700ms to t=1200ms
```

Change `fadeIn` to 500ms — `scale` and `slideY` shift automatically. Nothing to recalculate.

## Adding a pause between effects

`then()` takes an optional `delay` that adds breathing room after the previous effect ends:

```dart
MyWidget()
  .animate()
  .fadeIn(duration: 400.ms)
  .then(delay: 150.ms)            // 150ms pause after fade
  .scale(duration: 300.ms)
  .then(delay: 100.ms)
  .shimmer(duration: 600.ms)
```

That 150ms pause after the fade makes the sequence feel intentional rather than robotic. I use this on success confirmation widgets — fade in, brief pause, then a shimmer pass. Without the pause, the shimmer starts too immediately and the user doesn't register the fade.

## Onboarding reveal pattern

Here's a real pattern: title appears first, description second, button third. Three separate widgets in a `Column`, so I use `animate(delay:)` on each rather than ThenEffect (which works on effects stacked on a single widget):

![Mobile onboarding screen with staggered text and button reveal](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,
  children: [
    Text('Get things done', style: Theme.of(context).textTheme.displaySmall)
      .animate()
      .fadeIn(duration: 500.ms)
      .slideY(begin: -0.15, end: 0, duration: 500.ms),

    const SizedBox(height: 20),

    Text('Track tasks, set reminders, stay focused.')
      .animate(delay: 600.ms)
      .fadeIn(duration: 400.ms)
      .slideY(begin: 0.1, end: 0, duration: 400.ms),

    const SizedBox(height: 48),

    FilledButton(
      onPressed: () {},
      child: const Text('Start now'),
    )
      .animate(delay: 1100.ms)
      .fadeIn(duration: 300.ms)
      .scale(begin: const Offset(0.85, 0.85), duration: 300.ms),
  ],
)
```

The `delay` on the outer `animate()` is a one-shot offset. It's different from `ThenEffect` — ThenEffect is for sequencing effects *within* a single `.animate()` chain, while `animate(delay:)` controls when that widget's whole animation starts relative to mount.

[The staggered list approach from the previous post]({% post_url 2026-06-24-10-00-00-371817-flutter-animate-staggered-list %}) uses a similar pattern with `AnimateList` — worth reading alongside this if you're building list-entry sequences.

## ThenEffect inside a repeating animation

`then()` works fine inside `onPlay: (c) => c.repeat()`:

```dart
Icon(Icons.sync)
  .animate(onPlay: (c) => c.repeat())
  .rotate(duration: 600.ms)
  .then(delay: 200.ms)
  .fadeOut(duration: 300.ms)
  .then()
  .fadeIn(duration: 300.ms)
```

The whole sequence loops: rotate → 200ms pause → fadeOut → fadeIn → rotate → ...

One thing that tripped me up: if you also set `reverse: true` on the Animate, ThenEffect markers are respected in reverse too. The timeline plays forward, then backward through the same markers. Usually fine, but if your sequence has a one-directional feel (like a loader), skip `reverse` and just loop forward.

## Debugging tips

Two things I check when a sequence looks wrong:

**1. Verify durations add up.** If an effect has `duration: 0.ms` (which is the default for some effects when you forget to set it), `then()` snaps to that effect's end — which is the same position as its start. Easy to accidentally collapse a step.

**2. Enable hot-reload restart during development.** Add this at the top of `main()`:

```dart
void main() {
  Animate.restartOnHotReload = true; // remove before release
  runApp(const MyApp());
}
```

Without it, hot-reloading mid-sequence leaves animations in whatever state they were in. With it, every hot reload restarts all animations from zero — much easier to iterate on timing.

---

Next up: `repeat`, `loop`, and `autoPlay: false` — controlling when and how often a sequence plays beyond the initial mount.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate GitHub — ThenEffect](https://github.com/gskinner/flutter_animate#theneffect)
