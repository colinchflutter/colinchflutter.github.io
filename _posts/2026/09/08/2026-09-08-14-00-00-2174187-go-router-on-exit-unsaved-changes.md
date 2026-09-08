---
layout: post
title: "go_router onExit - Guard Flutter Routes with Unsaved Changes"
description: "Learn how to use go_router onExit in Flutter to block route removal, confirm unsaved changes, and handle async exit guards safely."
date: 2026-09-08
tags: [go_router, navigation, testing, Flutter]
comments: true
share: true
---

![go_router onExit route guard for unsaved changes in Flutter](/assets/images/go-router-on-exit-route-guard.png)

`GoRoute.onExit` is the right boundary for asking, “Can this route leave now?” in Flutter. It returns a `FutureOr<bool>`, so an editor can allow a normal pop immediately or wait for a confirmation dialog when the document is dirty. That keeps unsaved-change policy inside the route instead of scattering checks across every back button.

## The problem with checking only the AppBar button

An editor can be left in several ways: the system back gesture, `context.pop()`, a browser back action, or a parent route replacing the current stack. Protecting only an `IconButton` misses some of those paths. `onExit` is invoked when the route is removed from GoRouter's route history, which makes it a better last gate.

| Mechanism | Good for | Weak spot |
|---|---|---|
| `redirect` | Choosing a destination from app state | It is not an exit confirmation |
| `PopScope` | Widget-level back behavior | It does not cover every router change |
| `onExit` | Allowing or blocking route removal | It should stay focused on exit policy |

## A reusable async exit guard

The callback should return `true` when there is nothing to lose. The dialog is only needed for a dirty draft.

```dart
GoRoute(
  path: '/editor',
  builder: (context, state) => const EditorPage(),
  onExit: (context, state) async {
    final editor = EditorScope.of(context);

    if (!editor.hasUnsavedChanges) {
      return true;
    }

    final leave = await showDialog<bool>(
      context: context,
      builder: (dialogContext) => AlertDialog(
        title: const Text('Discard draft?'),
        content: const Text('Your changes have not been saved.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(dialogContext, false),
            child: const Text('Stay'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(dialogContext, true),
            child: const Text('Discard'),
          ),
        ],
      ),
    );

    return leave ?? false;
  },
),
```

The `false` fallback matters. A dismissed dialog returns `null`; treating that as permission would make a tap outside the dialog discard the draft. I initially returned `leave == true` only at the call site and later moved the rule into the guard so every exit path behaved consistently.

## Keep the guard free of save logic

`onExit` should decide whether leaving is allowed. It should not silently save, mutate the route, or call `context.go()` from inside the callback. If the user chooses to save, finish that operation in the editor first, update `hasUnsavedChanges`, and then retry the exit action. Mixing save and navigation in one callback can produce a second dialog or a stale dirty flag.

There is another boundary to test with `ShellRoute` and `StatefulShellRoute`. A branch route and the shell use different navigator layers, so place `onExit` on the route that actually owns the draft. A guard on the shell is too broad if only one child contains unsaved data.

## Test the routes, not just the dialog

At minimum, verify these cases:

- a clean editor returns `true` without opening a dialog;
- a dirty editor returns `false` when “Stay” is tapped;
- a dirty editor returns `true` after “Discard”;
- dialog dismissal is treated as `false`;
- Android back, `context.pop()`, and a route replacement use the same policy;
- a shell branch does not accidentally guard an unrelated branch.

`onExit` is a permission check for route removal, not a replacement for `redirect`. Use it when the current screen owns information that may be lost, keep the result deterministic, and make cancellation the safe default.
