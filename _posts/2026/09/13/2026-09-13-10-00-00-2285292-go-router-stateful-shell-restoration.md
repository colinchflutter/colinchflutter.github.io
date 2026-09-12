---
layout: post
title: "go_router StatefulShellRoute restorationScopeId - Restore Flutter Tab Navigation After Process Death"
description: "Learn how go_router StatefulShellRoute restorationScopeId preserves Flutter tab navigation after process death, and where repository state is still required."
date: 2026-09-13
tags: [go_router, navigation, Flutter, Android, iOS]
comments: true
share: true
---

![go_router StatefulShellRoute restoring Flutter tab navigation stacks](/assets/images/go-router-stateful-shell-route.png)

`StatefulShellRoute` keeps Flutter tab navigators alive while the app is running, but that is not the same as restoring them after Android or iOS kills the process. For process restoration, the router and the app need restoration IDs at the right levels. Without them, a tab can remember `/orders/42` while the app is open, then return to the default tab after a cold restore.

## Two kinds of state that look similar

The first trap is treating every “preserved tab” result as state restoration. These mechanisms solve different failures.

| Situation | Mechanism | What it preserves |
| --- | --- | --- |
| Switch from Home to Settings and back | `StatefulShellRoute` | In-memory branch Navigators and their stacks |
| OS kills the app in the background | Restoration IDs | Serializable Navigator history and route state |
| Product data is no longer in memory | Repository or local database | Application data used to rebuild the page |

I use `StatefulShellRoute` for a good tab experience and restoration configuration for a restart boundary. Neither mechanism turns a remote API response into permanent storage.

## Configure the router and the app together

The top-level router and `MaterialApp.router` need matching restoration scopes. The shell can then add its own scope for the parallel branch navigators.

Here is a compact setup for a Home and Orders tab:

```dart
final _rootNavigatorKey = GlobalKey<NavigatorState>();
final _homeNavigatorKey = GlobalKey<NavigatorState>();
final _ordersNavigatorKey = GlobalKey<NavigatorState>();

final router = GoRouter(
  navigatorKey: _rootNavigatorKey,
  restorationScopeId: 'app_router',
  routes: [
    StatefulShellRoute.indexedStack(
      restorationScopeId: 'main_tabs',
      builder: (context, state, navigationShell) {
        return Scaffold(
          body: navigationShell,
          bottomNavigationBar: NavigationBar(
            selectedIndex: navigationShell.currentIndex,
            onDestinationSelected: (index) {
              navigationShell.goBranch(index: index);
            },
            destinations: const [
              NavigationDestination(
                icon: Icon(Icons.home_outlined),
                label: 'Home',
              ),
              NavigationDestination(
                icon: Icon(Icons.receipt_long_outlined),
                label: 'Orders',
              ),
            ],
          ),
        );
      },
      branches: [
        StatefulShellBranch(
          navigatorKey: _homeNavigatorKey,
          routes: [
            GoRoute(
              path: '/home',
              builder: (context, state) => const HomePage(),
            ),
          ],
        ),
        StatefulShellBranch(
          navigatorKey: _ordersNavigatorKey,
          routes: [
            GoRoute(
              path: '/orders',
              builder: (context, state) => const OrdersPage(),
              routes: [
                GoRoute(
                  path: 'detail/:orderId',
                  builder: (context, state) => OrderPage(
                    orderId: state.pathParameters['orderId']!,
                  ),
                ),
              ],
            ),
          ],
        ),
      ],
    ),
  ],
);

MaterialApp.router(
  restorationScopeId: 'material_app',
  routerConfig: router,
);
```

The important part is not the `IndexedStack` itself. `StatefulShellRoute.indexedStack` supplies the default branch container, while `restorationScopeId` gives the shell a stable restoration boundary. The root router and the app scope are separate configuration points, so I do not add one ID and assume the entire tree is covered.

## Keep route identity restorable

A restored route still needs enough information to be rebuilt. A path such as `/orders/detail/42` carries the order identity in the URL, so the page can request order `42` again. Passing the entire `Order` object through `extra` may work during a normal tap, but it is not a reliable restoration contract.

```dart
class OrderPage extends StatefulWidget {
  const OrderPage({required this.orderId, super.key});

  final String orderId;

  @override
  State<OrderPage> createState() => _OrderPageState();
}

class _OrderPageState extends State<OrderPage>
    with RestorationMixin {
  final RestorableBool _showNotes = RestorableBool(false);

  @override
  String? get restorationId => 'order_${widget.orderId}';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_showNotes, 'show_notes');
  }

  @override
  void dispose() {
    _showNotes.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Order ${widget.orderId}')),
      body: SwitchListTile(
        title: const Text('Show notes'),
        value: _showNotes.value,
        onChanged: (value) => setState(() => _showNotes.value = value),
      ),
    );
  }
}
```

The route parameter identifies the page; `RestorationMixin` stores the small widget-level value. The order title, line items, and payment status still belong to application data. After restoration, the screen should show a loading state and fetch the order from a repository instead of expecting the old Dart object to be present.

## What commonly fails

`StatefulShellRoute` without restoration IDs preserves tab stacks only until the process disappears. That is a lifecycle boundary, not a bug in `goBranch`.

Another mistake is changing restoration IDs as part of a refactor. A new ID is a new restoration bucket, so old state will not match. Keep IDs stable, unique within the restoration tree, and tied to the meaning of the scope rather than a temporary widget label.

I also avoid putting volatile server data into restorable properties. Store identifiers, filters, selected modes, and small UI flags. Reload large or sensitive data from the repository and validate it against the current account.

## A short verification matrix

Test the boundaries separately instead of checking only that a tab switch feels correct:

- Open `/orders/detail/42`, switch to Home, and return. The Orders branch should still show order `42`.
- Enable “Show notes”, background the app, and simulate process death. The route and the small UI flag should be eligible for restoration.
- Launch a direct deep link such as `/orders/detail/99`. The repository should fetch `99` even when no prior restoration data exists.
- Change the account or remove order `42`. Restoration must not bypass authorization or data validation.

`StatefulShellRoute` is the right tool for parallel in-memory navigation stacks. `restorationScopeId` extends that behavior across a process boundary, while URL parameters and repositories make the restored page reconstructible. Keeping those responsibilities separate prevents a tab state fix from becoming a data persistence bug.

References: [StatefulShellRoute API](https://pub.dev/documentation/go_router/latest/go_router/StatefulShellRoute-class.html), [go_router configuration](https://pub.dev/documentation/go_router/latest/topics/Configuration-topic.html), and [Flutter state restoration](https://docs.flutter.dev/ui/adaptive-responsive/general).
