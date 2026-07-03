---
layout: post
title: "flutter_animate DataTable Motion - readable row updates without fake loading"
description: "Animate Flutter DataTable row changes with flutter_animate while keeping sorting, selection, and dense desktop layouts readable."
date: 2026-07-04
tags: [flutter_animate, animation, performance, Desktop]
comments: true
share: true
---

![Flutter desktop table with highlighted changing rows](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)
This image is useful because table motion should guide the eye to changed rows, not decorate every cell.

`flutter_animate` works best in a `DataTable` when it marks the rows that actually changed. Animate the inserted row, the refreshed status chip, or the selected row background. Do not replay every row after sorting, filtering, or paging, because a dense table becomes harder to scan exactly when the user needs stability.

The first version I tried wrapped the whole `DataTable` with `.fadeIn().slideY()`. It looked clean on the initial screen load. Then I sorted by amount and all 40 rows faded again, which made the table feel like it had fetched new data even though only the order changed. That is the wrong signal.

## Animate the row boundary

Keep the table itself boring. Put motion inside the cell that owns the state change, or around a row child with a stable key. For an order table, I usually animate only rows whose `updatedAt` value changed in the last few seconds.

```dart
class OrdersTable extends StatelessWidget {
  const OrdersTable({
    super.key,
    required this.orders,
    required this.recentlyChangedIds,
  });

  final List<OrderRow> orders;
  final Set<String> recentlyChangedIds;

  @override
  Widget build(BuildContext context) {
    return DataTable(
      sortColumnIndex: 2,
      columns: const [
        DataColumn(label: Text('Order')),
        DataColumn(label: Text('Status')),
        DataColumn(label: Text('Total')),
      ],
      rows: [
        for (final order in orders)
          DataRow(
            key: ValueKey(order.id),
            selected: order.isSelected,
            cells: [
              DataCell(Text(order.number)),
              DataCell(_StatusPill(
                label: order.status,
                changed: recentlyChangedIds.contains(order.id),
              )),
              DataCell(Text('\$${order.total.toStringAsFixed(2)}')),
            ],
          ),
      ],
    );
  }
}
```

The animated cell stays small, so sorting and horizontal scrolling remain predictable.

```dart
class _StatusPill extends StatelessWidget {
  const _StatusPill({
    required this.label,
    required this.changed,
  });

  final String label;
  final bool changed;

  @override
  Widget build(BuildContext context) {
    final pill = DecoratedBox(
      decoration: BoxDecoration(
        color: changed ? Colors.green.shade50 : Colors.grey.shade100,
        borderRadius: BorderRadius.circular(999),
      ),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
        child: Text(label),
      ),
    );

    if (!changed) return pill;

    return pill
        .animate(key: ValueKey('status-$label'))
        .fadeIn(duration: 140.ms)
        .scale(begin: const Offset(.96, .96), duration: 180.ms)
        .shimmer(duration: 420.ms, color: Colors.green.shade100);
  }
}
```

## Pick motion by table event

| Table event | Better animation | Avoid |
| --- | --- | --- |
| New row appended | short fade and 4px vertical settle | full table slide |
| Status changed | pill shimmer or color pulse | bouncing the whole row |
| Sort changed | no entrance animation | replaying all visible rows |
| Selection changed | background or checkbox feedback | delaying row selection |

That distinction matters on desktop. A `DataTable` often sits inside a split pane with filters, pagination, and detail panels. If each small change restarts a 500 ms animation, keyboard users and screen readers get a UI that feels slower than the data behind it.

## Keep keys and timing strict

Use stable row keys based on domain IDs, not list indexes. Index keys break as soon as sorting changes. The row at index `4` may now represent a different order, so Flutter can preserve the wrong element and `flutter_animate` may play on a row that did not change.

I also keep table motion below 250 ms unless it is a one-time empty-state transition. In a grid of financial records, support tickets, or admin users, the motion is a pointer. It should answer, "what just changed?" If the animation becomes the event, it is already too loud.

## Quick checklist

- Animate changed cells or newly inserted rows, not the whole `DataTable`.
- Use `ValueKey(order.id)` for rows and a separate key for state-specific animated children.
- Do not replay row entrances for sorting, filtering, pagination, or horizontal scroll.
- Prefer fade, subtle scale, shimmer, or color pulse over large slide distances.
- Keep dense table animations short enough that repeated edits still feel instant.
