---
layout: post
title: "hydrated_bloc v11 Migration - Recover Flutter State Without Losing Users After Storage Changes"
description: "Migrate hydrated_bloc to v11 safely, keep persisted Flutter state names stable, and recover corrupted or incompatible JSON without turning every upgrade into a logout."
date: 2026-10-23
tags: [hydrated_bloc, migration, caching, state_management, testing, Flutter]
comments: true
share: true
---

![hydrated_bloc v11 migration and persisted state recovery flow in Flutter](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

`hydrated_bloc` v11 is a small source migration, but persisted state is the part that can surprise a production app. If the app only caches disposable UI preferences, the safest recovery is to delete an invalid entry and rebuild it. If the state contains an unfinished draft, cart, or offline queue, keep the storage key stable, add an explicit payload version, and handle hydration failures per bloc. The v11 constructor change is required, but changing the key at the same time silently behaves like a fresh install.

## The v11 change that breaks the build

In v10, a `HydratedCubit` could receive its storage override positionally. In v11, that override is named so it matches `HydratedBloc`. The migration guide documents this as a breaking change:

| Old code | v11 code | What can go wrong | 
| --- | --- | --- |
| `HydratedCubit(0, storage)` | `HydratedCubit(0, storage: storage)` | Compile error after the package upgrade | 
| Default storage key | Same `storagePrefix` and runtime type | Existing state appears to vanish if the key changes | 
| Implicit JSON shape | Versioned `toJson()` payload | Old fields can crash `fromJson()` | 
| Default hydration error handling | Explicit `onHydrationError` policy | One bad entry can hide the recovery decision | 

The package migration guide is the source for the constructor change; the storage and error policy are application decisions layered on top of it. Do not solve a compile error by changing the class name or moving the cubit to a new file until you know whether that changes its storage key.

## Keep the persisted identity stable

`HydratedBloc` derives a storage key from its runtime identity unless you override `storagePrefix`. Refactors that rename a class, move a feature, or create two instances can therefore change where the JSON is read from. Give important state an intentional prefix and keep it stable across releases.

The following cubit keeps the key stable and makes the payload migration explicit. The `v` field is not provided by `hydrated_bloc`; it is a small contract owned by the app.

```dart
class CartCubit extends HydratedCubit<CartState> {
  CartCubit({required Storage storage})
      : super(const CartState.empty(), storage: storage);

  @override
  String get storagePrefix => 'checkout.cart';

  @override
  Map<String, dynamic>? toJson(CartState state) {
    return <String, dynamic>{
      'v': 2,
      'items': state.items.map((item) => item.toJson()).toList(),
    };
  }

  @override
  CartState? fromJson(Map<String, dynamic> json) {
    final version = json['v'];
    if (version == 1) {
      return CartState.fromLegacyJson(json);
    }
    if (version == 2) {
      return CartState.fromJson(json);
    }
    return null; // Unknown data is discarded, not trusted.
  }
}
```

Returning `null` from `fromJson` tells the bloc that the stored value cannot restore a state. That is a reasonable fallback for a cache. It is not a complete recovery plan for user-owned data: copy the important data to a server or a database with its own migration strategy instead of treating hydrated storage as durable storage.

## Handle a bad entry without hiding every error

The current API exposes an `onHydrationError` callback on hydrated blocs and cubits. A practical policy distinguishes malformed data from an unavailable storage backend. The exact logging destination is product-specific, but the decision should be visible in code:

```dart
class SessionCubit extends HydratedCubit<SessionState> {
  SessionCubit({required Storage storage})
      : super(
          const SessionState.signedOut(),
          storage: storage,
          onHydrationError: (error, stackTrace) {
            // Record the event, then allow the cubit to use its initial state.
            debugPrint('Session hydration failed: $error');
          },
        );

  @override
  SessionState? fromJson(Map<String, dynamic> json) {
    try {
      return SessionState.fromJson(json);
    } on FormatException {
      return null;
    }
  }
}
```

Do not log access tokens or the complete JSON payload. A useful event includes the stable prefix, payload version if it was readable, app version, and the exception type. If hydration fails because the storage directory or encryption layer is unavailable, silently resetting the state may log the user out while the underlying problem remains.

## Test the upgrade boundary, not only a clean install

A clean-install test proves that the current `toJson()` and `fromJson()` agree. It does not prove that a user carrying last month's payload can open today's build. Keep one fixture for each supported payload version and test both a valid old payload and an unknown version.

| Fixture | Expected result | Release decision |
| --- | --- | --- |
| v1 cart with renamed field | v2 cart with the same items | Safe to migrate in place | 
| v2 cart with one malformed item | Empty cart or a partial result, by policy | Show recovery UI if data matters | 
| Unknown future version | Initial state, plus telemetry | Never parse optimistically | 
| Storage read exception | Explicit fallback and an event | Investigate backend or permissions | 

For a reproducible test, inject a fake `Storage` containing the exact JSON written by the previous release. Then construct the v11 cubit and assert the state after hydration. This catches the two easy-to-miss regressions: changing `storagePrefix` during a refactor and assuming every old field is non-null.

## Upgrade checklist

- Change positional storage overrides to `storage: storage`.
- Freeze `storagePrefix` for state that must survive a class rename.
- Add a payload version before changing JSON fields.
- Return `null` for unknown or unsafe payloads rather than throwing from `fromJson`.
- Configure `onHydrationError` and record metadata without sensitive state.
- Test a stored v1 payload, malformed JSON, and a storage backend failure.
- Keep carts, drafts, and sync queues in a durable data store when losing them is unacceptable.

The useful boundary is simple: `hydrated_bloc` is convenient persisted state, not a database migration system. Apply the v11 API change, preserve the key, make the JSON contract explicit, and choose recovery behavior based on the cost of losing that particular state.

Sources: [Bloc migration guide](https://bloclibrary.dev/migration/), [hydrated_bloc API](https://pub.dev/documentation/hydrated_bloc/latest/hydrated_bloc/HydratedBloc-class.html), [hydrated_bloc package documentation](https://pub.dev/packages/hydrated_bloc).
