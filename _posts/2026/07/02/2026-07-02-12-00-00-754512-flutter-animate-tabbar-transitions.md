---
layout: post
title: "flutter_animate TabBar Transitions - content motion without fighting TabController"
description: "Build Flutter TabBar content transitions with flutter_animate while keeping TabController, swipe gestures, and rebuild behavior predictable."
date: 2026-07-02
tags: [flutter_animate, animation, navigation, performance]
comments: true
share: true
---
![Flutter app tabs with animated content panels](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

The cleanest way to animate `TabBar` content is to let `TabController` keep ownership of tab state and use `flutter_animate` only on the content that changes. Do not replace `TabBarView` with a pile of manual `AnimatedSwitcher` logic unless you also want to rebuild swipe gestures, page caching, and index synchronization.

The mistake I made the first time was animating the whole `TabBarView`. It looked fine on tap, then felt wrong during horizontal swipes. The page was already moving because `TabBarView` uses a page-style transition internally, and my extra `.slideX()` added a second movement on top. The result was a slightly rubbery transition that looked designed in screenshots and broken in the hand.

## Keep the TabController boring

Use the normal Flutter tab structure. The controller should decide which page is active, the `TabBar` should render the labels, and `TabBarView` should keep its gesture behavior. `flutter_animate` belongs inside each tab page.

Here is the baseline layout before adding motion.

```dart
class AccountTabs extends StatelessWidget {
  const AccountTabs({super.key});

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Account'),
          bottom: const TabBar(
            tabs: [
              Tab(text: 'Overview'),
              Tab(text: 'Activity'),
              Tab(text: 'Billing'),
            ],
          ),
        ),
        body: const TabBarView(
          children: [
            OverviewTab(),
            ActivityTab(),
            BillingTab(),
          ],
        ),
      ),
    );
  }
}
```

That code is intentionally ordinary. The tab bar works with keyboard focus, accessibility semantics, swipe gestures, and controller state. The animation layer should not disturb any of that.

## Animate the page contents, not the tab view

The pattern I use is a small wrapper per tab. It animates the top-level content when the tab is first built, but it does not try to own the selected index.

This wrapper gives each tab a short entrance without changing `TabBarView` behavior.

```dart
class TabEntrance extends StatelessWidget {
  const TabEntrance({
    required this.child,
    required this.identity,
    super.key,
  });

  final Widget child;
  final Object identity;

  @override
  Widget build(BuildContext context) {
    return KeyedSubtree(
      key: ValueKey(identity),
      child: child
          .animate()
          .fadeIn(duration: 180.ms, curve: Curves.easeOut)
          .slideY(
            begin: 0.025,
            end: 0,
            duration: 220.ms,
            curve: Curves.easeOutCubic,
          ),
    );
  }
}
```

Then each tab opts into the same behavior.

```dart
const TabBarView(
  children: [
    TabEntrance(identity: 'overview', child: OverviewTab()),
    TabEntrance(identity: 'activity', child: ActivityTab()),
    TabEntrance(identity: 'billing', child: BillingTab()),
  ],
)
```

The `identity` is not there for decoration. It makes the replay rule explicit. If you change it, the subtree is treated as new and the entrance runs again. If you keep it stable, the tab content does not restart its animation on every harmless parent rebuild.

## When you need replay on every tab selection

Some screens should replay when selected: analytics panels, empty states, onboarding tabs, or short profile sections where the motion helps the user notice the change. For that, listen to the current tab index and pass a changing identity into the active tab.

This version rebuilds only when the selected tab index changes.

```dart
class AnimatedAccountTabs extends StatefulWidget {
  const AnimatedAccountTabs({super.key});

  @override
  State<AnimatedAccountTabs> createState() => _AnimatedAccountTabsState();
}

class _AnimatedAccountTabsState extends State<AnimatedAccountTabs>
    with SingleTickerProviderStateMixin {
  late final TabController _controller;
  int _selectedIndex = 0;

  @override
  void initState() {
    super.initState();
    _controller = TabController(length: 3, vsync: this);
    _controller.addListener(_handleTabChange);
  }

  void _handleTabChange() {
    if (_controller.indexIsChanging) return;
    if (_selectedIndex == _controller.index) return;
    setState(() => _selectedIndex = _controller.index);
  }

  @override
  void dispose() {
    _controller.removeListener(_handleTabChange);
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        bottom: TabBar(
          controller: _controller,
          tabs: const [
            Tab(text: 'Overview'),
            Tab(text: 'Activity'),
            Tab(text: 'Billing'),
          ],
        ),
      ),
      body: TabBarView(
        controller: _controller,
        children: [
          TabEntrance(
            identity: 'overview-$_selectedIndex',
            child: const OverviewTab(),
          ),
          TabEntrance(
            identity: 'activity-$_selectedIndex',
            child: const ActivityTab(),
          ),
          TabEntrance(
            identity: 'billing-$_selectedIndex',
            child: const BillingTab(),
          ),
        ],
      ),
    );
  }
}
```

This is slightly blunt because every tab gets a changed identity when selection changes. In small tab sets, that is fine. In heavy tabs with charts, video thumbnails, or expensive lists, keep the identity stable and animate only a small header inside the active page.

## Avoid sideways motion during swipes

Vertical movement is safer than horizontal movement inside tabs. `TabBarView` already moves horizontally, so a child-level `.slideX()` can feel like a lagging duplicate of the route motion. I usually use a tiny `.slideY()` with fade, or a scale from `0.98` to `1.0` for dense dashboard content.

This is the version I ship most often.

```dart
child
    .animate()
    .fadeIn(duration: 160.ms, curve: Curves.easeOut)
    .scale(
      begin: const Offset(0.98, 0.98),
      end: const Offset(1, 1),
      duration: 200.ms,
      curve: Curves.easeOutCubic,
    )
```

It is visible enough to soften the swap, but not so visible that the tab interaction feels delayed. Anything over `250.ms` starts to feel slow when users tap between tabs quickly.

## State and performance notes

Do not put form fields, scroll controllers, or paginated list state inside a keyed wrapper that changes every selection unless you are willing to reset them. A key change tells Flutter this is a different subtree. That is useful for replaying animation, but it can also wipe local state.

For scroll-heavy tabs, prefer stable page widgets:

```dart
TabEntrance(
  identity: 'activity',
  child: const ActivityListTab(),
)
```

Then animate a lightweight header or summary row inside `ActivityListTab` when fresh data arrives. That keeps the list position intact and avoids rebuilding a large viewport just to get a 200ms fade.

The practical rule: animate the smallest thing that communicates the tab change. For a settings screen, animate the section title and first row. For an analytics screen, animate the summary cards, not the full chart canvas. For a tab with editable fields, skip replay entirely after the first build.

## Quick checklist

- Keep `TabController`, `TabBar`, and `TabBarView` in charge of navigation.
- Put `flutter_animate` inside each tab page, not around the whole `TabBarView`.
- Use stable keys when tab state must survive.
- Change keys only when replay is worth the rebuild cost.
- Prefer fade, slight vertical movement, or small scale over horizontal slide.
- Keep tab transition durations around `160.ms` to `220.ms`.

`flutter_animate` works best here as a content polish layer. Let Flutter handle tab mechanics, then add short, local motion where the user needs feedback. The difference is small in code, but obvious in the final feel: tabs stay responsive, swipe gestures remain native, and the animation does not fight the controller.
