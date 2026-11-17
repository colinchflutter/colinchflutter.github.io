---
layout: post
title: "firebase_auth Anonymous Accounts - Link Credentials Without Losing Local Data"
description: "Handle Flutter firebase_auth anonymous upgrades, credential collisions, and safe data transfer without deleting the guest account too early."
date: 2026-11-18
tags: [firebase_auth, authentication, migration, data, Android, iOS]
comments: true
share: true
---
![A phone login flow with a guest account being upgraded to a permanent Firebase Auth account](https://images.unsplash.com/photo-1556742049-0cfed4f6a45d?auto=format&fit=crop&w=1600&q=80)

The safest `firebase_auth` flow for a Flutter guest user is to call `linkWithCredential()` while the anonymous user is still signed in, then handle `credential-already-in-use` as an account-collision flow. This fits apps that let people build a cart, draft, or offline profile before signing in. It does not fit an app that can freely discard guest data; in that case, a fresh sign-in is simpler.

## The failure is an identity problem, not a button problem

`linkWithCredential()` keeps the current Firebase UID when it succeeds. That means Firestore documents, Storage paths, and security rules based on that UID can continue to work. Calling `signInWithCredential()` instead can switch to a different UID, so the guest data appears to have disappeared even though it is still stored under the anonymous account.

The dangerous case is that the provider credential already belongs to another Firebase user. Firebase returns `credential-already-in-use` (or a related account-exists error), and Firebase Auth does not automatically merge the two users.

| Situation | Keep current UID? | Correct action |
| --- | ---: | --- |
| Anonymous user + unused Google/email credential | Yes | `linkWithCredential()` |
| Credential already belongs to a permanent user | No automatic merge | Authenticate the existing user, then transfer allowed data |
| Credential cannot be recreated after a failed OAuth attempt | Not safely | Ask the user to authenticate again; do not cache raw secrets |
| Guest data is disposable | Not required | Sign in and delete the guest account after confirmation |

## Keep the upgrade operation explicit

The code below separates credential creation from account linking. `buildCredential()` represents the provider SDK result; it must return a fresh `AuthCredential` for the current sign-in attempt.

```dart
Future<UpgradeResult> upgradeGuestAccount({
  required AuthCredential credential,
}) async {
  final auth = FirebaseAuth.instance;
  final guest = auth.currentUser;

  if (guest == null) {
    return const UpgradeResult.signedOut();
  }
  if (!guest.isAnonymous) {
    return UpgradeResult.alreadyPermanent(guest.uid);
  }

  final guestUid = guest.uid;

  try {
    final linked = await guest.linkWithCredential(credential);
    return UpgradeResult.linked(linked.user!.uid);
  } on FirebaseAuthException catch (error) {
    switch (error.code) {
      case 'credential-already-in-use':
      case 'email-already-in-use':
      case 'account-exists-with-different-credential':
        // The guest account is still signed in here. Keep its UID and data.
        return UpgradeResult.conflict(guestUid);
      case 'provider-already-linked':
        return UpgradeResult.retryable('This provider is already linked.');
      case 'invalid-credential':
        return UpgradeResult.retryable('The credential expired.');
      default:
        rethrow;
    }
  }
}
```

The `conflict` result should move the user into an existing-account sign-in screen. Do not silently call `signInWithCredential()` and assume that the guest cart has moved. After sign-in, the app has a different `currentUser.uid`; that is the point where a deliberate data transfer is needed.

## Transfer data before deleting the guest user

A safe transfer has three properties: it is authorized by the newly signed-in user, it can be retried, and it does not delete the source before the destination is confirmed. A practical server-side shape is:

```text
1. Capture guestUid before link/sign-in changes currentUser.
2. Sign in to the existing permanent account.
3. Send guestUid and a transfer operation ID to a trusted backend.
4. Backend verifies both Firebase ID tokens and ownership rules.
5. Copy only transferable records to permanentUid with idempotent writes.
6. Mark the operation complete and return the result.
7. Delete the anonymous account only after the transfer is confirmed.
```

Use an operation ID such as `guestUid + permanentUid + transferVersion`, or a server-generated UUID. A retry must not create two copies of a purchase draft or apply the same credit twice. Sensitive data, subscription state, and anything requiring audit history should be transferred by a backend transaction or queue, not by a client loop over documents.

If the data is stored in Firestore, also decide whether a record is copied, reassigned, or left under the guest UID. Reassigning a document can break existing references and rules. Copying into a permanent-user namespace is easier to retry, but it requires a duplicate-resolution policy.

## Platform edge cases worth testing

- Google sign-in may be able to create a fresh credential from the returned ID token, but a failed Apple flow can involve a nonce that must not be reused. Treat a collision as a new authentication attempt when the provider cannot safely recreate the credential.
- `currentUser` can change after a successful sign-in. Store `guestUid` before calling any method that may change the session.
- A second tap can start two upgrade requests. Disable the action or guard it with a single-flight future; two concurrent transfers make collision handling much harder.
- Test an existing account, an unused credential, a cancelled provider sheet, an expired credential, and a process restart while the guest has unsynced data.
- Anonymous accounts are not a backup. If the device is cleared before the guest account is upgraded, its UID and private data may be unrecoverable.

The useful invariant is small: link when the credential is unused; authenticate the existing account when it is already owned; transfer data with server-side authorization; delete the guest account only after the transfer succeeds. That keeps identity migration separate from UI success messages and makes a retry recoverable.

Sources checked on 2026-11-18: [Firebase account linking](https://firebase.google.com/docs/auth/flutter/account-linking), [Firebase Auth error handling](https://firebase.google.com/docs/auth/flutter/errors), and the [`firebase_auth` User API](https://pub.dev/documentation/firebase_auth/latest/firebase_auth/User-class.html).
