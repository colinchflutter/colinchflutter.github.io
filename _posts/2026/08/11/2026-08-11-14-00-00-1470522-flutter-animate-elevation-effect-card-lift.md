---
layout: post
title: "flutter_animate ElevationEffect - Animate Flutter Card Lift Without BoxShadow Math"
description: "Use flutter_animate ElevationEffect to animate Flutter Material card elevation for focus and press states, with PhysicalModel and BoxShadow trade-offs explained."
date: 2026-08-11
tags: [flutter_animate, animation, Flutter, Material, performance]
comments: true
share: true
---

![flutter_animate ElevationEffect lifting a Flutter card with an animated Material shadow](https://colinchflutter.github.io/assets/images/flutter-animate-elevation-effect-card-lift.png)

*The visual difference to notice is not a larger card. It is the shadow changing as the card moves from a resting surface to a raised surface.*

`flutter_animate`'s `ElevationEffect` is a good fit when a Flutter widget should feel physically lifted: a focused input card, a pressed product tile, or a selected dashboard panel. It animates Material elevation through `PhysicalModel`, so the effect describes a surface moving above its parent instead of manually guessing blur radius, spread, and offset values for a `BoxShadow`.

## A card that lifts with state

The cleanest setup is to make the application state the `target` of an `Animate` widget. A target of `0` places the effect at its beginning, while `1` moves it to the end. Rebuilding the widget with a new target reverses or advances the same timeline.

Here is a small focusable card. The `Focus` widget owns the interaction state, and `ElevationEffect` owns only the visual interpolation.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class FocusCard extends StatefulWidget {
  const FocusCard({super.key});

  @override
  State<FocusCard> createState() => _FocusCardState();
}

class _FocusCardState extends State<FocusCard> {
  bool focused = false;

  @override
  Widget build(BuildContext context) {
    return Focus(
      onFocusChange: (value) => setState(() => focused = value),
      child: Builder(
        builder: (context) {
          return Animate(
            target: focused ? 1 : 0,
            effects: [
              ElevationEffect(
                begin: 1,
                end: 10,
                duration: 220.ms,
                curve: Curves.easeOut,
                color: Colors.black54,
                borderRadius: BorderRadius.circular(20),
              ),
            ],
            child: Card(
              elevation: 0,
              shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(20),
              ),
              child: const Padding(
                padding: EdgeInsets.all(24),
                child: Text('Keyboard-focusable card'),
              ),
            ),
          );
        },
      ),
    );
  }
}
```

The inner `Card` has `elevation: 0` on purpose. Giving both `Card` and `ElevationEffect` a visible elevation creates two shadow systems, and the result usually looks darker rather than more physical. The effect becomes the single owner of the animated shadow.

The `borderRadius` argument also matters. `ElevationEffect` builds a `PhysicalModel`; its radius should match the visual card radius. If the values differ, the shadow can look clipped at the corners or appear to float outside the surface.

## `target` versus replaying an entrance animation

A common first attempt is to call `animate()` again whenever `focused` changes. That turns a state change into a new entrance animation. The card may fade or slide from its original position when the user only wanted a small lift.

`target` communicates a different rule: keep the effect mounted, then move its controller between two positions.

| Requirement | Useful setup | Why |
| --- | --- | --- |
| Card enters the screen once | `value: 0` with normal effects | The initial animation is independent of focus |
| Card lifts on focus | `target: focused ? 1 : 0` | The same elevation timeline can reverse |
| Card stays raised after selection | Store selection in app state | Focus can disappear without changing selection |
| Shadow needs custom offset or spread | `BoxShadowEffect` | It exposes more shadow-specific control |

This distinction prevented a subtle bug in a settings screen. I initially connected elevation to the text field's rebuild, so validation messages replayed the card's entrance. The card looked as if it was jumping every time the user typed. Keeping `Animate` mounted and changing only `target` made validation and elevation independent.

## Press feedback without changing layout

The same pattern works for a tappable tile. Use a short duration and a modest elevation range; a jump from `0` to `24` feels more like a floating panel than a press response.

```dart
class ProductTile extends StatefulWidget {
  const ProductTile({super.key});

  @override
  State<ProductTile> createState() => _ProductTileState();
}

class _ProductTileState extends State<ProductTile> {
  bool pressed = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => setState(() => pressed = true),
      onTapUp: (_) => setState(() => pressed = false),
      onTapCancel: () => setState(() => pressed = false),
      child: Animate(
        target: pressed ? 0.25 : 0,
        effects: [
          ElevationEffect(
            begin: 2,
            end: 8,
            duration: 120.ms,
            curve: Curves.easeOut,
            borderRadius: BorderRadius.circular(16),
          ),
        ],
        child: const SizedBox(
          width: 220,
          height: 120,
          child: Card(
            elevation: 0,
            child: Center(child: Text('Tap to preview')),
          ),
        ),
      ),
    );
  }
}
```

Here `0.25` is intentional. A target does not have to be only `0` or `1`; it can stop partway through the effect. That gives a pressed state that is visibly raised without reaching the full selected-state shadow.

## When `BoxShadowEffect` is the better choice

`ElevationEffect` is convenient because it models Material depth, but it is not a universal shadow animator. It is implemented with `PhysicalModel`, which means the child is rendered inside a physical shape. That is useful for a rounded card, but it deserves testing when the child contains unusual clipping, platform views, or a transparent surface.

Choose `BoxShadowEffect` when the design requires a precise shadow offset, blur radius, spread radius, or multiple colored shadows. Choose `ElevationEffect` when the design language already uses Material surfaces and the important state is simply “resting” versus “raised.”

For accessibility, keep the state readable without the shadow. A focused card still needs a visible focus indicator, and a selected tile should not rely on elevation alone. Also test dark themes: a black shadow with a large elevation can disappear against a dark background, while a restrained border or surface-color change remains visible.

`ElevationEffect` removes the shadow-tween bookkeeping from a very common interaction. Give it a matching border radius, let `target` represent reversible UI state, and use `BoxShadowEffect` when the shadow itself—not Material depth—is the design requirement.

Official references: [ElevationEffect API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ElevationEffect-class.html) and [Animate API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/Animate-class.html).
