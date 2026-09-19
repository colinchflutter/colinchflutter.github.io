---
layout: post
title: "Isar Schema Migration - Prevent Data Loss from Renames and Enum Reordering"
description: "Fix Isar schema migration traps in Flutter: preserve renamed fields, migrate derived data safely, and prevent enum ordinal corruption."
date: 2026-09-19
tags: [isar, migration, database, Flutter, data_integrity]
comments: true
share: true
---

![Flutter database migration diagram showing a schema changing without losing stored records](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If a Flutter app stores user-created data in Isar, treat a Dart model rename and an enum reorder as data migrations, not refactors. Adding a nullable field is usually the low-risk path; renaming a persisted field needs `@Name`, and changing the order of an ordinal enum can silently turn an existing value into a different state. This approach fits local caches and offline records that can be rebuilt or migrated on-device. It is not enough for a database that must be restored from an external backup without a tested versioned procedure.

## The two failures that look harmless

Isar automatically updates its schema when collections, fields, or indexes are added or removed. That does not mean it knows the business meaning of your change. The official migration recipe separates schema migration from data migration: the first changes the structure, while the second reads old records, transforms them, and records an application version.

| Change in Dart | Risk on an existing database | Safer decision |
| --- | --- | --- |
| Add a nullable field | Existing rows have no meaningful value yet | Add it, then backfill in batches if queries need it |
| Rename a field or collection | Isar can delete and recreate the stored field or collection | Keep the old storage name with `@Name` |
| Reorder an `ordinal` enum | Old numeric values now point to another enum member | Freeze order or store `name`/an explicit value |
| Rename and transform a field | Identity and value conversion are separate problems | Preserve identity first, then run a versioned data migration |

The easy-to-miss case is a rename. Changing `displayName` to `name` changes the generated schema name unless the old name is retained explicitly:

```dart
@collection
class Profile {
  Id id = Isar.autoIncrement;

  @Name('displayName')
  String? name;
}
```

`@Name` is not cosmetic metadata. Isar's schema documentation warns that omitting it during a persisted rename can delete and recreate the field or collection. Apply the same rule to a class rename by keeping the previous collection name with `@Name('OldProfile')`.

## Enum changes need a storage policy

The default `@enumerated` representation uses the enum ordinal. If the stored sequence was `draft, pending, sent` and a new member is inserted before `pending`, old value `1` is now read as the new member. The database opens successfully, so a crash or migration error may never expose the problem.

For values that are part of a protocol or user-visible workflow, use a representation whose identity does not depend on declaration order:

```dart
enum SyncState { queued, uploaded, failed }

@collection
class Upload {
  Id id = Isar.autoIncrement;

  @Enumerated(EnumType.name)
  SyncState state = SyncState.queued;
}
```

`EnumType.name` stores the enum name, while `EnumType.value` can map storage to an explicit property. If an existing app already uses ordinals, do not simply switch the annotation and assume the old bytes will be interpreted correctly. Freeze the old order, add a compatibility reader, or create a deliberate conversion migration with fixtures for every old value.

## Version data migrations separately

Schema generation cannot infer a business rule such as “copy the old label into the new search key.” Keep a small database version outside the collection schema and advance it only after the data transaction succeeds. A simplified migration shape is:

```dart
Future<void> migrateIfNeeded(Isar isar, SharedPreferences prefs) async {
  final version = prefs.getInt('local_db_version') ?? 1;

  if (version < 2) {
    final total = await isar.profiles.count();

    for (var offset = 0; offset < total; offset += 50) {
      final rows = await isar.profiles
          .where()
          .offset(offset)
          .limit(50)
          .findAll();

      await isar.writeTxn(() async {
        // Transform and put the rows here.
        await isar.profiles.putAll(rows);
      });
    }

    await prefs.setInt('local_db_version', 2);
  }
}
```

The batch size is an example, not a benchmark. Pick it from record size and startup constraints, and move a large migration off the UI isolate. More importantly, make the operation idempotent: if the process is killed after one batch, reopening the app must safely repeat or resume that batch. Do not write version 2 before the final successful transaction.

## A reproducible release check

Before shipping a schema change, keep one fixture database made by the previous app version. Run this checklist in CI or a local release script:

- Open the old fixture with the new generated schema.
- Verify record counts and a few stable IDs before transforming values.
- Assert that every renamed field uses the old `@Name` value.
- Decode every persisted enum value, including records created by older versions.
- Kill or interrupt between migration batches, then reopen and run it again.
- Confirm that a failed migration does not advance `local_db_version`.
- Test a clean install separately; a missing version should follow the documented new-install path.

If the data is only a cache, deleting and rebuilding it may be cheaper than carrying a migration forever. If it contains drafts, downloads, or offline edits, deletion is a data-loss policy and should be treated as a product decision, not a recovery shortcut.

## Short rule set

Keep storage names stable with `@Name`, never reorder ordinal enums after release, and use an explicit version for transformations that change values. Isar handles structural schema updates, but your application owns the meaning of existing data. A small old-version fixture and an interruptible batch migration catch the failures that a clean-install test cannot see.

References: [Isar schema documentation](https://isar.dev/schema.html) and [Isar data migration recipe](https://isar.dev/recipes/data_migration.html).
