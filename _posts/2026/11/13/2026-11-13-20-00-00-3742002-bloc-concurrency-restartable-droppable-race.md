---
layout: post
title: "bloc_concurrency restartable vs droppable - Prevent Flutter Search and Submit Races"
description: "Choose bloc_concurrency transformers by side effect: use restartable for replaceable searches, droppable for duplicate submits, and add request cancellation when an HTTP call must actually stop."
date: 2026-11-13
tags: [bloc_concurrency, flutter_bloc, state_management, networking, testing, Flutter]
comments: true
share: true
---
![Flutter bloc_concurrency restartable and droppable event race](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

*The useful boundary is whether an older event is replaceable or whether its side effect must finish.*

If a Flutter search box shows an older response after a newer query, `restartable()` is usually the right starting point. If a payment, save, or upload button can be tapped twice, `droppable()` is safer. Neither transformer is a universal cancellation switch: `restartable` stops the old handler's event stream, but it cannot undo an HTTP request or database write that already started. This applies to `bloc` event handlers using `bloc_concurrency`; it does not replace server-side idempotency.

## The race hidden by a simple transformer

`bloc` processes events concurrently by default. That is fine for independent reads, but a slow request for `flutter` can finish after a fast request for `flutter package`. The UI then receives the wrong result for the current text. `bloc_concurrency` 0.3.0 provides `concurrent`, `sequential`, `droppable`, and `restartable`; the official `restartable` API is implemented with `switchMap`, so the latest handler remains subscribed.

| User action | Correct policy | Why |
| --- | --- | --- |
| Search or filter text | `restartable()` | Older results have no value once the query changes |
| Refresh button | `droppable()` or an explicit refresh queue | Ignore repeated taps while one refresh is active |
| Save or payment | `sequential()` + idempotency key | Every accepted write needs an ordered, retryable result |
| Independent analytics events | `concurrent()` | Completion order does not change the user-visible state |

The important distinction is not “fast versus slow.” It is “can the old operation be discarded without changing data?”

## A safe search handler

The following handler makes old search results disposable. The repository call is a read, and the API can tolerate an abandoned request. The `emit` after the `await` belongs to the active handler; an old handler no longer owns the event stream after a newer event arrives.

```dart
import 'package:bloc/bloc.dart';
import 'package:bloc_concurrency/bloc_concurrency.dart';

sealed class SearchEvent {}

final class QueryChanged extends SearchEvent {
  QueryChanged(this.value);
  final String value;
}

sealed class SearchState {}
final class SearchIdle extends SearchState {}
final class SearchLoading extends SearchState {}
final class SearchData extends SearchState {
  SearchData(this.items);
  final List<String> items;
}
final class SearchFailure extends SearchState {
  SearchFailure(this.error);
  final Object error;
}

class SearchBloc extends Bloc<SearchEvent, SearchState> {
  SearchBloc(this.repository) : super(SearchIdle()) {
    on<QueryChanged>(
      _queryChanged,
      transformer: restartable(),
    );
  }

  final SearchRepository repository;

  Future<void> _queryChanged(
    QueryChanged event,
    Emitter<SearchState> emit,
  ) async {
    final query = event.value.trim();
    if (query.isEmpty) {
      emit(SearchIdle());
      return;
    }

    emit(SearchLoading());
    try {
      final items = await repository.search(query);
      emit(SearchData(items));
    } catch (error) {
      emit(SearchFailure(error));
    }
  }
}
```

This controls state delivery, not transport work. If the search endpoint is expensive, pass a cancellation handle to the repository and cancel it when the handler is superseded. If cancellation is unavailable, keep the request read-only and add a request-generation check in the repository or UI layer. Do not use `restartable()` for `POST /orders`, `incrementBalance`, or a file upload merely because the screen wants the latest state.

## Why `droppable()` fits submit buttons

For a submit event, a second tap normally represents the same intent, not a newer value. `droppable()` ignores new events while the current handler is running. That prevents a second local transition, but it still does not make the server write idempotent. A crash after the server accepts the request and before the client receives the response can still lead to a retry.

```dart
import 'package:bloc_concurrency/bloc_concurrency.dart';

on<SavePressed>(
  _savePressed,
  transformer: droppable(),
);

Future<void> _savePressed(
  SavePressed event,
  Emitter<SaveState> emit,
) async {
  emit(SaveInProgress());
  try {
    await repository.save(
      event.form,
      idempotencyKey: event.idempotencyKey,
    );
    emit(SaveSucceeded());
  } catch (error) {
    emit(SaveFailed(error));
  }
}
```

The server should treat the `idempotencyKey` as the operation identity and return the original result for a repeated key. If the user can legitimately edit the form again while saving, `droppable()` may hide a real change. In that case use `sequential()` for ordered writes, or disable the submit action and show the pending version explicitly.

## A reproducible test matrix

The bug is easy to miss when every fake request has the same delay. Make the first operation slow and the second fast. Then assert both the visible state and the side effects.

```dart
test('restartable keeps the latest search result', () async {
  final repository = FakeSearchRepository(
    delays: {'old': const Duration(milliseconds: 80),
             'new': const Duration(milliseconds: 5)},
  );
  final bloc = SearchBloc(repository);

  final states = <SearchState>[];
  final subscription = bloc.stream.listen(states.add);
  bloc.add(QueryChanged('old'));
  await Future<void>.delayed(const Duration(milliseconds: 1));
  bloc.add(QueryChanged('new'));

  await Future<void>.delayed(const Duration(milliseconds: 100));
  expect((states.last as SearchData).items, ['new-result']);
  expect(repository.calls, ['old', 'new']);

  await subscription.cancel();
  await bloc.close();
});
```

The `calls` assertion is deliberate: `restartable()` can leave two network calls in flight even though only one result is emitted. For a write test, send two `SavePressed` events while the first future is pending and assert that the repository saw one call. Add a separate retry test with the same idempotency key; a green transformer test alone does not prove duplicate protection.

## Release checklist

- Classify the event as replaceable read, duplicate-prone action, ordered write, or independent work.
- Test with deliberately different completion times, not only with `Future.value`.
- For `restartable()`, decide whether abandoned transport requests need cancellation.
- For `droppable()` and `sequential()`, give writes an idempotency key at the API boundary.
- Log event type, request ID, and server idempotency key so a “duplicate” can be distinguished from a stale state emission.

`bloc_concurrency` solves the ordering policy at the event-stream boundary. The durable fix is a matching contract at the side-effect boundary: cancellable reads, idempotent writes, and tests that force the race. That is why copying `restartable()` onto every handler often makes the UI look correct while leaving production data bugs untouched.

Sources: [bloc_concurrency package](https://pub.dev/packages/bloc_concurrency), [restartable API documentation](https://pub.dev/documentation/bloc_concurrency/latest/bloc_concurrency/restartable.html), [Bloc migration guide](https://bloclibrary.dev/migration/), and [the documented restartable cancellation issue](https://github.com/felangel/bloc/issues/3349).
