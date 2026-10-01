---
layout: post
title: "firebase_crashlytics iOS Missing dSYM - Make Flutter Release Crashes Readable Again"
description: "Fix firebase_crashlytics Missing dSYM and obfuscated Flutter iOS release stacks by separating Xcode dSYM upload from split-debug-info symbol handling."
date: 2026-10-01
tags: [firebase_crashlytics, testing, iOS, Flutter, debugging]
comments: true
share: true
---

![Flutter iOS release crash symbolication workflow](https://images.unsplash.com/photo-1551288049-bebda4e38f71?auto=format&fit=crop&w=1600&q=80)

This fix is for Flutter teams that can see a crash in Firebase Crashlytics but only get `???`, addresses, or a **Missing dSYM** warning for an iOS release. It is useful when the app was archived successfully but the diagnostic symbols were not associated with the exact Firebase app and build. It is not a substitute for fixing a crash, and it does not make an old report readable when the matching symbols were never retained.

The important decision is to separate two symbol paths:

| Symptom | Symbol that is missing | Where to fix it |
| --- | --- | --- |
| Crashlytics shows “Missing dSYM” | Xcode/iOS framework dSYM | Xcode Run Script or manual `upload-symbols` |
| Dart frames remain obfuscated after `--obfuscate` | Flutter debug symbols and mapping | Flutter/Crashlytics symbol upload pipeline |
| Only some native frames say `(Missing)` | A framework or optional dSYM | Crashlytics dSYMs tab, then upload the exact UUID |

The image is a reminder that the build artifact, Firebase app, and crash report must belong to the same release identity.

## Why a successful archive can still produce unreadable crashes

An `.ipa` being accepted by TestFlight proves that the app can be distributed. It does not prove that Crashlytics received the symbols needed to decode its stack. Firebase’s current Flutter guide says `flutterfire configure` attempts to add a Crashlytics upload script, and that the script should appear as `[firebase_crashlytics] Crashlytics Upload Symbols` in the Runner target’s Build Phases. The phase must run after the relevant build products exist.

There is a second trap: `--split-debug-info` and `--obfuscate` change the Flutter symbol workflow. A team can repair the native dSYM upload and still see unreadable Dart frames because the Flutter symbols were not uploaded for that build. Treat those as two checks, not one.

## Repair the iOS upload path

Open `ios/Runner.xcworkspace`, not only the `.xcodeproj`, and inspect **Runner → Build Phases**. If the Crashlytics script is absent, add a final **Run Script Phase**. The command below is the validation form from Firebase’s documentation; run it first so a wrong App ID or missing input is visible during the archive.

```sh
$PODS_ROOT/FirebaseCrashlytics/upload-symbols \
  --build-phase --validate \
  -ai "$FIREBASE_APP_ID" \
  -- "$DWARF_DSYM_FOLDER_PATH/App.framework.dSYM"
```

After validation succeeds, the upload form is the same command without `--validate`:

```sh
$PODS_ROOT/FirebaseCrashlytics/upload-symbols \
  --build-phase \
  -ai "$FIREBASE_APP_ID" \
  -- "$DWARF_DSYM_FOLDER_PATH/App.framework.dSYM"
```

Use the **Firebase Apple App ID**, not the bundle ID. You can find it as `GOOGLE_APP_ID` in `GoogleService-Info.plist` or in Firebase Project settings. If the project has User Script Sandboxing enabled, add the dSYM, executable, and built `GoogleService-Info.plist` paths to the Run Script’s **Input Files**. Otherwise Xcode may prevent the script from reading the files even though the phase exists.

If the automatic phase is present but the console still lists a UUID, do not upload an arbitrary archive. Open the Crashlytics **dSYMs** tab, copy the missing UUID, and locate the matching dSYM from the same `.xcarchive`. A symbol from a different build can upload successfully and still solve nothing.

## Check Flutter obfuscation separately

For a build such as the following, preserve the symbol directory as a release artifact:

```sh
flutter build ipa \
  --obfuscate \
  --split-debug-info=build/symbols/1.4.0-42
```

The directory name is only an example. The useful rule is that every build gets a unique, retained directory tied to its version and build number. Do not put it in a temporary CI workspace that is deleted after the archive.

Firebase documents automatic Flutter symbol handling for Flutter 3.12.0+ with `firebase_crashlytics` 3.3.4+. If the release is still obfuscated, check the exact generated files and the upload step before changing Dart code. For Android, Firebase documents `firebase crashlytics:symbols:upload`; the iOS dSYM path remains the Xcode `upload-symbols` workflow above. The platform-specific artifact matters.

## A release checklist that catches the mismatch

```text
[ ] Same Firebase App ID as the iOS target being archived
[ ] Same bundle identifier and signing configuration as the shipped build
[ ] Crashlytics upload script is the last relevant Xcode build phase
[ ] Script Input Files include dSYM, executable, and GoogleService-Info.plist
[ ] Archive contains the expected App.framework.dSYM
[ ] Missing UUID from Crashlytics matches the archive's dSYM UUID
[ ] --split-debug-info output is retained outside the disposable CI workspace
[ ] A controlled test crash is sent after the symbol upload
```

Do not use a clean rebuild as proof that symbolication works. The meaningful test is an archive with the same configuration as the release, followed by a test crash and a check that the new issue has readable Dart and native frames. Also remember that uploading a missing dSYM later helps future processing; Firebase notes that it will not retroactively symbolicate crashes that have already been processed without the required symbols.

The practical rule is simple: **Missing dSYM means match and upload the iOS UUID; unreadable Dart frames after obfuscation means preserve and upload the Flutter symbol output.** Keeping those paths separate makes the failure reproducible and prevents repeated uploads of the wrong artifact.

Sources checked on 2026-10-01: [Firebase Flutter symbolication guide](https://firebase.google.com/docs/crashlytics/flutter/get-deobfuscated-reports), [Crashlytics troubleshooting](https://firebase.google.com/docs/crashlytics/troubleshooting), and [FlutterFire issue #13283](https://github.com/firebase/flutterfire/issues/13283).
