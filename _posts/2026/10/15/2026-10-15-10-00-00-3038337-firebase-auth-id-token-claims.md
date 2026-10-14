---
layout: post
title: "firebase_auth authStateChanges vs idTokenChanges - Fix Stale Flutter Roles After Custom Claims Change"
description: "Choose the right firebase_auth stream for Flutter login state, token refresh, and custom claims, then force a safe role update without signing users out."
date: 2026-10-15
tags: [firebase_auth, authentication, migration, state_management, Flutter]
comments: true
share: true
---

![firebase_auth stream selection for Flutter authentication and custom claims](https://pub.dev/static/img/pub-dev-icon.svg)

If a Flutter screen shows a stale role after Firebase Admin changes a user's custom claims, `authStateChanges()` is the wrong signal. Use `authStateChanges()` for signed-in or signed-out UI, `idTokenChanges()` when token refresh matters, and explicitly call `getIdTokenResult(true)` when a role change must be visible now. This does not apply to profile data changed only on the server; that path also needs `currentUser.reload()`.

## The three streams do different jobs

The confusing part is that all three streams emit a `User?`, so a `StreamBuilder<User?>` looks interchangeable. Their event boundaries are not interchangeable.

| Stream | Emits on | Good fit | Not enough for |
| --- | --- | --- | --- |
| `authStateChanges()` | initial restore, sign-in, sign-out | route guard and session shell | token claim changes, profile edits |
| `idTokenChanges()` | auth changes plus token refresh | API authorization and claim-aware state | an immediate Admin claim update |
| `userChanges()` | token changes plus local profile/provider updates | account settings screens | Admin-side changes without reload |

Firebase documents that custom claims arrive through a newly issued ID token. Waiting for the normal token expiry makes a subscription or role screen eventually correct, but it is a poor handoff after an admin action.

## Reproducible role-refresh boundary

Assume a callable function grants `editor: true`, then the app returns to a role-gated screen. Reading the old token immediately can still produce `editor: false`. The refresh must happen before deriving UI state.

The following service keeps the stream for background changes and exposes an explicit operation for a just-completed role mutation.

```dart
import 'package:firebase_auth/firebase_auth.dart';

class AuthRoleController {
  AuthRoleController(this.auth);

  final FirebaseAuth auth;

  Stream<User?> get sessionChanges => auth.authStateChanges();

  Stream<User?> get tokenChanges => auth.idTokenChanges();

  Future<Map<String, dynamic>> refreshClaimsNow() async {
    final user = auth.currentUser;
    if (user == null) {
      throw StateError('A signed-in user is required to refresh claims.');
    }

    final result = await user.getIdTokenResult(true);
    return result.claims ?? const <String, dynamic>{};
  }
}
```

Call `refreshClaimsNow()` only after the backend confirms the claim update. A forced refresh before the server mutation completes can correctly return the old role. After the result returns, update the app's role state from `claims`; do not infer authorization from a cached boolean created before the refresh.

## Avoid the silent false-negative cases

- A route guard based only on `authStateChanges()` can keep the old authorization branch because the user is still signed in.
- `idTokenChanges()` is the right passive listener, but it does not guarantee an immediate event just because an Admin SDK call finished.
- `userChanges()` detects local `updateProfile()`, `reload()`, and provider changes. It still does not magically observe a profile or claim mutation made through the Admin SDK.
- Backend rules remain authoritative. A role in Flutter is a display and navigation hint, not permission enforcement.

Use this small decision checklist:

| Requirement | Choice |
| --- | --- |
| Show login or logout state | `authStateChanges()` |
| Rebuild API-related state after token rotation | `idTokenChanges()` |
| Refresh a role immediately after an admin mutation | `getIdTokenResult(true)` |
| Read an Admin-side display-name change | `currentUser.reload()` |
| Enforce access to data | Firebase Security Rules or backend verification |

The practical fix is to separate session state from authorization state. Keep the cheap session stream for navigation, listen to token changes for passive updates, and make claim refresh an explicit boundary after a known server mutation. That removes the common “the claim was set, but Flutter still says false” race without forcing a sign-out and sign-in cycle.

Sources checked 2026-10-15: [Firebase Auth for Flutter](https://firebase.google.com/docs/auth/flutter/start), [Firebase custom claims propagation](https://firebase.google.com/docs/auth/admin/custom-claims), and the [FlutterFire `FirebaseAuth` API](https://pub.dev/documentation/firebase_auth/latest/firebase_auth/FirebaseAuth-class.html).
