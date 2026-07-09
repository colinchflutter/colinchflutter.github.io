---
layout: post
title: "flutter_animate Autocomplete Option Motion - suggestions without overlay jitter"
description: "Use flutter_animate with Flutter Autocomplete to highlight suggestion changes without moving the overlay or stealing keyboard focus."
date: 2026-07-10
tags: [flutter_animate, animation, Flutter, state_management]
comments: true
share: true
---

![flutter_animate Autocomplete suggestion motion](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

The useful `flutter_animate` pattern for Flutter `Autocomplete` is to animate the option row, not the overlay. Keep the text field, focus node, and options view stable, then add a short fade or color pulse to the suggestion that changed. The first version I tried animated the whole options panel. It looked fine with 4 results, but with 12 results the overlay recalculated height while the user typed, and the menu felt like it was slipping away from the cursor.

`Autocomplete` already has enough moving parts: query text, filtered options, highlighted index, keyboard navigation, and an overlay that follows the input field. Motion should explain a state change, not become another state owner.

| Change | Animate this | Avoid |
| --- | --- | --- |
| New query results | Option row entrance | Full overlay slide |
| Keyboard highlight | Background tint | Row size change |
| Selected option | Small trailing check | Text field movement |

Here is the compact version I use for search-heavy forms.

```dart
class CityAutocomplete extends StatelessWidget {
  const CityAutocomplete({super.key, required this.cities});

  final List<String> cities;

  @override
  Widget build(BuildContext context) {
    return Autocomplete<String>(
      optionsBuilder: (value) {
        final query = value.text.trim().toLowerCase();
        if (query.isEmpty) return const Iterable<String>.empty();

        return cities.where((city) {
          return city.toLowerCase().contains(query);
        }).take(8);
      },
      optionsViewBuilder: (context, onSelected, options) {
        return Align(
          alignment: Alignment.topLeft,
          child: Material(
            elevation: 4,
            child: ConstrainedBox(
              constraints: const BoxConstraints(maxHeight: 280, maxWidth: 360),
              child: ListView.builder(
                padding: EdgeInsets.zero,
                itemCount: options.length,
                itemBuilder: (context, index) {
                  final option = options.elementAt(index);

                  return _AnimatedOptionRow(
                    key: ValueKey(option),
                    label: option,
                    onTap: () => onSelected(option),
                  );
                },
              ),
            ),
          ),
        );
      },
    );
  }
}
```

The important part is the `ValueKey(option)`. Without it, Flutter may reuse a row slot when the filtered list changes from `san` to `se`. The visual result is subtle but annoying: the wrong row fades in, or an old highlight appears attached to a new city name.

```dart
class _AnimatedOptionRow extends StatelessWidget {
  const _AnimatedOptionRow({
    super.key,
    required this.label,
    required this.onTap,
  });

  final String label;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: onTap,
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        child: Row(
          children: [
            Expanded(child: Text(label)),
            const Icon(Icons.north_west, size: 16),
          ],
        ),
      ),
    )
        .animate()
        .fadeIn(duration: 90.ms)
        .moveY(begin: 4, end: 0, duration: 120.ms, curve: Curves.easeOut);
  }
}
```

The animation is intentionally small. A 90 ms fade is enough to show that the list has refreshed. A 300 ms cascade feels polished in a demo and slow in a real search field, especially when every keystroke changes the options.

The trap is animating layout properties inside the overlay. Do not animate row height, panel width, or the `Align` that positions the options view. That can fight the overlay's own positioning logic and create a tiny vertical jump after every rebuild.

Use this checklist when the menu feels unstable:

- Keep `Autocomplete` responsible for text input, focus, and selection.
- Put animation inside each option row, not around `optionsViewBuilder`.
- Use stable option keys based on IDs or labels.
- Cap the list height so new results do not resize the overlay too much.
- Keep durations under 150 ms for typing-driven updates.

`flutter_animate` fits `Autocomplete` when it stays local. Let Flutter own the overlay mechanics, then use motion as a quick hint that the suggestion set changed.
