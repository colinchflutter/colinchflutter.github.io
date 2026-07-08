---
layout: post
title: "flutter_animate SearchBar Query Motion - search feedback without input jitter"
description: "Use flutter_animate with Flutter SearchBar to animate query feedback, result counts, and empty states without shaking the input field."
date: 2026-07-09
tags: [flutter_animate, animation, Flutter, state_management, Web]
comments: true
share: true
---

![Flutter SearchBar query feedback with flutter_animate](https://images.unsplash.com/photo-1498050108023-c5249f4df085?w=800&q=80)

This image points to the useful part of search motion: the interface around the query should respond, while the text field stays steady.

`flutter_animate` fits `SearchBar` when it explains query state outside the typing surface. Animate the result count, loading strip, suggestion group, or empty state. Keep the field, caret, and typed text visually stable. Search already demands attention at the character level, so even a 4 px slide on the input can feel like lag.

The mistake I made on one admin screen was wrapping the whole search header in a fade and scale chain. It looked polished with three test rows. With 180 rows and a debounced backend request, every pause in typing replayed the header animation. The user was trying to check spelling, but the search box kept calling attention to itself.

| Search area | Animate | Keep still |
| --- | --- | --- |
| `SearchBar` | optional trailing clear icon opacity | field width, text, caret |
| Result header | count change, loading state | title baseline |
| Suggestions | inserted group, empty state | list scroll position |
| Network status | thin progress indicator | search focus |

Here is a small pattern I use for local or debounced search. The query owns the state, while `flutter_animate` only decorates the feedback below the bar.

```dart
class AnimatedSearchPanel extends StatefulWidget {
  const AnimatedSearchPanel({super.key, required this.items});

  final List<String> items;

  @override
  State<AnimatedSearchPanel> createState() => _AnimatedSearchPanelState();
}

class _AnimatedSearchPanelState extends State<AnimatedSearchPanel> {
  final _controller = SearchController();
  String _query = '';

  List<String> get _matches {
    final text = _query.trim().toLowerCase();
    if (text.isEmpty) return widget.items;
    return widget.items
        .where((item) => item.toLowerCase().contains(text))
        .toList(growable: false);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final matches = _matches;
    final hasQuery = _query.trim().isNotEmpty;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        SearchBar(
          controller: _controller,
          hintText: 'Search packages',
          leading: const Icon(Icons.search),
          trailing: [
            IconButton(
              tooltip: 'Clear',
              icon: const Icon(Icons.close),
              onPressed: hasQuery
                  ? () {
                      _controller.clear();
                      setState(() => _query = '');
                    }
                  : null,
            ).animate(target: hasQuery ? 1 : 0).fade(duration: 120.ms),
          ],
          onChanged: (value) => setState(() => _query = value),
        ),
        const SizedBox(height: 12),
        _SearchStatus(query: _query, count: matches.length),
        const SizedBox(height: 8),
        for (final item in matches.take(5))
          ListTile(
            dense: true,
            title: Text(item),
          ),
      ],
    );
  }
}

class _SearchStatus extends StatelessWidget {
  const _SearchStatus({required this.query, required this.count});

  final String query;
  final int count;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final empty = query.trim().isNotEmpty && count == 0;

    return AnimatedSwitcher(
      duration: 160.ms,
      child: Text(
        empty ? 'No matches for "$query"' : '$count matching packages',
        key: ValueKey('$query-$count'),
        style: theme.textTheme.labelLarge,
      )
          .animate()
          .fadeIn(duration: 120.ms)
          .slideY(begin: 0.18, end: 0, duration: 120.ms),
    );
  }
}
```

The useful boundary is simple: `SearchBar` captures input, `_SearchStatus` shows movement. That separation prevents two common bugs. One is focus loss when a keyed parent rebuilds the search field. The other is accidental scroll jump when a suggestion list is replaced during every keystroke.

For remote search, I keep the same shape and add a tiny loading row under the bar. The row can fade in after 150 ms so fast local hits do not flash a spinner. If the API usually returns in 80 ms, showing animated loading on every character is noise. If it returns in 500 ms, the feedback helps because the user can tell the query was accepted.

```dart
Widget searchLoadingLine(bool loading) {
  return SizedBox(
    height: 3,
    child: LinearProgressIndicator(
      minHeight: 3,
      backgroundColor: Colors.transparent,
    )
        .animate(target: loading ? 1 : 0)
        .fade(duration: 140.ms)
        .scaleX(
          begin: 0.92,
          end: 1,
          alignment: Alignment.centerLeft,
          duration: 180.ms,
        ),
  );
}
```

The checks I use are concrete. Type ten characters quickly and the caret should never move sideways. Clear the query and focus should remain in the field. Filter from 50 rows to zero rows and only the status or empty state should animate. If the list, app bar, and input all move together, the animation is explaining the rebuild instead of explaining the search result.

Short version: animate the answer, not the question. `flutter_animate` makes `SearchBar` feel better when query feedback changes smoothly and the typing surface behaves like plain Flutter.
