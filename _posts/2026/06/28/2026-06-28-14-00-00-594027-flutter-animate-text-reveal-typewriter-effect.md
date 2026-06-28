---
layout: post
title: "flutter_animate Text Reveal — Typewriter, Character Stagger, and Word-by-Word Animations"
description: "Build typewriter and character-by-character text reveal animations in Flutter using flutter_animate, with working code for staggered word reveal and reset-on-trigger patterns."
date: 2026-06-28
tags: [flutter_animate, animation, Flutter, text, state_management]
comments: true
share: true
---

# flutter_animate Text Reveal — Typewriter, Character Stagger, and Word-by-Word Animations

![Typewriter text animation on a dark screen](https://images.unsplash.com/photo-1455390582262-044cdead277a?w=800&q=80)

Text animations are one of those things that look effortless in demos but turn into a mess once you try to build them yourself. The "typewriter" effect specifically — where characters appear one by one — usually pulls you toward a `Timer`, a `StatefulWidget`, and a string slice that grows character by character. With flutter_animate, you can skip most of that plumbing and build the same thing with pure composition.

## The naive approach (and why it's annoying)

Most tutorials show you the `Timer.periodic` + `setState` approach:

```dart
class TypewriterText extends StatefulWidget { ... }

class _TypewriterTextState extends State<TypewriterText> {
  String _displayed = '';
  int _index = 0;
  Timer? _timer;

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(const Duration(milliseconds: 60), (_) {
      if (_index < widget.text.length) {
        setState(() => _displayed = widget.text.substring(0, ++_index));
      } else {
        _timer?.cancel();
      }
    });
  }

  // ...dispose, build, etc.
}
```

Works, but you're managing a Timer lifecycle, a dispose call, and setState on every character tick. If the text changes mid-animation, you have to cancel the old timer and restart — that's another few lines. 

## Building it with flutter_animate

The trick is to animate individual characters. Split the text, map to a list of `Text` widgets, then stagger each one with `.animate()`.

```dart
class TypewriterAnimate extends StatelessWidget {
  final String text;
  final Duration charDelay;

  const TypewriterAnimate({
    super.key,
    required this.text,
    this.charDelay = const Duration(milliseconds: 50),
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: text.characters.toList().asMap().entries.map((entry) {
        final index = entry.key;
        final char = entry.value;
        return Text(
          char,
          style: const TextStyle(fontSize: 24, fontWeight: FontWeight.w500),
        )
            .animate(delay: charDelay * index)
            .fadeIn(duration: 1.ms)
            .then()
            .slideY(begin: 0.3, end: 0.0, duration: 80.ms, curve: Curves.easeOut);
      }).toList(),
    );
  }
}
```

The `delay: charDelay * index` is the key part — each character waits proportionally longer before starting, which creates the typewriter feel. The `.then()` + `.slideY` adds a small upward drift so it's not just a flat fade, which looks more polished.

One thing to watch: `text.characters` handles multi-byte characters (emoji, non-ASCII) correctly. `.split('')` does not. Use `.characters` from the `characters` package (bundled with Flutter).

## Word-by-word reveal

Sometimes character-by-character is too slow, especially for longer sentences. Word reveal is faster and still animated:

```dart
class WordReveal extends StatelessWidget {
  final String sentence;

  const WordReveal({super.key, required this.sentence});

  @override
  Widget build(BuildContext context) {
    final words = sentence.split(' ');
    return Wrap(
      spacing: 6,
      children: words.asMap().entries.map((entry) {
        final i = entry.key;
        final word = entry.value;
        return Text(word, style: const TextStyle(fontSize: 18))
            .animate(delay: 120.ms * i)
            .fadeIn(duration: 200.ms)
            .slideX(begin: -0.2, end: 0.0, duration: 200.ms, curve: Curves.easeOut);
      }).toList(),
    );
  }
}
```

`Wrap` with `spacing` handles line breaks automatically. Each word slides in from the left with a short delay — reads naturally and doesn't feel mechanical.

## Replay on trigger

The static version above plays once when the widget mounts. If you need to replay it — say, after a button tap or when a new message arrives — you need a `key` reset:

```dart
class ReplayDemo extends StatefulWidget {
  @override
  State<ReplayDemo> createState() => _ReplayDemoState();
}

class _ReplayDemoState extends State<ReplayDemo> {
  String _message = "Hello, Flutter!";
  int _replayKey = 0;

  void _replay(String newMessage) {
    setState(() {
      _message = newMessage;
      _replayKey++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TypewriterAnimate(
          key: ValueKey(_replayKey),
          text: _message,
        ),
        const SizedBox(height: 24),
        ElevatedButton(
          onPressed: () => _replay("Animation replayed!"),
          child: const Text('Replay'),
        ),
      ],
    );
  }
}
```

Changing `key` forces Flutter to destroy and recreate the widget, which restarts the animations from scratch. `ValueKey(_replayKey)` is the simplest way to do it — no animation controller management needed.

![Staggered word reveal animation demo](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

## Combining with other effects

Since each character is just a widget with `.animate()`, you can stack effects freely. Here's a "highlight reveal" for a heading:

```dart
Text('New Feature')
    .animate()
    .fadeIn(delay: 200.ms, duration: 400.ms)
    .then(delay: 100.ms)
    .shimmer(
      duration: 800.ms,
      color: const Color(0xFFFFD700).withOpacity(0.6),
    )
```

This fades in first, then sweeps a gold shimmer — looks good on feature announcement screens. The stagger approach works for any list of widgets, not just characters. I used the same pattern for an onboarding checklist where each bullet point slides in one after another.

## The edge case that tripped me up

Spaces. When you do `text.characters.toList()`, a space character becomes an empty-width `Text(' ')` widget inside a `Row`. On most fonts that's fine, but with custom fonts or tight letter-spacing, the space can collapse to zero. Fix it by replacing spaces with a `SizedBox`:

```dart
final widget = char == ' '
    ? const SizedBox(width: 7)
    : Text(char, style: textStyle)
        .animate(delay: charDelay * index)
        .fadeIn(duration: 1.ms);
```

Width `7` is roughly right for 24sp body text — adjust per your font.

---

Next up: following a path with `flutter_animate`'s `MoveEffect` and custom `Curve` to animate widgets along bezier curves.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter characters package](https://pub.dev/packages/characters)
- [Flutter animations overview](https://docs.flutter.dev/ui/animations)
