---
layout: post
title: "flutter_animate SizeEffect - Animate Flutter Expansion Without Manual Controllers"
description: "Learn how to use flutter_animate SizeEffect for Flutter expand and collapse animations, with layout trade-offs, clipping fixes, and state-driven examples."
date: 2026-08-12
tags: [flutter_animate, animation, performance, Android, iOS]
comments: true
share: true
---

![flutter_animate SizeEffect expanding a Flutter settings panel](https://colinchflutter.github.io/assets/images/flutter-animate-size-effect-expand-collapse.png)

*The useful detail is the changing space between the header and the action row, not just a child being visually squeezed.*

`flutter_animate`'s `SizeEffect` is the right effect when a Flutter widget should reveal or hide the space it occupies. That makes it different from `ScaleEffect`: scale changes pixels inside the same layout box, while size changes the layout along an axis and lets neighboring widgets move naturally.

## A state-driven expanding panel

The cleanest setup is a stable child with a `target` value. `0` keeps the panel collapsed and `1` restores its full size. There is no separate `AnimationController`, and rebuilding the parent does not require manually forwarding animation values.

The code below keeps the header outside the effect so it remains tappable while the details are closed:

```dart
class SettingsPanel extends StatefulWidget {
  const SettingsPanel({super.key});

  @override
  State<SettingsPanel> createState() => _SettingsPanelState();
}

class _SettingsPanelState extends State<SettingsPanel> {
  bool expanded = false;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [
          ListTile(
            title: const Text('Advanced settings'),
            trailing: Icon(expanded
                ? Icons.keyboard_arrow_up
                : Icons.keyboard_arrow_down),
            onTap: () => setState(() => expanded = !expanded),
          ),
          Column(
            children: [
              const Divider(height: 1),
              const Padding(
                padding: EdgeInsets.all(16),
                child: Text('These options are only needed for power users.'),
              ),
              FilledButton(
                onPressed: () {},
                child: const Text('Open configuration'),
              ),
            ],
          ).animate(target: expanded ? 1 : 0).size(
                begin: 0,
                end: 1,
                alignment: Alignment.topCenter,
                duration: 220.ms,
                curve: Curves.easeOutCubic,
              ),
        ],
      ),
    );
  }
}
```

The `alignment` matters. `Alignment.topCenter` pins the top edge below the header, so the content appears to unfold downward. With the default center alignment, both edges can move and the panel may look as if it is floating away from its title.

## SizeEffect versus ScaleEffect

Choosing the wrong one is the most common source of awkward expansion motion. A transform can look correct in isolation but leave the rest of the screen arranged as if the hidden content still exists.

| Goal | Better choice | What neighboring widgets do |
|---|---|---|
| Press feedback on a button | `ScaleEffect` | Stay in the same positions |
| Reveal a details section | `SizeEffect` | Move as the section grows |
| Hide content but preserve its slot | `VisibilityEffect` | Keep the layout space |
| Replace one child with another | `SwapEffect` or `CrossfadeEffect` | Depends on the replacement layout |

For a settings panel, accordion row, or inline validation message, moving siblings is usually the intended behavior. For a card inside a fixed grid, it is usually safer to scale or fade the contents instead, because a real size change can disturb every item in the grid.

## Prevent the overflow flash

Size animation reveals a partial child during the transition. If that child has a large image, a rounded background, or a button with a shadow, its edges can briefly paint outside the animated area. Wrap the animated content with `ClipRect` when the effect should behave like a viewport:

```dart
ClipRect(
  child: details.animate(target: expanded ? 1 : 0).size(
    begin: 0,
    end: 1,
    alignment: Alignment.topCenter,
    duration: 220.ms,
    curve: Curves.easeOutCubic,
  ),
)
```

This is especially useful inside a `ListView`. Without clipping, a collapsing row can draw over the next row for a frame, which is easy to miss on a fast device but obvious in a screen recording.

## Keep the state and identity stable

The effect should be attached to the same widget across state changes. A changing key or conditional branch that creates a fresh subtree can make the panel restart from its initial value instead of reversing smoothly.

```dart
// Stable: the same animated subtree receives a new target.
details.animate(target: expanded ? 1 : 0).size(
  begin: 0,
  end: 1,
  duration: 220.ms,
)
```

When the content itself changes, use a stable outer animated widget and put the changing child inside it. This separates “the panel opened” from “the panel received new data”, so a network refresh does not replay the expansion.

## Practical limits

`SizeEffect` is convenient, but it still causes layout work while the size changes. A long list of independently expanding rows can therefore cost more than a transform-only animation. Keep the duration short, avoid nesting several size animations, and profile a realistic list on a lower-end Android device.

For most panels, these starting values are enough:

| Interface | Duration | Curve | Extra guard |
|---|---:|---|---|
| Inline error message | 160 ms | `easeOut` | `ClipRect` if text has decoration |
| Accordion details | 220 ms | `easeOutCubic` | Anchor at the top |
| Large filter sheet content | 280 ms | `easeInOut` | Avoid many nested size effects |

`SizeEffect` earns its place when layout should participate in the motion. Use it for content that truly opens a space, keep the animated subtree stable, and add clipping when partial frames should stay inside the row. That combination gives Flutter expansion behavior without hand-written controller lifecycle code.
