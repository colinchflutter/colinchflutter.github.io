---
layout: post
title: "Drift schema migration - Keep Flutter database upgrades safe with generated steps and tests"
description: "Drift schema migrations can pass on a fresh install and still fail for users upgrading from an older app. Use generated steps, schema snapshots, and a reproducible migration test to protect existing SQLite data."
date: 2026-11-03
tags: [drift, migration, database, testing, Flutter, Android, iOS]
comments: true
share: true
---

![Drift schema migration protecting an existing Flutter SQLite database](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

The safe choice for a Flutter app using Drift is to test the upgrade path from every supported old schema, not only `onCreate` on a fresh install. A database that opens correctly after reinstall says nothing about a user moving from version 1 to version 4 with real rows. Drift's `make-migrations` workflow can generate schema snapshots, step-by-step migration code, and test scaffolding. Use it when the local database contains user data or when releases can skip intermediate app versions.

The image represents the part that is easy to miss: the deployed app is only one layer, while the database files already sitting on users' devices are older layers that still need a route forward.

## The failure pattern

Suppose the app has these schema changes:

| App database version | Change | Risk when skipped |
| --- | --- | --- |
| 1 | `notes` table exists | Baseline for existing installs |
| 2 | Add nullable `color` column | Old rows need a valid default or nullable field |
| 3 | Add `tags` and `note_tags` tables | Foreign keys and indexes must exist before queries run |
| 4 | Add `word_count` and backfill it | Data transformation can fail even when DDL succeeds |

The common mistake is writing only the final shape and assuming a fresh generated database proves the upgrade. It does not. A user on v1 may jump directly to the release containing v4. The upgrade callback must apply v2, v3, and v4 in order.

## Prefer generated migration steps

Create a schema snapshot before changing the database, then regenerate it after each versioned change. Drift's official migration tooling uses the snapshots to generate a step-by-step file and migration tests. The exact generated filenames can vary with the database name, but the workflow is reproducible:

```bash
# Run once before the first schema change, then again after each change.
dart run drift_dev make-migrations

# Generate normal Dart code after the migration code is ready.
dart run build_runner build --delete-conflicting-outputs
```

Configure the database and schema directories in `build.yaml` so CI can find the same inputs every time:

```yaml
targets:
  $default:
    builders:
      drift_dev:
        options:
          databases:
            app_database: lib/database/app_database.dart
          schema_dir: drift_schemas/
          test_dir: test/drift/
```

The important boundary is the schema snapshot, not the generated Dart file alone. If a generated file is deleted and recreated, the snapshot history is still what tells Drift how the schema evolved.

## Keep the upgrade callback independent from the current schema

A subtle production trap is asking the current database model to describe an older migration step. The current table definition already contains v4 columns, so a query built from it can refer to a column that did not exist yet while upgrading from v1. Keep each step tied to the historical schema object supplied by Drift:

```dart
import 'package:drift/drift.dart';

import 'database.steps.dart';

part 'app_database.g.dart';

@DriftDatabase(tables: [Notes, Tags, NoteTags])
class AppDatabase extends _$AppDatabase {
  AppDatabase(super.e);

  @override
  int get schemaVersion => 4;

  @override
  MigrationStrategy get migration => MigrationStrategy(
        onCreate: (Migrator m) => m.createAll(),
        onUpgrade: _schemaUpgrade,
      );
}

extension AppDatabaseMigrations on GeneratedDatabase {
  OnUpgrade get _schemaUpgrade => stepByStep(
        from1To2: (m, schema) async {
          await m.addColumn(schema.notes, schema.notes.color);
        },
        from2To3: (m, schema) async {
          await m.createTable(schema.tags);
          await m.createTable(schema.noteTags);
        },
        from3To4: (m, schema) async {
          await m.addColumn(schema.notes, schema.notes.wordCount);
          await customStatement(
            'UPDATE notes '
            'SET word_count = CASE '
            "WHEN content = '' THEN 0 "
            "ELSE LENGTH(content) - LENGTH(REPLACE(content, ' ', '')) + 1 "
            'END',
          );
        },
      );
}
```

The generated API may use a slightly different extension shape in the installed Drift version, so copy the structure from `database.steps.dart` rather than inventing method names. The rules stay the same: bump `schemaVersion`, apply intermediate steps, and avoid querying a table with its final shape before that step has run.

## Test the path a real user takes

The useful test is not “can a new database create all tables?” It is “can a v1 database containing representative rows become a valid v4 database without losing data?” Drift's generated migration test can validate the schema at each step. Add data-integrity assertions for transformations that the schema comparer cannot understand:

```dart
import 'package:drift_dev/api/migrations.dart';

@DriftSchemaTest(schemaDir: 'drift_schemas')
void main() {
  schemaTest('upgrade v1 to v4 keeps note content', (tester) async {
    final database = await tester.runMigrations(from: 1, to: 4);

    final row = await database.customSelect(
      'SELECT content, word_count FROM notes WHERE id = 1',
    ).getSingle();

    expect(row.read<String>('content'), isNotEmpty);
    expect(row.read<int>('word_count'), greaterThanOrEqualTo(1));
  });
}
```

This example assumes the fixture contains `notes.id = 1`; use the fixture format generated for the project and insert rows that exercise nulls, empty strings, long text, and foreign-key relationships. The assertion is intentionally about data, not only column existence.

## Release checklist

- [ ] `schemaVersion` increased exactly once for the release's schema change.
- [ ] A v1-to-current test exists if v1 is still a supported upgrade source.
- [ ] The test also covers the oldest version that production users can still have.
- [ ] New non-null columns have a migration default, a backfill, or are introduced nullable first.
- [ ] Data transformation runs after the required column or table exists.
- [ ] Foreign keys, indexes, and unique constraints are checked after the upgrade.
- [ ] CI runs `make-migrations`, code generation, and migration tests from a clean checkout.
- [ ] A failed migration is treated as a release blocker, not as a reinstall instruction.

Do not delete the local database as a release workaround. That may make a developer machine green while destroying drafts, cached work, or offline records on a user's device. During development, reinstalling can also be misleading because Android backup may restore the old database file.

Drift's manual migration API is still useful for a small one-off change, but generated steps are the better default once more than one version must be supported. The decision is simple: fresh installs test the final schema; migration tests protect the upgrade contract that existing users actually depend on.

Sources checked 2026-11-03: [Drift migrations](https://drift.simonbinder.eu/migrations/), [Drift Migrator API](https://pub.dev/documentation/drift/latest/drift/Migrator-class.html), and [Drift schema testing API](https://pub.dev/documentation/drift_dev/latest/api_migrations/).
