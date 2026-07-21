---
layout: post
title: "flutter_animate StreamBuilder - Animate Live Flutter Updates Without Replay Loops"
description: "Use flutter_animate with StreamBuilder to animate only changed live data while avoiding replay loops, stale snapshots, and noisy rebuilds."
date: 2026-07-21
tags: [flutter_animate, animation, networking, performance, state_management]
comments: true
share: true
---

![Flutter StreamBuilder live update animation with flutter_animate](https://colinchflutter.github.io/assets/images/flutter-animate-streambuilder-live-updates.png)

The reliable way to animate a Flutter `StreamBuilder` is to keep the stream responsible for data and give each changing visual value a stable identity. I initially put `.animate().fadeIn().slideY()` around the entire snapshot builder. Every stream event then replayed the entrance animation, including events that changed an unrelated row. The feed looked like it was constantly refreshing instead of receiving live updates.

The fix is small: render the stream state normally, key only the value that changed, and use `flutter_animate` as a short feedback layer. This is useful for chat previews, delivery tracking, market data, and any screen where updates arrive more often than the user changes screens.

## Separate the stream state from the animated item

The stream can emit loading, data, and error states. Those states should not all be treated as animation triggers.

| Responsibility | Flutter / flutter_animate piece | Why it stays separate |
| --- | --- | --- |
| Connection and snapshots | `StreamBuilder` | Owns subscription and async state |
| List identity | `ValueKey(item.id)` | Prevents rows from inheriting the wrong animation |
| Changed value feedback | `ValueKey(item.updatedAt)` | Replays only when the displayed value changes |
| Entrance effect | `fadeIn`, `slideY`, `then` | Makes a settled update easy to notice |

Here is a compact live feed. The stream is injected so the widget remains easy to test with a synchronous or fake stream.

```dart
class LiveFeed extends StatelessWidget {
  const LiveFeed({super.key, required this.events});

  final Stream<List<FeedItem>> events;

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<List<FeedItem>>(
      stream: events,
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return const Center(child: Text('Could not load updates'));
        }
        if (!snapshot.hasData) {
          return const Center(child: CircularProgressIndicator());
        }

        return ListView.builder(
          itemCount: snapshot.data!.length,
          itemBuilder: (context, index) {
            final item = snapshot.data![index];
            return FeedRow(
              key: ValueKey(item.id),
              item: item,
            );
          },
        );
      },
    );
  }
}
```

The important key is on the row, not on the entire `ListView`. If the backend sends the same item again, Flutter can preserve that row's state. If an item is inserted at the top, the existing rows also keep their identity instead of inheriting a neighbor's animation.

## Animate only the changed field

A live event often contains more fields than the user needs to notice. For example, a delivery item can receive a new `status` while its address and timestamp stay the same. Wrapping the full row causes the icon, address, and action button to move on every event. Animate the status label instead.

```dart
class FeedRow extends StatelessWidget {
  const FeedRow({super.key, required this.item});

  final FeedItem item;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(item.title),
      subtitle: Text(item.subtitle),
      trailing: SizedBox(
        width: 96,
        child: Text(
          item.status,
          key: ValueKey('${item.id}:${item.updatedAt}'),
        )
            .animate()
            .fadeIn(duration: 180.ms)
            .slideX(begin: .12, end: 0, duration: 180.ms)
            .then(delay: 500.ms)
            .shimmer(duration: 350.ms),
      ),
    );
  }
}
```

`updatedAt` is the event's meaningful change token. Do not use `DateTime.now()` in the widget build; that creates a new key on every rebuild and turns theme changes, orientation changes, and parent updates into fake live events. If the server does not provide a version or timestamp, derive a deterministic token from the fields that should trigger feedback.

## Avoid stale stream subscriptions

One subtle failure happens when the stream is created inside `build`:

```dart
// Avoid: a new stream can be subscribed to on every rebuild.
stream: repository.watchFeed(),
```

If `watchFeed()` returns a new broadcast transformation each time, a parent rebuild can cancel and recreate the subscription. The result is duplicated listeners, missing events, or a loading indicator that flashes again. Create the stream once in the state object, or pass a stable stream from above.

```dart
class FeedPage extends StatefulWidget {
  const FeedPage({super.key, required this.repository});

  final FeedRepository repository;

  @override
  State<FeedPage> createState() => _FeedPageState();
}

class _FeedPageState extends State<FeedPage> {
  late final Stream<List<FeedItem>> _events;

  @override
  void initState() {
    super.initState();
    _events = widget.repository.watchFeed();
  }

  @override
  Widget build(BuildContext context) => LiveFeed(events: _events);
}
```

Keep the animation short when events can arrive quickly. A 180 ms value transition is readable without creating a backlog of outgoing widgets. If the same status can arrive repeatedly, compare the incoming version in the repository and suppress identical events before they reach the UI.

The pattern is straightforward: `StreamBuilder` owns asynchronous state, stable keys preserve row identity, and `flutter_animate` marks the exact value that changed. That combination keeps live Flutter screens responsive while making real updates visible instead of replaying the whole feed on every snapshot.
