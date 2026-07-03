---
layout: post
title: "flutter_animate PageView Motion - onboarding transitions without noisy replays"
description: "Use flutter_animate with PageView onboarding screens while keeping keys, page state, and transition timing stable across swipes."
date: 2026-07-03
tags: [flutter_animate, animation, onboarding, Flutter, UI]
comments: true
share: true
---
![flutter_animate PageView onboarding motion](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

Look at this image as a reminder that onboarding motion should guide attention, not make every swipe feel like a new app launch.

`flutter_animate` is a good fit for `PageView` onboarding, but the common mistake is replaying the full entrance animation every time a page rebuilds. A title fade, image slide, and button scale can look polished in the first pass. After the user swipes back and forth, it starts to feel jumpy.

The first version I built animated every child inside `build()`. It worked until the locale changed and all pages replayed while the `PageController` kept its current index. The content was correct, but the motion made the screen look unstable.

| Element | Motion | Limit |
|---|---|---|
| Hero image | small fade + vertical slide | once per page entry |
| Title | fade only | no large movement |
| Body copy | delayed fade | under 120ms delay |
| CTA button | scale from .98 | never bounce on rebuild |

I keep page identity explicit.

```dart
PageView.builder(
  controller: controller,
  itemCount: pages.length,
  itemBuilder: (context, index) {
    final page = pages[index];

    return OnboardingPage(
      key: ValueKey(page.id),
      page: page,
      isActive: index == activeIndex,
    );
  },
)
```

Inside the page, animate based on active state instead of assuming every rebuild is a new entrance.

```dart
class OnboardingPage extends StatelessWidget {
  const OnboardingPage({
    super.key,
    required this.page,
    required this.isActive,
  });

  final OnboardingData page;
  final bool isActive;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Image.asset(page.image)
            .animate(target: isActive ? 1 : 0)
            .fadeIn(duration: 220.ms)
            .slideY(begin: .04, end: 0),
        Text(page.title)
            .animate(target: isActive ? 1 : 0)
            .fadeIn(duration: 160.ms),
      ],
    );
  }
}
```

The trap is using `ValueKey(index)` when onboarding pages are remote-configured. If page order changes, Flutter may reuse the wrong state. Use a stable page ID from the content model. That one choice prevents a lot of weird replay bugs.

For button motion, I avoid repeated attention grabs.

```dart
FilledButton(
  onPressed: onContinue,
  child: const Text('Continue'),
).animate().fadeIn(duration: 180.ms).scaleXY(begin: .98, end: 1);
```

A short checklist:

- Use stable page IDs, not only indexes.
- Animate active page state, not every rebuild.
- Keep text motion smaller than image motion.
- Test swipe back, locale change, and theme change.
- Do not replay the CTA animation after every state update.

Onboarding animation should make the first experience feel clear. Once it distracts from reading, the motion is doing too much.
