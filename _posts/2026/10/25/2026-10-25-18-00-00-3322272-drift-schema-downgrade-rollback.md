---
layout: post
title: "Drift Schema Downgrade - Keep Flutter Rollback Builds Safe"
description: "A production-focused Drift migration pattern for Flutter rollback builds: handle schema downgrades before stepByStep, preserve additive data, and test upgrade, rollback, and re-upgrade paths."
date: 2026-10-25
tags: [drift, migration, sqlite, rollback, testing, Flutter]
comments: true
share: true
---

![Drift SQLite schema migration and Flutter rollback build flow](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If a Flutter app uses Drift and you may ship a rollback build, handle the downgrade path in `onUpgrade` before calling generated migration steps. This applies to apps that keep local user data across store updates. It does not apply to a disposable cache that can be deleted and rebuilt; for that case, resetting the database is usually cheaper than maintaining reverse compatibility.

## The failure that only appears on a real device

Imagine the following release sequence:

| Build | Drift schema | Change |
| --- | ---: | --- |
| A | 11 | Current production build |
| B | 12 | Adds nullable columns |
| C | 11 | Rollback build with a higher store build number |

The device installs B, writes data, and then receives C. Drift does not treat C as a fresh database. It opens the existing file, sees `from = 12` and `to = 11`, and invokes `onUpgrade`. The `stepByStep` helper is designed for forward migrations, so it can throw before the database becomes available.

Putting the check in `beforeOpen` is too late. Drift documents that `beforeOpen` runs after migrations, while its API defines `onUpgrade` as the callback for both version upgrades and downgrades. A clean install test misses this because the database starts at version 0.

## Decide whether the rollback is structurally safe

Do not start by dropping every table. Separate these two questions:

1. Can the old binary open and query the newer SQLite file?
2. After the new binary returns, will it understand data written during the rollback window?

For an additive change such as new nullable columns, the old generated queries can usually ignore those columns. SQLite keeps them as `NULL`, so a no-op downgrade is often safer than destructive reconstruction. A new non-null constraint, renamed column, changed type, trigger, or index may need a different release policy.

## Guard the generated migration function

The following pattern keeps a downgrade branch explicit. The `stepByStep` function remains the source of truth for normal upgrades, but it is never called with `from > to`.

```dart
@override
int get schemaVersion => 12;

@override
MigrationStrategy get migration => MigrationStrategy(
      onCreate: (m) async {
        await m.createAll();
      },
      onUpgrade: (m, from, to) async {
        if (from > to) {
          // Schema 12 only added nullable columns.
          // Keep the file intact so offline data survives rollback.
          return;
        }

        await stepByStep(
          from11To12: (m, schema) async {
            await m.addColumn(schema.settings, schema.settings.darkModeEnabled);
            await m.addColumn(schema.reports, schema.reports.submittedAt);
          },
        )(m, from, to);
      },
      beforeOpen: (details) async {
        await customStatement('PRAGMA foreign_keys = ON');
      },
    );
```

The exact generated callback names depend on your schema history. Keep the direction check in application code even when the current migration is small: a future generated helper may deliberately reject downgrades, and that failure should be a release decision rather than an accidental startup crash.

This is not a universal downgrade recipe. If schema 12 removes or changes data that schema 11 requires, a no-op leaves semantic incompatibility behind. In that case, choose one of these policies before release:

| Database change | Safer rollback policy |
| --- | --- |
| Add nullable field | No-op downgrade; treat missing value as unknown |
| Add table or index | Usually keep it; old code can ignore it |
| Rename or remove field | Keep a compatibility column or block rollback |
| Change meaning of existing data | Forward-only release or explicit reverse transform |
| Destructive migration | Backup/export and restore plan; never hide it in `beforeOpen` |

## Make the 12 → 11 → 12 cycle testable

Drift's migration tooling can export schema snapshots and generate tests. The important test is not “the latest schema opens.” Start with an older schema, insert representative data, migrate, and verify both structure and values.

```dart
test('keeps data through the additive migration', () async {
  final schema = await verifier.schemaAt(11);
  final oldDb = Database(schema.newConnection());

  await oldDb.into(oldDb.settings).insert(
        SettingsCompanion.insert(name: 'demo'),
      );
  await oldDb.close();

  final migrated = Database(schema.newConnection());
  await verifier.migrateAndValidate(migrated, 12);

  final row = await (migrated.select(migrated.settings)).getSingle();
  expect(row.name, 'demo');
  expect(row.darkModeEnabled, isNull);
  await migrated.close();
});
```

For rollback itself, use a copied fixture file rather than uninstalling the app. The test sequence should be:

```text
schema 11 fixture
  -> open with schema 12 build and write old-compatible data
  -> open with schema 11 rollback build
  -> open again with schema 12 build
  -> verify rows, defaults, and newly added fields
```

On Android, build numbers still need to increase for the re-upgrade APK even when the Dart code returns to the newer schema. Use the same signing key for all three builds. Uninstalling between steps destroys the evidence you are trying to test.

## Release checklist

- [ ] Classify every schema change as additive, compatible, semantic, or destructive.
- [ ] Confirm whether `from > to` can occur for a store or closed-test rollback.
- [ ] Branch before invoking `stepByStep`.
- [ ] Keep schema snapshots under version control with the migration code.
- [ ] Test data preservation, not only `migrateAndValidate` schema equality.
- [ ] Run upgrade → rollback → re-upgrade on one installed device or fixture.
- [ ] Decide how values written by the rollback build are interpreted later.
- [ ] Avoid dropping tables as a generic recovery action; it deletes offline data and session state.

Drift gives you migration callbacks, schema exports, and verification helpers, but it cannot decide whether a rollback is semantically safe for your data. For additive changes, a guarded no-op downgrade plus a three-build fixture test is a practical boundary. For destructive changes, block rollback or ship an explicit compatibility migration instead of discovering the policy when the app cannot open.

Sources: [Drift migration guide](https://drift.simonbinder.eu/migrations/), [Drift migration testing](https://drift.simonbinder.eu/migrations/tests/), [MigrationStrategy API](https://pub.dev/documentation/drift/latest/drift/MigrationStrategy-class.html), and [OnUpgrade API](https://pub.dev/documentation/drift/latest/drift/OnUpgrade.html).
