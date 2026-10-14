---
layout: post
title: "sqflite database is locked - Fix Flutter's Async Database Opening Race"
description: "A production-safe sqflite pattern for preventing database is locked errors: share one opening Future, keep one connection, and keep isolate access at a clear boundary."
date: 2026-10-14
tags: [sqflite, SQLite, persistence, concurrency, Flutter]
comments: true
share: true
---

![Flutter sqflite single database connection and async opening race](https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1600&q=80)

If a Flutter app can open the same `sqflite` file from repositories, providers, and startup code, make the database holder a single shared service. The important fix is not adding random delays around queries: cache the in-flight `openDatabase()` future, then expose the same `Database` instance to every caller. This fits Android and iOS apps where local data matters; it is unnecessary for a throwaway cache that can be deleted and rebuilt.

## Why the error appears only under load

`openDatabase()` is asynchronous. A nullable field by itself does not prevent two callers from seeing `null` before the first open finishes.

```dart
Database? _db;

Future<Database> get database async {
  if (_db != null) return _db!;
  _db = await openDatabase(path);
  return _db!;
}
```

Imagine `SyncRepository` and `SettingsRepository` request the database during the same frame. Both enter before the `await` completes. They can create separate handles, or one caller can retain a handle while another code path closes and reopens the file. The failure then looks random: cold start works, a refresh or background sync produces `SQLiteDatabaseLockedException`, or a write is observed by one connection later than expected.

The sqflite maintainer recommends keeping one global database reference to avoid lock issues. The package also warns that opening the same database more than once with `singleInstance: false` can produce a locked database. The race is therefore in the application’s ownership model, not something a retry loop should hide.

## Cache the opening future, not only the result

Put path creation, schema versioning, and the open operation behind one repository. The cached `Future` closes the gap between “not opened” and “opened”.

```dart
import 'package:path/path.dart' as p;
import 'package:sqflite/sqflite.dart';

class AppDatabase {
  AppDatabase._();
  static final AppDatabase instance = AppDatabase._();

  Future<Database>? _opening;

  Future<Database> get database => _opening ??= _open();

  Future<Database> _open() async {
    final root = await getDatabasesPath();
    final file = p.join(root, 'app.db');

    return openDatabase(
      file,
      version: 3,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE notes(
            id INTEGER PRIMARY KEY,
            body TEXT NOT NULL,
            updated_at INTEGER NOT NULL
          )
        ''');
      },
      onUpgrade: (db, oldVersion, newVersion) async {
        if (oldVersion < 2) {
          await db.execute(
            'ALTER TABLE notes ADD COLUMN updated_at INTEGER NOT NULL DEFAULT 0',
          );
        }
        if (oldVersion < 3) {
          await db.execute(
            'CREATE INDEX notes_updated_at ON notes(updated_at)',
          );
        }
      },
    );
  }
}
```

The `??=` assignment happens before `_open()` completes, so later callers receive the same future. If opening fails, the future completes with an error and remains cached. That is useful for exposing a real startup failure, but a long-lived app may want an explicit reset method after logging and fixing a recoverable path problem:

```dart
Future<void> close() async {
  final db = await _opening;
  await db?.close();
  _opening = null;
}
```

Call `close()` only at an application-owned shutdown boundary. Do not let a screen call it from `dispose()` while another repository is still active.

## Keep transactions and isolates within the same boundary

A single connection prevents most accidental lock contention, but it does not make unrelated writes atomic. Group related changes in one transaction and await every write before starting the next state transition.

```dart
Future<void> saveNote(Note note) async {
  final db = await AppDatabase.instance.database;
  await db.transaction((txn) async {
    await txn.insert(
      'notes',
      note.toMap(),
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
  });
}
```

Keep native `sqflite` access in the main isolate. Dart isolates do not share memory, and the sqflite documentation specifically warns that its transaction mechanism is not cross-isolate safe. If a background worker needs database work, send a command to one database owner or persist a file/message for the main isolate to consume. Creating another `AppDatabase.instance` in a new isolate does not create a process-wide singleton.

| Situation | Safer choice | Why |
| --- | --- | --- |
| Multiple repositories in one isolate | One cached opening `Future` | Removes the async initialization race |
| Related insert and update | One `transaction` | Prevents partial state |
| Widget leaves the screen | Keep the database open | Screen lifecycle is not database lifecycle |
| Background isolate needs a write | Queue to the database owner | Avoids cross-isolate transaction assumptions |
| Disposable cache | Recreate after clearing | Lock recovery matters less than durability |

## A short reproduction checklist

Before shipping a schema or repository refactor, run these cases against a copied test database:

1. Start two repository reads without awaiting either call, then assert both receive the same opening future.
2. Start two writes at the same time and verify both are awaited; related writes should use one transaction.
3. Upgrade a database from each supported old version, not only from a fresh install.
4. Navigate away while a query is running; confirm no widget `dispose()` closes the shared database.
5. If using a worker isolate, verify that it does not open the native database directly.

The practical boundary is simple: one owner opens the file, repositories borrow that connection, transactions describe atomic work, and screens never manage database lifetime. That makes `database is locked` a diagnosable ownership bug instead of a timing lottery.

References: [sqflite usage recommendations](https://github.com/tekartik/sqflite/blob/master/sqflite/doc/usage_recommendations.md), [sqflite database opening guidance](https://github.com/tekartik/sqflite/blob/master/sqflite/doc/opening_db.md), and [Flutter SQLite recipe](https://docs.flutter.dev/cookbook/persistence/sqlite).
