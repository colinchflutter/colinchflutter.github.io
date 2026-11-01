---
layout: post
title: "graphql_flutter Logout Cache - Stop Flutter HiveStore Leaking the Previous User"
description: "Fix graphql_flutter logout bugs where HiveStore serves the previous user's data by separating cache reset, token removal, and post-logout refetch behavior."
date: 2026-11-01
tags: [graphql_flutter, caching, authentication, networking, testing, Flutter]
comments: true
share: true
---

![graphql_flutter logout cache reset flow for a Flutter app](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

The safe logout choice for a Flutter app using `graphql_flutter` with `HiveStore` is to clear the GraphQL cache as part of the session boundary, then rebuild or refetch with the new authentication state. This applies when cached entities contain account-scoped data. It is unnecessary for an app using only `InMemoryStore` with no sensitive data, but it is still useful to stop stale widgets from showing after logout.

## Why the bug appears only after a restart

`GraphQLCache` uses `InMemoryStore` by default. Switching to `HiveStore` makes the normalized cache persistent, so a screen can render the previous user's profile before the network request finishes. Removing a bearer token does not remove entities already stored under keys such as `User:42`.

There is a second trap: clearing the token and immediately calling `resetStore()` can refetch active queries while the widget tree still represents the old session. The result is a race between logout navigation, the auth link, and cache broadcasts.

| Operation | What it changes | Safe logout use |
|---|---|---|
| Remove token | Future request headers | Required, but not cache cleanup |
| `client.resetStore()` | Clears store and refetches active queries | Use only with a controlled auth state |
| `client.resetStore(refetchQueries: false)` | Clears store without refetching | Good before replacing the session tree |
| New `GraphQLClient` | Cache, links, and watchers | Best isolation for account switching |

## Put logout at the session boundary

The following sequence clears persisted entities before the unauthenticated screen can read them. The exact token repository is application-specific; the important part is the ordering.

```dart
Future<void> signOut() async {
  sessionState.value = SessionState.loggingOut;

  // Do not let resetStore refetch protected queries with a half-removed session.
  await client.resetStore(refetchQueries: false);
  await tokenStore.clear();

  // Recreate the client or publish the login route only after the cache is empty.
  sessionState.value = SessionState.signedOut;
}
```

If the app must keep one client instance, disable the old query widgets before publishing `signedOut`. If the app supports account switching, creating a new `GraphQLClient` with a new `HiveStore` is easier to reason about than trying to partition every normalized entity key manually.

## The authenticated link must read current state

Do not capture the first access token when constructing the client:

```dart
final link = AuthLink(getToken: () async {
  final token = await tokenStore.read();
  return token == null ? null : 'Bearer $token';
});
```

The callback is evaluated when a request is made. That matters after logout, login as another account, and refresh-token rotation. A cached token in the closure can make a post-logout refetch look like a cache failure even though the store was cleared correctly.

## Reproduce the failure before shipping

- Write account A's `User` and `Orders` queries into `HiveStore`.
- Log out, kill the process, and start the signed-out route.
- Log in as account B and verify that no A entity is painted before B's response.
- Switch accounts while an active query is loading; assert that its result cannot update the new session.
- Inspect the resolved `graphql_flutter` version and keep the cache-reset behavior in an integration test.

`resetStore()` is not a server-side revocation mechanism. Revoke or rotate credentials on the backend as required, and treat local cache clearing as a privacy and UI consistency boundary. Sources checked on 2026-11-01: [graphql_flutter package documentation](https://pub.dev/documentation/graphql_flutter/latest/), [GraphQLClient API](https://pub.dev/documentation/graphql_flutter/latest/graphql_flutter/GraphQLClient-class.html), and [GraphQLCache API](https://pub.dev/documentation/graphql_flutter/latest/graphql_flutter/GraphQLCache-class.html).
