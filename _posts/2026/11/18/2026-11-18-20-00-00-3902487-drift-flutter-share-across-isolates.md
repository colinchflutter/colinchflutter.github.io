---
layout: post
title: "drift_flutter shareAcrossIsolates - Keep Flutter Background Writes and watch() Streams in Sync"
description: "Use drift_flutter shareAcrossIsolates when Flutter UI and background isolates touch one database, with the close, engine, and stream boundaries that prevent stale reads."
date: 2026-11-18
tags: [drift_flutter, drift, database, performance, testing, Flutter]
comments: true
share: true
---

![drift_flutter sharing one SQLite database between Flutter isolates](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If a Flutter app has a UI isolate and a background worker that both read or write Drift data, use `drift_flutter` with `shareAcrossIsolates: true` when they run inside the same Flutter engine. This is the useful choice for a `WorkManager`-style worker that must update a UI `watch()` query. It is not a bridge between two independently created Flutter engines, and it is not automatically faster than a single database connection.

## The failure that looks like a database bug

Opening the same SQLite file twice does not create one logical Drift database. Each isolate can have its own connection, query stream, and update notification path. A background insert may be durable on disk while the UI stream remains quiet until the next manual reload.

Drift’s isolate documentation separates these cases:

| Setup | What is shared | UI `watch()` updates from the worker | Choice |
| --- | --- | --- | --- |
| Two independent database instances | SQLite file only | Not guaranteed | Avoid for reactive cross-isolate data |
| `shareAcrossIsolates: true` | Drift database server discovered by name | Yes, inside one Flutter engine | Good default for related isolates |
| Two Flutter engines | No shared `IsolateNameServer` discovery | No with this option | Use a process-safe queue or a connection-pool design |

The distinction matters when a background callback succeeds but the foreground screen still shows old data. Re-running the query can hide the architectural mistake without fixing the stream boundary.

## Minimal setup with one database definition

The database class needs a constructor that accepts the connection returned by `driftDatabase`. Keep the database name stable; it is part of how the shared instance is found.

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

part 'app_database.g.dart';

class InboxItems extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get body => text()();
  DateTimeColumn get createdAt => dateTime()();
}

@DriftDatabase(tables: [InboxItems])
class AppDatabase extends _$AppDatabase {
  AppDatabase(super.e);

  AppDatabase.defaults()
      : super(
          driftDatabase(
            name: 'app_db',
            native: DriftNativeOptions(
              shareAcrossIsolates: true,
            ),
          ),
        );

  @override
  int get schemaVersion => 1;

  Stream<List<InboxItem>> watchInbox() => select(inboxItems).watch();
}
```

Both isolates must construct `AppDatabase.defaults()` with the same `name` and compatible database path. Do not pass an `AppDatabase` object through `Isolate.run`; Drift databases contain streams and futures that cannot be serialized between isolates. Construct a database instance in each isolate instead.

## The background callback still needs its own lifecycle

The sharing option does not turn a background callback into a UI callback. The worker must initialize the Flutter binding and close its database instance when its work is complete. The exact callback signature varies by the worker package, so the following is the relevant boundary rather than a copy-paste `Workmanager` registration.

```dart
import 'package:flutter/widgets.dart';

@pragma('vm:entry-point')
Future<bool> syncInboxInBackground() async {
  WidgetsFlutterBinding.ensureInitialized();

  final db = AppDatabase.defaults();
  try {
    await db.into(db.inboxItems).insert(
      InboxItemsCompanion.insert(
        body: 'Background sync completed',
        createdAt: DateTime.now(),
      ),
    );
    return true;
  } finally {
    await db.close();
  }
}
```

`drift_flutter` starts the shared database isolate when the first connection is made and can stop it only after attached databases are closed. Forgetting `close()` may keep resources alive after a short-lived worker finishes. Closing the worker connection does not mean the UI connection has been closed; each owner should release its own instance.

## Do not enable it for every database

The official Drift documentation says that background-isolate execution reduces UI blocking but adds copying and messaging overhead. A small settings database accessed only by the UI isolate gains little from this option. A large import, notification callback, or worker that must update a live query has a clearer reason to use it.

Use this checklist before changing the connection:

- [ ] The foreground and worker are isolates in the same Flutter engine.
- [ ] Both use the same database name and path.
- [ ] The worker creates its own database instance.
- [ ] The worker callback is reachable in release mode and initializes its dependencies.
- [ ] Every short-lived database owner calls `close()` in `finally`.
- [ ] The test checks a `watch()` stream, not only that a row exists after reopening.

A practical test creates the UI database, subscribes to `watchInbox()`, runs one background-isolate insert, and waits for the emitted row. Repeat the test with two independent connections to document the failure your application is avoiding. Also test a terminated-app worker separately: `shareAcrossIsolates` only discovers databases inside the same Flutter engine, so a second engine needs a different coordination design.

The decision is narrow: use `shareAcrossIsolates` for synchronized reactive access among isolates that can discover the same Drift server; use a queue, server-side sync, or an explicitly coordinated connection pool when the work crosses engine boundaries. That boundary prevents a durable write from being mistaken for a delivered UI update.

Sources checked on 2026-11-18: [drift_flutter package documentation](https://pub.dev/packages/drift_flutter), [Drift isolate guide](https://drift.simonbinder.eu/isolates/), and [`DriftNativeOptions` API](https://pub.dev/documentation/drift_flutter/latest/drift_flutter/DriftNativeOptions-class.html).
