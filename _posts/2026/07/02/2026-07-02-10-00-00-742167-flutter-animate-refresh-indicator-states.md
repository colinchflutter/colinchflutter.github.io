---
layout: post
title: "flutter_animate RefreshIndicator States - pull, loading, and updated feedback"
description: "Use flutter_animate with RefreshIndicator to show clear pull-to-refresh states, loading motion, and updated feedback without custom scroll physics."
date: 2026-07-02
tags: [flutter_animate, animation, ListView, state_management]
comments: true
share: true
---
![Flutter pull to refresh list state animation](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

The cleanest pull-to-refresh animation is usually not a custom `RefreshIndicator`. Keep Flutter's built-in refresh behavior, then animate a small state panel above the list with `flutter_animate`. That gives users useful feedback without taking ownership of scroll physics, overscroll behavior, or platform gestures.

The mistake I made once was trying to animate the indicator itself. It worked on Android, felt slightly wrong on iOS, and broke when the list had a pinned header. The less fragile version is a normal `RefreshIndicator` plus a lightweight status row that changes between idle, loading, and updated.

## The state model

Use one enum. Do not let three booleans fight each other.

```dart
enum RefreshUiState { idle, refreshing, updated, failed }
```

The page owns the state around the async refresh call:

```dart
class FeedPageState extends State<FeedPage> {
  RefreshUiState _refreshState = RefreshUiState.idle;

  Future<void> _refresh() async {
    setState(() => _refreshState = RefreshUiState.refreshing);

    try {
      await repository.reloadFeed();
      if (!mounted) return;
      setState(() => _refreshState = RefreshUiState.updated);
      await Future<void>.delayed(900.ms);
      if (mounted) setState(() => _refreshState = RefreshUiState.idle);
    } catch (_) {
      if (!mounted) return;
      setState(() => _refreshState = RefreshUiState.failed);
    }
  }
}
```

That delay is not for fake loading. It keeps the "updated" confirmation visible long enough to read. Without it, the state flips so fast that the animation looks like a glitch.

## Animated status row

Place the row as the first item in the list. `ValueKey` matters because it tells `flutter_animate` to replay the entry animation when the refresh state changes.

```dart
RefreshIndicator(
  onRefresh: _refresh,
  child: ListView(
    physics: const AlwaysScrollableScrollPhysics(),
    children: [
      RefreshStatusRow(state: _refreshState),
      for (final item in items) FeedTile(item: item),
    ],
  ),
)
```

The row can stay compact:

```dart
class RefreshStatusRow extends StatelessWidget {
  const RefreshStatusRow({super.key, required this.state});

  final RefreshUiState state;

  @override
  Widget build(BuildContext context) {
    final (icon, label) = switch (state) {
      RefreshUiState.idle => (Icons.keyboard_arrow_down, 'Pull to refresh'),
      RefreshUiState.refreshing => (Icons.sync, 'Refreshing feed'),
      RefreshUiState.updated => (Icons.check_circle, 'Feed updated'),
      RefreshUiState.failed => (Icons.error_outline, 'Refresh failed'),
    };

    return Padding(
      padding: const EdgeInsets.fromLTRB(16, 10, 16, 6),
      child: Row(
        key: ValueKey(state),
        children: [
          Icon(icon, size: 18),
          const SizedBox(width: 8),
          Text(label),
        ],
      )
          .animate(key: ValueKey(state))
          .fadeIn(duration: 160.ms)
          .slideY(begin: -0.25, end: 0, curve: Curves.easeOutCubic),
    );
  }
}
```

For the refreshing state, I usually add a tiny rotation to the icon instead of animating the whole row forever. Constant full-row motion makes the list feel unstable.

```dart
Icon(Icons.sync, size: 18)
    .animate(onPlay: (controller) => controller.repeat())
    .rotate(duration: 700.ms, curve: Curves.linear)
```

Keep that animation only inside the `refreshing` branch. If it remains mounted after refresh, the repeat controller keeps spinning in a state where the UI says the work is done.

## Practical limits

This pattern does not show real drag progress. If the design requires a progress ring that follows overscroll distance, listen to scroll notifications or use a custom sliver. For most feed screens, that is extra machinery for little gain.

Also, keep the status row outside item pagination state. Pull-to-refresh replaces the current dataset; infinite scroll appends to it. Mixing both into one `loading` flag is how stale "Refreshing feed" labels get stuck above newly loaded items.

## Short checklist

- Use `RefreshIndicator` for gesture and platform behavior.
- Use one enum for refresh UI state.
- Key the animated row by state so transitions replay.
- Animate confirmation briefly, then return to idle.
- Reserve looping motion for the active refreshing state only.

