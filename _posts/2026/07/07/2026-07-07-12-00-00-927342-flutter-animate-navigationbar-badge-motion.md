---
layout: post
title: "flutter_animate NavigationBar Badge Motion - unread counts without tab jitter"
description: "Use flutter_animate with Flutter NavigationBar badges to show unread count changes without replaying tabs or shifting layout."
date: 2026-07-07
tags: [flutter_animate, navigation, animation, Flutter, performance]
comments: true
share: true
---
![flutter_animate NavigationBar badge motion](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)
The useful detail in this image is the fixed bottom app navigation area: badge animation should clarify a count change without making the whole destination feel unstable.

The best place to use `flutter_animate` with a Flutter `NavigationBar` is the badge or icon layer, not the entire destination. When unread counts change, a tiny fade, scale, or color pulse is enough. If the whole tab item moves, users can read it as a navigation state change instead of a notification update.

The trap I hit was wrapping each `NavigationDestination` label with `.animate().fadeIn().slideY()`. It looked polished in a recording. In a real app, a chat count changing from 2 to 3 caused the selected tab to replay, the icon nudged upward, and the bar felt like it was reselecting itself. Nothing was broken in Flutter. The animation was attached to the wrong part of the widget tree.

## Keep NavigationBar boring

Let `NavigationBar` own selection, indicator motion, keyboard focus, and semantics. The animation should sit inside the icon slot, where the visual event actually happens.

```dart
NavigationBar(
  selectedIndex: index,
  onDestinationSelected: onSelect,
  destinations: [
    NavigationDestination(
      icon: AnimatedNavIcon(
        icon: Icons.inbox_outlined,
        selectedIcon: Icons.inbox,
        count: unreadInbox,
        selected: index == 0,
      ),
      label: 'Inbox',
    ),
    NavigationDestination(
      icon: AnimatedNavIcon(
        icon: Icons.person_outline,
        selectedIcon: Icons.person,
        count: pendingProfiles,
        selected: index == 1,
      ),
      label: 'Profile',
    ),
  ],
);
```

This keeps the tab layout stable. The destination still has one label, one icon slot, and one selected index. Only the badge changes when data changes.

## Animate the badge, not the tab

The badge needs a stable footprint. I prefer reserving the badge area even when the count is zero, then animating opacity and scale inside it. That avoids a subtle width jump when the badge appears.

```dart
class AnimatedNavIcon extends StatelessWidget {
  const AnimatedNavIcon({
    super.key,
    required this.icon,
    required this.selectedIcon,
    required this.count,
    required this.selected,
  });

  final IconData icon;
  final IconData selectedIcon;
  final int count;
  final bool selected;

  @override
  Widget build(BuildContext context) {
    final visible = count > 0;
    final label = count > 99 ? '99+' : '$count';

    return SizedBox(
      width: 32,
      height: 32,
      child: Stack(
        clipBehavior: Clip.none,
        children: [
          Center(
            child: Icon(selected ? selectedIcon : icon),
          ),
          Positioned(
            right: -4,
            top: -3,
            child: _UnreadBadge(
              key: ValueKey(label),
              label: label,
              visible: visible,
            ),
          ),
        ],
      ),
    );
  }
}
```

The `ValueKey(label)` is intentional. It tells Flutter that `2`, `3`, and `99+` are different visual states, so the badge can replay a short count-change animation. I do not key the whole icon, because that would also rebuild the selected icon layer.

## Badge widget

Here is the badge itself. The important part is that the outer box keeps the same size whether the count is visible or hidden.

```dart
class _UnreadBadge extends StatelessWidget {
  const _UnreadBadge({
    super.key,
    required this.label,
    required this.visible,
  });

  final String label;
  final bool visible;

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: 22,
      height: 18,
      child: Center(
        child: DecoratedBox(
          decoration: BoxDecoration(
            color: Theme.of(context).colorScheme.error,
            borderRadius: BorderRadius.circular(9),
          ),
          child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 5),
            child: Text(
              label,
              style: Theme.of(context).textTheme.labelSmall?.copyWith(
                    color: Theme.of(context).colorScheme.onError,
                    fontSize: 10,
                    height: 1.1,
                  ),
            ),
          ),
        )
            .animate(target: visible ? 1 : 0)
            .fade(duration: 90.ms)
            .scale(
              begin: const Offset(.82, .82),
              end: const Offset(1, 1),
              duration: 140.ms,
              curve: Curves.easeOutCubic,
            ),
      ),
    );
  }
}
```

I tried a stronger bounce here first: scale from `0.6` to `1.15`, then settle. It made sense on paper, but it looked too much like an error alert when three counts arrived in quick succession. A badge in persistent navigation should be noticed once, not compete with the page content.

## What should move

| Change | Good motion | Bad motion |
|---|---|---|
| Count `0` to `1` | badge fade and small scale | shifting the icon slot |
| Count `4` to `5` | badge text replay | whole destination fade |
| Selected tab changes | let `NavigationBar` handle it | custom slide on every tab |
| Burst updates | short pulse, maybe throttled | repeated bounce for each event |

The rule is simple: navigation state and notification state are different signals. If both animate at the same time, the bar becomes harder to read.

## Handling fast count updates

Real notification counts often arrive in bursts. A sync job may update inbox from 0 to 1, then 3, then 6 inside a second. Replaying the badge animation for every intermediate value can feel noisy.

For chat or inbox tabs, I usually debounce the display value for 150 to 250 ms at the state layer. The stored unread count remains accurate, but the UI shows the final count for the burst. That gives one clean badge motion instead of three tiny flashes.

If the count is tied to a stream, keep that behavior outside the badge widget:

```dart
final visibleUnreadCountProvider = Provider<int>((ref) {
  final raw = ref.watch(unreadCountProvider);
  return raw.clamp(0, 99);
});
```

That example only clamps, but the same idea applies to debouncing in Riverpod, Bloc, or a `ValueNotifier`. The badge should render a number; it should not become the place where app event timing is solved.

## Checklist

- Keep `NavigationBar` and `NavigationDestination` stable.
- Animate inside the icon slot, preferably on the badge only.
- Reserve badge width and height so the icon does not jump.
- Key the badge value, not the whole destination.
- Cap large counts with `99+`.
- Debounce bursty count updates before they reach the widget.

`flutter_animate` works well for bottom navigation badges when the motion is local and short. The bar should still feel like navigation. The badge can move just enough to say the data changed, then get out of the user's way.
