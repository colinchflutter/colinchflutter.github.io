---
layout: post
title: "go_router Query Parameters - Preserve Flutter Search State in the URL"
description: "Learn how to use go_router query parameters for shareable Flutter search and filter URLs, including safe updates and state restoration."
date: 2026-08-17
tags: [go_router, navigation, Web, testing]
comments: true
share: true
---

![Flutter catalog search screen with query and filter values preserved in a browser URL](https://images.unsplash.com/photo-1558655146-d09347e92766?w=1200&q=80)

When a Flutter screen has search, sorting, or filtering, keeping that state only in a widget makes refreshes and deep links frustrating. `go_router` query parameters provide a small but reliable contract: the URL describes the current catalog state, and the page reconstructs itself from that URL.

## Why query parameters fit search state

Path parameters identify a resource, such as `/products/42`. Query parameters describe a view of a resource, such as `/products?q=keyboard&sort=price`. That distinction matters when a user copies a link, refreshes Flutter Web, or presses the browser back button.

I initially stored the selected filter in a `StatefulWidget` and updated the list locally. It looked fine on mobile, but a browser refresh silently reset the filter. Putting every temporary value into the path was worse because `/catalog/keyboard/price` is difficult to extend and does not read like a view URL.

| State | Recommended URL location | Example |
| --- | --- | --- |
| Product identity | Path parameter | `/products/42` |
| Search and sorting | Query parameter | `/catalog?q=keyboard&sort=price` |
| Temporary animation or dialog | Widget state | Not shareable |
| Large, sensitive, or non-serializable data | App state/service | Do not put it in the URL |

The key rule is simple: if another person should be able to open the same view, serialize it into the URL.

## Define a route that reads query parameters

The route should read from `state.uri.queryParameters` inside the builder. The values are always strings, so parsing and defaults belong at this boundary rather than being scattered through the page.

```dart
final router = GoRouter(
  initialLocation: '/catalog',
  routes: [
    GoRoute(
      path: '/catalog',
      builder: (context, state) {
        final query = state.uri.queryParameters['q'] ?? '';
        final sort = state.uri.queryParameters['sort'] ?? 'relevance';

        return CatalogPage(
          initialQuery: query,
          initialSort: _allowedSorts.contains(sort) ? sort : 'relevance',
        );
      },
    ),
  ],
);

const _allowedSorts = {'relevance', 'price', 'rating'};
```

`state.uri` is useful here because it exposes the complete parsed location. It also avoids manually splitting `state.location`, which is error-prone when values contain spaces, ampersands, or non-ASCII characters.

## Update the URL when the user changes a filter

The page can update the current route with `context.go`. Build a `Uri` instead of concatenating strings. `Uri` takes care of encoding a search such as `mechanical keyboard` correctly.

```dart
class CatalogPage extends StatefulWidget {
  const CatalogPage({
    required this.initialQuery,
    required this.initialSort,
    super.key,
  });

  final String initialQuery;
  final String initialSort;

  @override
  State<CatalogPage> createState() => _CatalogPageState();
}

class _CatalogPageState extends State<CatalogPage> {
  late final TextEditingController _searchController;
  late String _sort;

  @override
  void initState() {
    super.initState();
    _searchController = TextEditingController(text: widget.initialQuery);
    _sort = widget.initialSort;
  }

  void _updateUrl({String? query, String? sort}) {
    final nextQuery = query ?? _searchController.text.trim();
    final nextSort = sort ?? _sort;
    final parameters = <String, String>{
      if (nextQuery.isNotEmpty) 'q': nextQuery,
      if (nextSort != 'relevance') 'sort': nextSort,
    };

    context.go(Uri(path: '/catalog', queryParameters: parameters).toString());
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: TextField(
          controller: _searchController,
          textInputAction: TextInputAction.search,
          onSubmitted: (value) => _updateUrl(query: value),
          decoration: const InputDecoration(hintText: 'Search products'),
        ),
      ),
      body: DropdownButton<String>(
        value: _sort,
        items: const [
          DropdownMenuItem(value: 'relevance', child: Text('Relevance')),
          DropdownMenuItem(value: 'price', child: Text('Lowest price')),
          DropdownMenuItem(value: 'rating', child: Text('Top rated')),
        ],
        onChanged: (value) {
          if (value == null) return;
          setState(() => _sort = value);
          _updateUrl(sort: value);
        },
      ),
    );
  }

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }
}
```

The example only commits the search term when the user submits it. Updating the route on every keystroke creates a browser history entry for every character and can trigger unnecessary data requests. For live search, debounce the input and use `context.replace` if the intermediate states should not be added to back-button history.

## Handle browser back and external deep links

There are two directions of synchronization:

```text
user action ──> URL query parameters ──> route builder ──> catalog request
browser back ──> URL changes ──────────> page receives new values
```

The second direction is easy to miss. A page can be rebuilt with a new `state.uri` after browser navigation, while its existing `TextEditingController` still contains the old text. If the page is kept alive by a shell route, implement `didUpdateWidget` and synchronize local controls.

```dart
@override
void didUpdateWidget(covariant CatalogPage oldWidget) {
  super.didUpdateWidget(oldWidget);

  if (oldWidget.initialQuery != widget.initialQuery &&
      _searchController.text != widget.initialQuery) {
    _searchController.value = TextEditingValue(
      text: widget.initialQuery,
      selection: TextSelection.collapsed(offset: widget.initialQuery.length),
    );
  }

  if (oldWidget.initialSort != widget.initialSort) {
    _sort = widget.initialSort;
  }
}
```

Without this synchronization, the URL may say `sort=rating` while the dropdown still displays `relevance`. That mismatch is especially confusing when testing deep links because the network request and visible controls disagree.

## Common mistakes with go_router query parameters

1. **Using `extra` for shareable state.** `extra` is convenient for passing Dart objects, but it is not a portable URL contract. A copied link cannot contain the original object.
2. **Writing empty defaults into every URL.** `/catalog?q=&sort=relevance` works, but `/catalog` is cleaner. Omit default values and apply them when reading.
3. **Accepting arbitrary enum strings.** Query parameters can be edited by anyone. Validate `sort`, page numbers, and IDs before using them in a request.
4. **Concatenating query strings manually.** A value containing `&` or `?` can change the meaning of the URL. Build a `Uri` instead.
5. **Calling `context.go` during `build`.** Route updates during build can cause loops. Update the URL from user actions, a debounced listener, or a controlled redirect.

## A short verification checklist

| Test | Expected result |
| --- | --- |
| Open `/catalog?q=keyboard&sort=price` directly | Search field and sort control match the URL |
| Search for `mechanical keyboard` | Spaces are encoded and decoded correctly |
| Press browser back after changing sort | Both URL and dropdown return to the previous value |
| Use an unknown sort value | The page safely falls back to `relevance` |
| Refresh Flutter Web | The same catalog view is reconstructed |

Query parameters are not a replacement for application state. They are the right boundary for small, serializable view state that users expect to bookmark, share, and restore. With `state.uri`, `Uri`, validation, and two-way control synchronization, `go_router` can keep Flutter navigation and catalog behavior aligned across mobile deep links and browser navigation.
