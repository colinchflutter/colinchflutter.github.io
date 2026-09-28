---
layout: post
title: "Freezed v2 to v3 Migration - Fix Generated Code and Pattern Matching Breaks"
description: "A practical Freezed v2 to v3 migration guide for Flutter: update class modifiers, replace when/map calls, and recover from build_runner JSON errors."
date: 2026-09-29
tags: [freezed, migration, code_generation, json_serializable, testing, Flutter]
comments: true
share: true
---

![Flutter Freezed migration from generated pattern matching to Dart switch expressions](/assets/images/shared-preferences-async-migration-cache-paths.png)

If a Flutter project is moving from Freezed 2 to 3, migrate the model declarations and every `.when`/`.map` call together, then regenerate code from a clean dependency resolution. This is a good choice for projects already on Dart 3 and using sealed unions. It is not a good reason to upgrade a stable release branch by itself: the migration changes source syntax, generated APIs, and sometimes the `freezed`/`json_serializable` builder combination at the same time.

The Freezed migration guide documents two breaking areas: factory-based classes need `abstract` or `sealed`, and Freezed no longer generates the old pattern-matching helpers such as `.when` and `.map`. A separate real-world failure reports `Cannot populate the required constructor argument` when `@JsonSerializable()` is combined with a Freezed 3 model. Treat those as separate checks instead of trying random `build_runner` flags.

## What changes between v2 and v3

| Area | Freezed 2 style | Freezed 3 action | Risk if skipped |
| --- | --- | --- | --- |
| Model declaration | `class Person with _$Person` | Use `abstract class` or `sealed class` | Analyzer or generator errors |
| Union dispatch | `model.when(...)` / `model.map(...)` | Use Dart pattern `switch` | Compile errors across UI and tests |
| JSON generation | `fromJson` plus generated parts | Keep both parts and regenerate | Missing `_$ModelFromJson` or stale output |
| Builder customization | Class annotations mixed with defaults | Keep configuration in one place | Constructor population errors |

The first distinction matters. Use `abstract` for an ordinary Freezed model whose generated implementation is the only concrete type. Use `sealed` for a union where every case should be known to Dart's exhaustiveness checks.

The v2 declaration below is a typical migration failure:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'payment_state.freezed.dart';

@freezed
class PaymentState with _$PaymentState {
  const factory PaymentState.idle() = PaymentIdle;
  const factory PaymentState.processing(String paymentId) = PaymentProcessing;
  const factory PaymentState.failed(String message) = PaymentFailed;
}
```

For a union, add `sealed`:

```dart
@freezed
sealed class PaymentState with _$PaymentState {
  const factory PaymentState.idle() = PaymentIdle;
  const factory PaymentState.processing(String paymentId) = PaymentProcessing;
  const factory PaymentState.failed(String message) = PaymentFailed;
}
```

For a non-union model, `abstract class` is the safer equivalent:

```dart
@freezed
abstract class Payment with _$Payment {
  const factory Payment({
    required String id,
    required int amount,
  }) = _Payment;
}
```

## Replace generated `.when` and `.map` calls

The old call is not fixed by regenerating files. The source call itself must become a Dart pattern switch:

```dart
final label = state.when(
  idle: () => 'Ready',
  processing: (id) => 'Charging $id',
  failed: (message) => 'Failed: $message',
);
```

The v3 form keeps the same behavior while making the union cases visible to the Dart analyzer:

```dart
final label = switch (state) {
  PaymentIdle() => 'Ready',
  PaymentProcessing(:final paymentId) => 'Charging $paymentId',
  PaymentFailed(:final message) => 'Failed: $message',
};
```

Search both production and test code. A focused migration search is less noisy than scanning generated files:

```bash
rg -n "\.when\(|\.maybeWhen\(|\.map\(|\.maybeMap\(" lib test
rg -n "class .* with _\$|@freezed" lib test
```

Do not replace every `.map` in the repository. Collection `map` calls are unrelated. The receiver must be a Freezed union or the analyzer error must identify a generated Freezed extension.

## Keep JSON generation explicit

A serializable model needs both generated parts and a `fromJson` factory. Freezed's documented minimal shape is:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'payment.freezed.dart';
part 'payment.g.dart';

@freezed
abstract class Payment with _$Payment {
  const factory Payment({
    required String id,
    required int amount,
  }) = _Payment;

  factory Payment.fromJson(Map<String, Object?> json) =>
      _$PaymentFromJson(json);
}
```

The common mistake is to add `@JsonSerializable()` automatically because the model has JSON. Freezed already coordinates the normal `fromJson`/`toJson` generation through the factory and parts. Keep the class annotation only when a specific `json_serializable` option is required, such as a project that has intentionally configured nested serialization. If a Freezed 3 build reports:

```text
Cannot populate the required constructor argument: name.
```

reduce the model to the documented shape above, remove the class-level annotation, and run the generator again. A maintainer issue reproduced this failure with Freezed 3, `@JsonSerializable()`, and a required constructor argument; it is not evidence that the JSON field should become nullable.

For nested Freezed objects, choose one configuration source. Either use the supported annotation for the package versions resolved by your project, or configure the builder in `build.yaml`; do not copy several snippets from different major versions into the same model. The important constraint is that the generated `.freezed.dart` and `.g.dart` files come from the same build.

## A reproducible migration sequence

Run the migration in a branch and record the resolved versions before changing source files:

```bash
flutter pub deps --style=compact > /tmp/freezed-deps-before.txt
flutter pub add freezed_annotation
flutter pub add dev:freezed dev:build_runner
flutter pub add json_annotation
flutter pub add dev:json_serializable
```

Then apply the source changes, regenerate, and validate the generated output without editing it manually:

```bash
dart run build_runner build --delete-conflicting-outputs
dart format lib test
flutter analyze
flutter test
```

The command deletes conflicting generated outputs, not your hand-written model files. It is useful when a branch still contains artifacts from the old builder, but it does not repair an invalid declaration or an obsolete `.when` call.

Use this decision table when the build fails:

| Symptom | Likely cause | Action |
| --- | --- | --- |
| `class ... with _$...` generator error | Missing modifier | Add `abstract` or `sealed` |
| `.when` or `.map` is undefined | Removed v2 helper | Convert that union dispatch to `switch` |
| `Target ...freezed.dart hasn't been generated` | Generator not run or `part` typo | Check the exact filename, then run build_runner |
| Required constructor argument cannot be populated | Incompatible JSON annotation/configuration | Start from minimal Freezed JSON shape, then re-add one option |
| Generated file still contains old APIs | Stale artifact or wrong package resolution | Delete conflicting outputs and inspect `pub deps` |

## Migration checklist for a release branch

1. Confirm the Dart SDK satisfies the package constraints before changing the lockfile.
2. Add `abstract` or `sealed` to every factory-based Freezed declaration.
3. Convert union `.when`, `.maybeWhen`, `.map`, and `.maybeMap` calls in `lib` and `test`.
4. Verify every JSON model has matching `.freezed.dart` and `.g.dart` parts.
5. Remove copied `@JsonSerializable()` annotations from models that need only standard JSON support.
6. Run `build_runner`, `flutter analyze`, and focused model tests from the same dependency resolution.
7. Review the generated diff. A generated file changing far beyond the migrated models is a signal to inspect dependency or builder changes.

The safe stopping point is not “the generator finished.” The migration is complete when generated code is reproducible, union branches are exhaustively handled, and a JSON round-trip test still passes for each public model. If the application is not ready for Dart pattern matching, pin the existing Freezed major version temporarily and schedule the source migration as a deliberate change; a forced package upgrade only moves the breakage into the next build.

References: [Freezed v2-to-v3 migration guide](https://github.com/rrousselGit/freezed/blob/master/packages/freezed/migration_guide.md), [Freezed package documentation](https://pub.dev/packages/freezed), [Freezed issue #1218 on `JsonSerializable` generation](https://github.com/rrousselGit/freezed/issues/1218), and [build_runner documentation](https://pub.dev/packages/build_runner).
