---
layout: post
title: "flutter_animate Reusable Presets - keeping animation chains consistent"
description: "Build reusable flutter_animate presets with Dart extensions so repeated fade, slide, and scale chains stay consistent across screens."
date: 2026-07-01
tags: [flutter_animate, animation, performance, state_management]
comments: true
share: true
---
![Flutter code editor showing reusable animation presets for a mobile app](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=800&q=80)

Short answer: once the same `fadeIn().slideY().scale()` chain appears in three places, I move it into a small Dart extension. `flutter_animate` is readable inline, but repeated chains drift fast when one screen uses `220.ms`, another uses `260.ms`, and a third forgets the curve.

The problem shows up late. A settings card, empty state, and dialog all start with the same entrance motion. During review, someone asks for the entrance to feel "less jumpy." If the animation is copied across widgets, you now have a search-and-edit job. I have missed one of those copies before, and the app felt oddly inconsistent even though every individual animation looked fine in isolation.

## Create a preset extension

Keep the preset close to your design system, not inside a random feature screen. I usually put this in `animation_presets.dart`.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

extension AppAnimatePresets on Widget {
  Widget appEntrance({Duration? delay}) {
    return animate(delay: delay)
        .fadeIn(duration: 220.ms, curve: Curves.easeOut)
        .slideY(begin: 0.08, end: 0, duration: 260.ms, curve: Curves.easeOutCubic)
        .scale(begin: const Offset(0.98, 0.98), end: const Offset(1, 1));
  }

  Widget appPressFeedback({required bool pressed}) {
    return animate(target: pressed ? 1 : 0)
        .scale(
          begin: const Offset(1, 1),
          end: const Offset(0.97, 0.97),
          duration: 90.ms,
          curve: Curves.easeOut,
        );
  }
}
```

Then the widget code stays boring:

```dart
Column(
  children: [
    const Text('Storage cleaned').appEntrance(),
    const SizedBox(height: 12),
    FilledButton(
      onPressed: onDone,
      child: const Text('Done'),
    ).appEntrance(delay: 80.ms),
  ],
)
```

The important detail is the return type: `Widget`, not `Animate`. The caller should not care whether the preset uses one effect or five. If you expose the chain internals, every call site starts customizing it and the preset stops being a preset.

## Keep presets parameter-light

I only pass values that genuinely vary by context: `delay`, `pressed`, `enabled`, sometimes `distance`. I do not pass every duration and curve. That turns a preset into a wrapper around the package API, which adds indirection without adding consistency.

For list items, I still let the caller own the stagger:

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(title: Text(items[index].title))
        .appEntrance(delay: (index * 45).ms);
  },
)
```

One caveat: avoid hiding business state inside the extension. A preset can animate based on a `pressed` or `selected` boolean, but it should not read providers, start network work, or decide navigation. The extension is a visual contract. The feature still owns the state.

Short version: use inline `flutter_animate` chains while exploring. Promote them to extension presets once they become shared UI language. Keep the parameters few, return a plain `Widget`, and let feature code provide state instead of burying behavior inside animation helpers.
