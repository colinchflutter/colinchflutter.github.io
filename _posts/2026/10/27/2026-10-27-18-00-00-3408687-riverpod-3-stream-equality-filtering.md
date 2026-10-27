---
layout: post
title: "Riverpod 3 StreamProvider Equality Filtering - Prevent Flutter Updates from Disappearing"
description: "Riverpod 3 compares provider values with ==, so stream updates can stop reaching Flutter widgets when equal-looking objects are emitted. Find the migration trap and choose a safe fix with updateShouldNotify."
date: 2026-10-27
tags: [riverpod, state_management, migration, testing, performance, Flutter]
comments: true
share: true
---

![Riverpod 3 stream update filtering and Flutter rebuild flow](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

Riverpod 3 is a good upgrade for apps that want the unified `Notifier` API, but a `StreamProvider` or `StreamNotifier` may stop notifying widgets after the migration when successive values compare equal with `==`. Keep the new default for immutable state where equal values really mean “no visible change”. Override `updateShouldNotify` when the stream represents events, mutable snapshots, or values whose identity matters. A simple `ref.watch` change will not repair this behavior.

## The production symptom

Imagine a stream that emits a new `Telemetry` object every second. The object contains the same temperature but a new `receivedAt` timestamp. If `Telemetry` implements value equality using only temperature, Riverpod 3 can filter the second object before the widget sees it. The stream is still active; the missing update is a notification policy, not a disconnected socket.

| Situation | Riverpod 3 default | Better decision |
|---|---|---|
| Immutable API state with unchanged fields | Filter with `==` | Keep the default |
| Event stream where every event matters | Equal events may be filtered | Override `updateShouldNotify` |
| Mutable object reused and changed in place | Equality can hide a mutation | Emit a new immutable value |
| Stream used for analytics or logging | UI rebuild is not the goal | Consume it outside the widget tree |

The official Riverpod 3 migration notes call out the change: providers now use `==` consistently, and `StreamProvider`/`StreamNotifier` are the most likely places to notice it. This is separate from Riverpod 3 automatic retry. A retry can duplicate a failed request; equality filtering can hide a successful emission.

## Reproduce the filtering boundary

This small model makes the equality rule explicit. It is a reproducible example, not a claim about a particular backend.

```dart
class StockTick {
  const StockTick({required this.price, required this.sequence});

  final double price;
  final int sequence;

  @override
  bool operator ==(Object other) =>
      other is StockTick && other.price == price;

  @override
  int get hashCode => price.hashCode;
}
```

`StockTick(100, 1)` and `StockTick(100, 2)` are different events but equal values in this model. If the UI must display the sequence or animate every tick, the equality definition is too narrow. Changing the provider first only hides the design problem.

## Choose the fix at the provider boundary

For state, include every field that changes the meaning of the value in equality. For event-like streams, make the provider decide that each `AsyncValue` transition should be delivered.

```dart
class StockFeed extends StreamNotifier<StockTick> {
  @override
  Stream<StockTick> build() async* {
    yield* marketStream();
  }

  @override
  bool updateShouldNotify(
    AsyncValue<StockTick> previous,
    AsyncValue<StockTick> next,
  ) {
    // This feed drives a ticker tape: equal prices are still events.
    return true;
  }
}

final stockFeedProvider =
    StreamNotifierProvider<StockFeed, StockTick>(StockFeed.new);
```

Use `return previous != next` when the default value comparison is the intended contract but the override documents that choice. Do not return `true` globally just to make a stale screen redraw: it can increase rebuilds and can mask a broken `==` implementation.

## Migration checklist

Run this review before and after moving to Riverpod 3:

```text
[ ] Find StreamProvider and StreamNotifier declarations.
[ ] Identify models with custom ==, Equatable, records, or mutable fields.
[ ] Decide whether each stream is state, snapshot, or event data.
[ ] Add updateShouldNotify only where the contract needs it.
[ ] Test two consecutive emissions with equal values and different identity.
[ ] Test a genuine changed value and an error/recovery transition.
[ ] Count widget rebuilds in profile mode before optimizing further.
```

One practical test should assert the business result instead of only checking that a stream remains connected: emit two equal-by-`==` ticks, then verify whether the consumer is supposed to receive one or two updates. If the answer is two, the provider contract must say so.

Riverpod 3’s equality filtering is useful for immutable state and noisy streams, but it changes an assumption that older code could leave implicit. Treat `==` as part of the provider API: define it for state, override it for events, and keep durable side effects outside a widget-scoped listener.

Sources checked 2026-10-27: [Riverpod 3 migration guide](https://riverpod.dev/docs/3.0_migration), [Riverpod 3 changes](https://riverpod.dev/docs/whats_new), and the [riverpod changelog](https://pub.dev/packages/riverpod/changelog).
