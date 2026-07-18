---
layout: post
title: "flutter_animate FutureBuilder - Animate Loading, Error, and Success States Without Replays"
description: "Use flutter_animate with Flutter FutureBuilder to animate loading, error, and success states while avoiding new futures and accidental animation replays."
date: 2026-07-18
tags: [flutter_animate, animation, state_management, networking, Flutter]
comments: true
share: true
---

![Flutter app showing loading, error, and success states in an async card](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

*The useful visual distinction is between a state that changed and a widget that merely rebuilt.*

`FutureBuilder` and `flutter_animate` work well together when the animation belongs to the async state, not to the `FutureBuilder` itself. The reliable pattern is to create the future once, map the snapshot to a small status value, and use that status as the animation key. A parent rebuild can then update surrounding UI without replaying the loading card.

## The trap: creating a future inside `build`

This looks harmless, but it can start a new request whenever the widget rebuilds:

```dart
FutureBuilder<List<Order>>(
  future: repository.fetchOrders(),
  builder: (context, snapshot) => ...,
)
```

That causes two problems. The request may run more often than expected, and the `waiting` state can appear again, so a loading animation restarts while an unrelated counter changes. Keep the future in `initState` instead:

```dart
class OrdersPanel extends StatefulWidget {
  const OrdersPanel({super.key});

  @override
  State<OrdersPanel> createState() => _OrdersPanelState();
}

class _OrdersPanelState extends State<OrdersPanel> {
  late final Future<List<Order>> _orders;

  @override
  void initState() {
    super.initState();
    _orders = repository.fetchOrders();
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<List<Order>>(
      future: _orders,
      builder: (context, snapshot) => _buildState(snapshot),
    );
  }
}
```

The request now has stable identity. A retry should intentionally replace `_orders`, rather than relying on a rebuild to do it.

## Animate the state boundary, not the whole builder

Give each visual state a stable identity. `flutter_animate` restarts when its key changes, so the success card enters once when the request becomes successful and stays still during later rebuilds.

```dart
Widget _buildState(AsyncSnapshot<List<Order>> snapshot) {
  final state = snapshot.hasError
      ? 'error'
      : snapshot.hasData
          ? 'success'
          : 'loading';

  final child = switch (state) {
    'error' => ErrorCard(
        message: 'Could not load orders',
        onRetry: _retry,
      ),
    'success' => OrdersList(orders: snapshot.data!),
    _ => const LoadingCard(),
  };

  return KeyedSubtree(
    key: ValueKey(state),
    child: child.animate(
      key: ValueKey('orders-$state'),
    ).fadeIn(
      duration: 280.ms,
    ).slideY(
      begin: 0.04,
      end: 0,
      duration: 280.ms,
      curve: Curves.easeOut,
    ),
  );
}
```

The outer `KeyedSubtree` gives Flutter a clear replacement boundary. The `animate` key makes the package's restart intent explicit. In practice, one stable key is often enough; I keep both when the state widget is extracted into a reusable builder because it makes the ownership obvious.

## Retry without animating stale data

A retry should create a new future and return to `loading` deliberately:

```dart
void _retry() {
  setState(() {
    _orders = repository.fetchOrders();
  });
}
```

If the old list should remain visible while refreshing, use a separate `isRefreshing` flag instead of forcing the whole `FutureBuilder` back to `waiting`. Animate a small progress indicator with that flag. Otherwise, users see the content disappear even though the app still has usable data.

| Async situation | Animation target | Avoid |
| --- | --- | --- |
| Initial load | Loading placeholder | Recreating the future in `build` |
| Request failed | Error message and retry action | Shaking the entire page |
| Request succeeded | New content boundary | Replaying on every parent rebuild |
| Refresh with old data | Small progress cue | Hiding usable content |

## Edge cases I check

An empty list is still a successful response, so give it a distinct state such as `empty`; otherwise an empty-state card can reuse the success subtree without entering. Also guard retries after disposal if the repository callback updates local state asynchronously. `FutureBuilder` handles snapshot delivery, but it does not decide whether a visual state is meaningful enough to animate.

The practical rule is simple: stabilize the future, name the async states, and key only the boundary that actually changed. `flutter_animate` then adds motion to the loading, error, and success transitions without turning ordinary Flutter rebuilds into fake network activity.

## Quick checklist

- Create long-lived futures outside `build`.
- Use `error`, `loading`, `success`, and `empty` as explicit visual states.
- Key state replacement with `ValueKey`.
- Keep refresh feedback local when old data is still valid.
- Test a parent `setState` while the request is pending and after it completes.

- [Flutter FutureBuilder API](https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html)
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
