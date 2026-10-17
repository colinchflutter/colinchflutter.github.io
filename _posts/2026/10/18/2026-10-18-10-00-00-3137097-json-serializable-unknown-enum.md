---
layout: post
title: "json_serializable Unknown Enum Values - Keep Older Flutter Apps from Crashing on API Expansion"
description: "Handle new server enum values safely with json_serializable, choose between a sentinel and null, and test the migration without hiding malformed API data."
date: 2026-10-18
tags: [json_serializable, networking, migration, testing, Flutter]
comments: true
share: true
---

![json_serializable enum compatibility in a Flutter API model](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

If an API can add enum strings without synchronizing every installed Flutter build, put an explicit fallback on the generated model. `json_serializable` throws when an enum value is not known; `@JsonKey(unknownEnumValue: ...)` keeps decoding alive. Use a sentinel when the UI needs to render an “other” state, and use `JsonKey.nullForUndefinedEnumValue` only when absence is a valid business state. This fix does not make a breaking field type or invalid payload safe automatically.

## The failure appears after a harmless server release

Imagine an older app knows `pending` and `paid`, while the server adds `refunded`. The generated decoder has no matching enum entry, so a list response can fail during model construction instead of showing the unaffected orders. A `try/catch` around the HTTP call is usually too broad: it turns one bad field into a failed screen and hides which contract changed.

| API value | Default generated behavior | Safer model choice |
| --- | --- | --- |
| `pending`, `paid` | Decodes normally | Keep the typed enum |
| New value such as `refunded` | `ArgumentError` | Map to `OrderStatus.unknown` |
| Missing or explicit `null` | Depends on field nullability | Use a nullable field only if absence is meaningful |
| Wrong JSON type such as `42` | Still an invalid payload | Report or reject it; do not treat it as an unknown enum |

The key distinction is that an unknown string can be a forward-compatible API change, while the wrong JSON type is a contract error.

## Add a typed fallback, not a magic string

The fallback must be another value from the same enum. `json_serializable` documents `unknownEnumValue` for this case, and its nullable decoder supports the special null sentinel.

```dart
import 'package:json_annotation/json_annotation.dart';

part 'order.g.dart';

enum OrderStatus {
  @JsonValue('pending')
  pending,
  @JsonValue('paid')
  paid,
  @JsonValue('refunded')
  refunded,
  unknown,
}

@JsonSerializable()
class Order {
  const Order({required this.id, required this.status});

  factory Order.fromJson(Map<String, dynamic> json) =>
      _$OrderFromJson(json);

  final String id;

  @JsonKey(unknownEnumValue: OrderStatus.unknown)
  final OrderStatus status;

  Map<String, dynamic> toJson() => _$OrderToJson(this);
}
```

Run the generator after changing the annotation:

```bash
dart run build_runner build --delete-conflicting-outputs
```

`defaultValue` is not a replacement for `unknownEnumValue`. It handles a missing or null source value; it does not catch a present string that is absent from the enum map. That difference is easy to miss during a migration because both cases can look like “the API did not provide a usable status.”

## Choose null only when the domain can represent it

For an optional filter, a new server value may be safely ignored. Make that decision visible in the type instead of putting `null` into a non-nullable field:

```dart
part 'order_summary.g.dart';

@JsonSerializable()
class OrderSummary {
  const OrderSummary(this.status);

  factory OrderSummary.fromJson(Map<String, dynamic> json) =>
      _$OrderSummaryFromJson(json);

  @JsonKey(unknownEnumValue: JsonKey.nullForUndefinedEnumValue)
  final OrderStatus? status;
}
```

This is a poor fit for payment state, permissions, or a workflow where “unknown” must trigger a visible review. In those cases, keep a non-nullable `unknown` value and make UI code handle it deliberately. Do not serialize the sentinel back to the server as if it were a supported command unless the API contract defines that value.

## Lock the behavior with a decoding test

The test should cover a future value and a malformed type separately. That prevents a later refactor from replacing a useful compatibility boundary with a broad catch-all.

```dart
import 'package:test/test.dart';

void main() {
  test('maps a future status to unknown', () {
    final order = Order.fromJson({
      'id': 'A-17',
      'status': 'refunded_by_partner',
    });

    expect(order.status, OrderStatus.unknown);
  });

  test('does not hide a wrong JSON type', () {
    expect(
      () => Order.fromJson({'id': 'A-18', 'status': 42}),
      throwsA(isA<ArgumentError>()),
    );
  });
}
```

Before shipping, check these four points:

1. The server can add values without breaking older clients.
2. The fallback is visible in the enum and handled by every relevant UI state.
3. `defaultValue` is not being used to solve an unknown-string case.
4. Generated files and a fixture for the future value are included in CI.

The official `json_serializable` documentation covers enum value mapping, while `json_annotation` defines the runtime behavior of `$enumDecodeNullable`. Read those contracts when upgrading the generator because generated output and supported annotation versions can change.

Sources: [json_serializable enum documentation](https://pub.dev/packages/json_serializable#enums), [JsonKey API](https://github.com/google/json_serializable.dart/blob/master/json_annotation/lib/src/json_key.dart), and [$enumDecodeNullable API](https://pub.dev/documentation/json_annotation/latest/json_annotation/%24enumDecodeNullable.html).
