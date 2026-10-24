---
layout: post
title: "Hive to hive_ce Migration - Keep Flutter Box Data Readable and Rollback-Safe"
description: "A production migration plan from Hive v2 to hive_ce for Flutter, covering box compatibility, adapter IDs, generated schema files, and rollback limits."
date: 2026-10-25
tags: [hive_ce, migration, caching, persistence, testing, Flutter]
comments: true
share: true
---
![Flutter Hive to hive_ce migration with box compatibility and rollback checkpoints](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

The safest `hive` to `hive_ce` migration is a storage-compatibility exercise, not a pubspec rename. It suits an app that already has user data in Hive v2 boxes and needs a maintained package without forcing a logout or data reset. It does not suit a new app with no persisted data, or a release that cannot keep a rollback build compatible with the bytes written by the new build.

## Why the package swap can still break existing installs

`hive_ce` describes itself as a continuation of Hive v2, and its API is intentionally familiar. That makes the first build look deceptively easy: change imports, regenerate adapters, and ship. The production risk starts after the first successful write.

There are three separate compatibility questions:

| Question | Safe assumption | What must be checked |
| --- | --- | --- |
| Can the new binary open an old box? | Often, but do not infer it from a clean install | A copied box from the oldest supported release |
| Can old model data be decoded? | Only if type IDs and field indexes stay stable | Every registered adapter and generated schema |
| Can the old binary open data written by the new binary? | Not always | Type IDs and features used after the rollout |

The last row is the one that turns a normal upgrade into a rollback incident. Hive CE expands the supported type-ID range beyond the original Hive v2 range. That is useful for new models, but an older rollback build may not understand a type ID that the new build has already written. Treat that as a one-way boundary unless you have tested both directions with real fixtures.

## Freeze the data contract before changing imports

Before editing `pubspec.yaml`, record the storage contract that is already in production. This is a small text file or test fixture, not a screenshot of the model classes.

```text
box: user_cache
encryption: enabled
models: UserProfile, DraftMessage
typeIds: UserProfile=10, DraftMessage=11
fieldIndexes:
  UserProfile.email=0
  UserProfile.displayName=1
  DraftMessage.body=0
oldest_supported_release: 4.7.0
rollback_release: 4.9.2
```

Keep the type IDs and field indexes. A Dart field name is not the storage contract by itself. If a generated adapter assigns a new field index after a rename, the app may open the box successfully while reading the wrong value, which is harder to detect than a startup exception.

For generated adapters, check in the generated Hive schema. The official Hive CE documentation says the `.g.yaml` schema is used to incrementally update adapters, must be committed, and may need a manual edit when a class or field is renamed. Do not delete it during a routine migration.

## Use a staged dependency change

The package names are different, so update the runtime package, Flutter helper, and generator as a set. Keep your existing `build_runner` line unless dependency resolution tells you otherwise.

```yaml
dependencies:
  hive_ce: ^2.22.0
  hive_ce_flutter: ^2.5.0

dev_dependencies:
  build_runner: ^2.4.0
  hive_ce_generator: ^1.11.0
```

The versions above are examples of the package lines checked on 2026-10-25, not a recommendation to blindly copy them. Resolve them against the Flutter and Dart SDK used by your project. In particular, do not assume that `hive_ce_generator` shares the same major version as `hive_ce`.

Then change imports and initialization together:

```dart
import 'package:hive_ce/hive_ce.dart';
import 'package:hive_ce_flutter/hive_ce_flutter.dart';
import 'hive/hive_registrar.g.dart';

Future<void> initializeStorage() async {
  await Hive.initFlutter();
  Hive.registerAdapters();
  await Hive.openBox<UserProfile>('user_cache');
}
```

Do not leave `package:hive/` or `package:hive_flutter/` imports in a partially migrated library. A mixed dependency graph can leave two Hive implementations in the same application, with separate registries and separate Dart types. Check it before merging:

```bash
flutter pub deps | rg '(^| )(hive|hive_flutter|hive_ce|hive_ce_flutter|hive_generator|hive_ce_generator) '
rg "package:hive(_flutter)?/" lib test
```

If another package still requires the original Hive namespace, pause the migration. Replacing your direct dependency does not make that transitive package use Hive CE.

## Generate adapters without losing history

Hive CE supports generated adapters and stores the evolving field mapping in a YAML schema. A minimal generated model setup looks like this:

```dart
import 'package:hive_ce/hive_ce.dart';

part 'hive_adapters.g.dart';

@GenerateAdapters([AdapterSpec<UserProfile>()])
class UserProfile {
  final String email;
  final String displayName;

  const UserProfile({
    required this.email,
    required this.displayName,
  });
}
```

Run generation from the package root and review all generated files:

```bash
dart run build_runner build --delete-conflicting-outputs
git diff -- lib/hive
```

Adding a field is usually the least dangerous change when the new field has a constructor default. Renaming a field needs a schema edit so the old field index remains attached to the renamed field. Changing a field's type is not a rename; create a new field and convert the old value explicitly. The Hive CE adapter documentation lists type changes as unsupported for generated adapter evolution.

This is the point where a clean install test is useless. A clean install has no old field indexes to exercise.

## Make the rollback boundary explicit

Use a copied production box for a compatibility test. Do not use a user's live storage directory and do not overwrite the only backup.

```dart
Future<void> openMigrationFixture(Directory fixtureDirectory) async {
  Hive.init(fixtureDirectory.path);
  Hive.registerAdapters();

  final box = await Hive.openBox<UserProfile>('user_cache');
  final profile = box.get('current');

  if (profile == null || profile.email.isEmpty) {
    throw StateError('Migration fixture was not decoded correctly');
  }
}
```

The minimum matrix is:

| Fixture | New hive_ce build | Old rollback build | Expected result |
| --- | --- | --- | --- |
| Oldest supported box | Open and read | Already known good | No data loss |
| Box after adding a field | Read with default | Read while ignoring new field | Only if rollback is required |
| Box after a renamed field | Read old value under new name | Read old value | Requires schema preservation |
| Box after a new high type ID | Read | May fail | Block rollback or keep IDs in old range |

The final row is a release decision, not a coding detail. If rollback must remain possible, do not write data using a capability the rollback binary cannot decode. If the migration is intentionally one-way, document that the next release must repair or reinstall rather than silently rolling back.

## A release checklist that catches the expensive failures

- [ ] Export a copy of a real box from the oldest supported app version.
- [ ] Record encryption settings, type IDs, field indexes, and the rollback version.
- [ ] Update `hive_ce`, `hive_ce_flutter`, and the generator as a compatible set.
- [ ] Confirm no library still imports or pulls the original Hive packages unexpectedly.
- [ ] Commit the generated `.g.yaml` schema and adapter output.
- [ ] Test added fields, renamed fields, removed fields, and a deliberately incompatible type change.
- [ ] Open the fixture with the new build before writing anything to it.
- [ ] Decide whether the old build can read data written by the new build.
- [ ] Add a startup recovery path for an unreadable box instead of deleting it automatically.

Do not “fix” an adapter collision by renumbering an existing model. The number is part of the stored data contract. Choose a new ID only for a genuinely new type, and preserve the old adapter long enough to decode existing records.

## Short version

`hive_ce` can reduce maintenance pressure for a Hive v2 Flutter app, but the migration is safe only when old box files and generated schema history are treated as release artifacts. Keep IDs and field indexes stable, test a copied oldest-version box, inspect the dependency tree for a second Hive implementation, and decide the rollback boundary before the new build writes data. A green clean-install launch proves almost none of that.

The package facts in this guide were checked against the [hive_ce package page](https://pub.dev/packages/hive_ce), the [Hive CE adapter documentation](https://github.com/IO-Design-Team/hive_ce_docs/blob/master/custom-objects/generate_adapters.md), and the [Hive CE repository](https://github.com/IO-Design-Team/hive_ce) on 2026-10-25.
