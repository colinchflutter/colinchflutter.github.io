---
layout: post
title: "flutter_animate Hero Route Transitions - Avoid Double Motion Between Screens"
description: "Learn how to combine Flutter Hero transitions with flutter_animate so shared elements move once and destination content enters without replay bugs."
date: 2026-07-20
tags: [flutter_animate, animation, navigation, performance]
comments: true
share: true
---

![Flutter Hero route transition with a shared profile card](https://colinchflutter.github.io/assets/images/flutter-animate-hero-route-transition.png)

*The important visual boundary is simple: the profile avatar travels with `Hero`, while the destination details fade in after the route settles.*

Flutter `Hero` should own the shared element's movement between routes, and `flutter_animate` should add motion to the new content around it. Wrapping the entire `Hero` in a second entrance animation looks attractive in a demo, but the card can scale, fade, and fly at the same time. That reads as a broken route transition rather than one continuous object.

## The split that keeps the transition readable

I use this ownership table before writing the widget tree:

| Visual change | Owner | Why |
|---|---|---|
| Avatar or thumbnail travels between routes | `Hero` | It knows both route positions |
| Destination title and metadata appear | `flutter_animate` | These widgets exist only on the new route |
| Route push and pop timing | `PageRoute` | Navigation remains the source of truth |

The source and destination must use the same tag and a compatible shared child. Animate the non-shared destination content instead:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key, required this.user});

  final User user;

  @override
  Widget build(BuildContext context) {
    final heroTag = 'profile-avatar-${user.id}';

    return Scaffold(
      appBar: AppBar(title: const Text('Profile')),
      body: ListView(
        padding: const EdgeInsets.all(24),
        children: [
          Center(
            child: Hero(
              tag: heroTag,
              child: CircleAvatar(
                radius: 48,
                backgroundImage: NetworkImage(user.avatarUrl),
              ),
            ),
          ),
          const SizedBox(height: 20),
          Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(user.name, style: Theme.of(context).textTheme.headlineSmall),
              const SizedBox(height: 8),
              Text(user.bio),
            ],
          )
              .animate(key: ValueKey('profile-details-${user.id}'))
              .fadeIn(duration: 220.ms, delay: 100.ms)
              .slideY(begin: .04, end: 0, duration: 220.ms),
        ],
      ),
    );
  }
}
```

The `ValueKey` is not decoration. If the same route widget rebuilds because a provider or future completes, it gives the details subtree a deliberate identity. Do not put a changing timestamp or list index in that key; either can restart the entrance while the user is reading.

## The two traps I test before shipping

First, keep the Hero child geometry stable. A source avatar with a `CircleAvatar` and a destination child with a padded `Container` can produce a visible jump at the handoff. Match shape, clipping, and image behavior, then animate nearby text separately.

Second, handle pop as well as push. A destination-only fade is fine for entry, but reverse navigation can feel abrupt if the details remain fully visible while the shared avatar flies back. Keep the detail animation short and avoid business work in its lifecycle callbacks. Saving data or deciding navigation inside an animation callback makes a canceled route surprisingly fragile.

For a quick review, check these conditions:

- one stable Hero tag per logical item;
- no scale or opacity effect wrapped around the shared child;
- destination-only effects are short, usually 160–280 ms;
- rebuilding the route does not replay the whole screen;
- push, pop, back button, and a fast repeated tap all look intentional.

`Hero` provides continuity across routes. `flutter_animate` provides emphasis after the destination exists. Keeping those responsibilities separate removes the double-motion bug and makes the transition easier to tune.
