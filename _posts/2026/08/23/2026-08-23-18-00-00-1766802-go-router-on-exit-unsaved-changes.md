---
layout: post
title: "go_router onExit - Protect Unsaved Flutter Forms Before Leaving"
description: "Learn how go_router onExit blocks Flutter route removal with async unsaved-change confirmation and how to test back navigation safely."
date: 2026-08-23
tags: [go_router, navigation, state_management, testing, Flutter]
comments: true
share: true
---

![go_router onExit protecting an unsaved Flutter form before route removal](/assets/images/go-router-on-exit-route-guard.png)

`go_router`'s `onExit` is the right place to protect a route that contains unsaved work. It runs when the route is being removed and returns `true` to allow the removal or `false` to keep the route. Because the callback accepts `FutureOr<bool>`, the decision can wait for a real confirmation dialog without putting navigation calls inside the form widget.

## Why a form's `WillPopScope` is not enough

I first handled this with `WillPopScope` inside the editor page. That covered the Android back button, but it did not describe the route policy itself. A `context.go('/home')` call, a browser back action, and a nested shell navigation can all remove the route through different paths.

| Requirement | Better location |
| --- | --- |
| Decide whether the current route may be removed | `GoRoute.onExit` |
| Send an unauthenticated user to sign-in | `redirect` |
| Decide whether a destination may be entered | `onEnter` |
| Validate a form while the user edits | Form state/controller |

The important boundary is that `onExit` returns a decision. It should not call `context.pop()` or `context.go()` to escape the callback; that creates a second navigation event while the first one is still being resolved.

## Add an asynchronous exit guard

The example uses a small controller with a dirty flag. The route callback only asks the controller for a decision, so the same rule works for the system back button, an app-bar back button, and imperative navigation.

```dart
class EditorController extends ChangeNotifier {
  bool _dirty = false;

  bool get isDirty => _dirty;

  void markChanged() {
    if (!_dirty) {
      _dirty = true;
      notifyListeners();
    }
  }

  void markSaved() {
    if (_dirty) {
      _dirty = false;
      notifyListeners();
    }
  }
}

Future<bool> confirmExit(
  BuildContext context,
  EditorController editor,
) async {
  if (!editor.isDirty) return true;

  final discard = await showDialog<bool>(
    context: context,
    builder: (context) => AlertDialog(
      title: const Text('Discard changes?'),
      content: const Text('Your unsaved edits will be lost.'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('Keep editing'),
        ),
        FilledButton(
          onPressed: () => Navigator.pop(context, true),
          child: const Text('Discard'),
        ),
      ],
    ),
  );

  return discard ?? false;
}
```

Register the callback on the route that owns the editor. In a real app, I provide the controller with a scoped provider rather than creating it inside the route callback. That keeps the dirty state alive while the route is mounted and makes it possible for the save button to call `markSaved()`.

```dart
GoRoute(
  path: '/editor/:documentId',
  builder: (context, state) => const EditorPage(),
  onExit: (context, state) async {
    final editor = context.read<EditorController>();
    return confirmExit(context, editor);
  },
),
```

The callback receives the route's `BuildContext` and `GoRouterState`. It is tempting to read the document ID from a page widget instead, but the route state is the more stable source when the guard needs to log or load route-specific information.

## Keep the guard side-effect free

There are two failure cases that looked harmless during implementation:

- Showing a dialog after the route has already been removed. This usually happens when the guard starts an unawaited future and immediately returns `true`.
- Returning `true` after the dialog is dismissed. A barrier tap or system cancellation should mean “keep editing”, not “discard”. Returning `discard ?? false` makes that policy explicit.

The guard also needs to handle a widget that is no longer mounted after an asynchronous save. Keep the save operation outside `onExit`, or check `context.mounted` before using the context again. `onExit` is a permission check, not a cleanup hook.

## Test every route-removal path

The most useful test does not tap only the visible back icon. It makes the editor dirty, triggers router navigation, and verifies that the destination is not reached when the confirmation returns `false`.

```dart
testWidgets('keeps dirty editor when exit is rejected', (tester) async {
  final router = createTestRouter();
  await tester.pumpWidget(TestApp(router: router));
  await router.push('/editor/42');
  await tester.pumpAndSettle();

  await tester.enterText(find.byType(TextField), 'draft');
  await router.push('/home');
  await tester.pumpAndSettle();

  expect(find.text('Discard changes?'), findsOneWidget);
  await tester.tap(find.text('Keep editing'));
  await tester.pumpAndSettle();

  expect(router.state.uri.path, '/editor/42');
});
```

Also verify the opposite result, browser back on Flutter Web, and an editor nested under `ShellRoute`. The last case matters because the shell's persistent navigator can make it look as if the page disappeared without the expected callback. I check the installed `go_router` version and run this test against the actual nested route tree rather than a simplified mock.

The practical rule is simple: use `redirect` to change a location, `onEnter` to reject entry, and `onExit` to approve removal. Once unsaved-work policy lives at the route boundary, every navigation path gets the same answer and the editor stays focused on editing rather than navigation plumbing.

Reference: [GoRoute API](https://pub.dev/documentation/go_router/latest/go_router/GoRoute-class.html) and [go_router API](https://pub.dev/documentation/go_router/latest/go_router/).
