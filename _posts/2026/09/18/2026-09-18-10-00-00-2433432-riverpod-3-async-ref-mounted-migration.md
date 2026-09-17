---
layout: post
title: "Riverpod 2 to 3 Migration - Prevent Disposed Ref Failures After Async Gaps"
description: "A production-focused Riverpod 2 to 3 migration guide for async providers: replace AutoDispose types, cancel work on disposal, and guard ref access after awaits."
date: 2026-09-18
tags: [riverpod, state_management, migration, testing, performance, Android, iOS, Web]
comments: true
share: true
---

![Flutter Riverpod migration flow from an async request to a mounted provider](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

If a Flutter app is moving from Riverpod 2 to 3 and has async work inside `Notifier` or `FutureProvider`, the safe choice is to migrate the type names and make every post-`await` provider access conditional. A simple search-and-replace is enough for `AutoDisposeNotifier`, but it is not enough for a request that finishes after its screen has disappeared. A new app without async provider work does not need this pattern yet.

## The failure that only appears after navigation

Riverpod 3.0 no longer permits using a disposed `Ref` or `Notifier`. The official migration notes recommend `ref.mounted` after an async gap. A typical failure looks harmless in Riverpod 2:

```dart
class Profile extends AutoDisposeAsyncNotifier<User> {
  @override
  Future<User> build() => api.fetchUser();

  Future<void> saveName(String name) async {
    state = const AsyncLoading();
    final user = await api.updateName(name);
    state = AsyncData(user); // Can run after this provider was disposed.
  }
}
```

The user can submit, leave the route, and return to a different screen before `updateName` completes. In Riverpod 3, the final state write may throw rather than silently touching an obsolete provider.

## Migration decisions that should not be mixed together

| Code in Riverpod 2 | Riverpod 3 change | Production check |
| --- | --- | --- |
| `AutoDisposeNotifier<T>` | `Notifier<T>` | Keep `.autoDispose` on the provider if short-lived state is intended |
| `FamilyNotifier<T, Arg>` | `Notifier<T>` with a constructor field | Move the family argument into `this.arg` and make `build()` parameterless |
| `ExampleRef` or `FutureProviderRef<T>` | `Ref` | Do not change `WidgetRef`; it is a different API |
| `await` then `ref.read`/`state =` | Check `ref.mounted`, or cancel the work | Decide whether cancellation or a discarded result is correct |

The type migration is mechanical. The lifetime migration is a behavior change and deserves a test.

## A safe async provider pattern

For a request that can be cancelled, cancellation is preferable because it saves network and parsing work. The following example uses Dio’s `CancelToken`; the same boundary can be implemented with another HTTP client.

```dart
final profileProvider = AsyncNotifierProvider.autoDispose<Profile, User>(
  Profile.new,
);

class Profile extends AsyncNotifier<User> {
  late final CancelToken _cancelToken;

  @override
  Future<User> build() async {
    _cancelToken = CancelToken();
    ref.onDispose(_cancelToken.cancel);
    return api.fetchUser(cancelToken: _cancelToken);
  }

  Future<void> saveName(String name) async {
    state = const AsyncLoading();

    try {
      final user = await api.updateName(
        name,
        cancelToken: _cancelToken,
      );

      // Covers invalidation or disposal that happened after the response.
      if (!ref.mounted) return;
      state = AsyncData(user);
    } on DioException catch (error, stackTrace) {
      if (CancelToken.isCancel(error) || !ref.mounted) return;
      state = AsyncError(error, stackTrace);
    }
  }
}
```

There are two separate protections here. `ref.onDispose` stops work while it is in flight. `ref.mounted` protects the small race between the request completing and the callback writing state. Do not use `ref` inside the `onDispose` callback to read provider state; disposal is already in progress.

If the underlying operation cannot be cancelled, keep the mounted guard and ignore the result. That is different from reporting a failed request:

```dart
final result = await repository.recalculateCart();
if (!ref.mounted) return; // The result is now irrelevant to this provider.
state = AsyncData(result);
```

For a workflow that must finish even after the user leaves the screen—such as a payment confirmation—move the workflow into an application-scoped service or use case. A screen-scoped auto-dispose notifier should not own work whose lifetime is longer than the screen.

## Migration checklist

- Change `AutoDisposeNotifier`, `AutoDisposeAsyncNotifier`, and family notifier base classes to their unified Riverpod 3 counterparts.
- Keep `.autoDispose` on the provider declaration when disposal is part of the design.
- Replace generated `ExampleRef` parameters with `Ref`; leave `WidgetRef` untouched.
- Search every provider method for `await`, then inspect the next `ref` or `state` access.
- Add `ref.onDispose` for cancellable work and `if (!ref.mounted) return` after the await.
- Review stream providers that rely on identity changes: Riverpod 3 filters notifications with `==`.
- Add a test that invalidates or disposes the provider while the future is pending.

The last check is the one most smoke tests miss. Test the lifetime boundary, not only the successful response:

```dart
test('does not write after disposal', () async {
  final pending = Completer<User>();
  final container = ProviderContainer(
    overrides: [apiProvider.overrideWithValue(FakeApi(pending.future))],
  );
  addTearDown(container.dispose);

  final subscription = container.listen(profileProvider, (_, __) {});
  await container.read(profileProvider.future);
  final notifier = container.read(profileProvider.notifier);

  final saving = notifier.saveName('New name');
  container.invalidate(profileProvider);
  pending.complete(User(name: 'New name'));

  await expectLater(saving, completes);
  subscription.close();
});
```

Adapt the fake API to your project’s provider names. The important assertion is that completion after invalidation does not produce an uncaught disposed-Ref exception.

Riverpod’s 3.0 stable release was published on September 10, 2025, and the package changelog records later lifecycle fixes, so pin the version used by your app and read its changelog before upgrading again. The official migration guide is the source of truth for API changes; this post’s decision rule is practical: cancel screen-owned work, guard unavoidable races, and move durable workflows out of screen-owned providers.
