---
layout: post
title: "go_router onExit - Guard Unsaved Flutter Forms Before Leaving a Route"
description: "Learn how to use go_router onExit to protect unsaved Flutter form changes across back navigation, deep links, and programmatic route changes."
date: 2026-08-19
tags: [go_router, navigation, Flutter]
comments: true
share: true
---

![go_router route guard protecting a Flutter form before navigation](/assets/images/go-router-stateful-shell-route.png)

The safest place to protect an unsaved Flutter form is the route's `onExit` callback. It runs when go_router is about to remove that route, so the same confirmation logic can cover the app bar back button, Android system back, browser history, and `context.go()` calls.

## Why a widget-only pop guard is easy to bypass

It is tempting to put `WillPopScope` or `PopScope` directly around the form. That handles some back gestures, but it does not describe every way a declarative router can replace the current route. A call such as `context.go('/home')` can change the route match without looking like a normal `Navigator.pop` from the form widget.

`onExit` is attached to the route itself. The callback returns `true` to allow removal and `false` to keep the route in place.

| Navigation attempt | Route `onExit` | Form widget callback |
| --- | --- | --- |
| App bar `context.pop()` | Runs | Usually runs indirectly |
| Android back gesture | Runs | Depends on route setup |
| Browser back button | Runs | Not a reliable boundary |
| `context.go('/home')` | Runs | Can be skipped |

The route is the useful boundary because it owns the decision to leave, not just one visual back button.

## Keep the dirty state in the form

The form only needs to expose whether its current values differ from the last saved values. A `ValueNotifier<bool>` keeps the example small and avoids rebuilding the whole route whenever one field changes.

```dart
class ProfileForm extends StatefulWidget {
  const ProfileForm({super.key});

  @override
  State<ProfileForm> createState() => _ProfileFormState();
}

class _ProfileFormState extends State<ProfileForm> {
  final nameController = TextEditingController();
  final isDirty = ValueNotifier(false);

  void markDirty() {
    if (!isDirty.value) isDirty.value = true;
  }

  Future<void> save() async {
    await repository.saveProfile(nameController.text);
    isDirty.value = false;
  }

  @override
  void dispose() {
    nameController.dispose();
    isDirty.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: nameController,
      onChanged: (_) => markDirty(),
      decoration: const InputDecoration(labelText: 'Name'),
    );
  }
}
```

In a real screen, expose the notifier through a controller, provider, or inherited model instead of reaching into the `State` object from the router. The important detail is that saving resets the dirty flag before navigation continues.

## Put the asynchronous confirmation on `GoRoute`

The callback receives the route context and state, which is enough to show a dialog and decide whether the route may be removed.

```dart
GoRoute(
  path: '/profile/edit',
  builder: (context, state) => const ProfileEditPage(),
  onExit: (context, state) async {
    final dirty = ProfileFormScope.of(context).isDirty.value;
    if (!dirty) return true;

    final discard = await showDialog<bool>(
      context: context,
      builder: (dialogContext) => AlertDialog(
        title: const Text('Discard changes?'),
        content: const Text('Your edits have not been saved.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(dialogContext, false),
            child: const Text('Keep editing'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(dialogContext, true),
            child: const Text('Discard'),
          ),
        ],
      ),
    );

    return discard ?? false;
  },
),
```

The `dialogContext` matters. Closing the dialog with the router's context can accidentally pop the page that is being protected. Returning `false` when the dialog is dismissed also makes a tap outside the dialog or the system back button safe by default.

## Prevent duplicate dialogs

`onExit` can be reached by more than one navigation event while a dialog is open. A guard object should serialize the decision rather than showing two dialogs on top of each other.

```dart
class UnsavedChangesGuard {
  Future<bool>? _pending;

  Future<bool> confirm(BuildContext context) {
    return _pending ??= _show(context).whenComplete(() {
      _pending = null;
    });
  }

  Future<bool> _show(BuildContext context) async {
    return await showDialog<bool>(
          context: context,
          builder: (context) => AlertDialog(
            title: const Text('Discard changes?'),
            actions: [
              TextButton(
                onPressed: () => Navigator.pop(context, false),
                child: const Text('Cancel'),
              ),
              FilledButton(
                onPressed: () => Navigator.pop(context, true),
                child: const Text('Discard'),
              ),
            ],
          ),
        ) ??
        false;
  }
}
```

This is especially useful when a `StatefulShellRoute` contains the edit page. The shell can preserve tab state while the child route independently decides whether it may exit.

## Traps worth testing

- Mark the form clean after a successful save, not when the save button is tapped. A failed request must still warn the user.
- Return `false` for a dismissed dialog. Treating dismissal as approval is an easy data-loss bug.
- Test `context.go('/home')`, `context.pop()`, a browser back action, and the Android back gesture separately. They represent different user paths even though they should reach the same guard.
- Do not call `context.go()` from inside `onExit` to perform the discard navigation. That can re-enter the guard. Return `true` and let the original navigation finish.

The useful mental model is simple: the form reports whether it is dirty, while `GoRoute.onExit` owns the permission to leave. Keeping those responsibilities separate makes route changes predictable without scattering back-button logic across every screen.
