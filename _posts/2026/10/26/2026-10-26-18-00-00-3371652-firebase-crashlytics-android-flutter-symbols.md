---
layout: post
title: "firebase_crashlytics Android Flutter Symbols - Decode Obfuscated Release Crashes"
description: "Fix unreadable Android Flutter release stacks in firebase_crashlytics by preserving split-debug-info output and uploading the exact symbols for each obfuscated build."
date: 2026-10-26
tags: [firebase_crashlytics, Flutter, Android, testing, performance]
comments: true
share: true
---

![firebase_crashlytics Android Flutter symbol upload flow](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If an Android Flutter release uses `--obfuscate`, upload the matching `--split-debug-info` directory for every build; otherwise Firebase Crashlytics can receive the crash while Dart frames remain unreadable. This applies to teams that want smaller or harder-to-reverse-engineer release binaries. It does not fix a missing native `.so` symbol file or make an old build readable when its symbol output was discarded.

## The failure has two different symbol paths

Flutter obfuscation renames Dart symbols. Android native code has its own symbol artifacts. Treating both as “Crashlytics symbols” creates a misleading green build: the app ships, the crash appears in the console, but the useful Dart method names are gone.

| Symptom in Crashlytics | Artifact to check | Typical upload path |
| --- | --- | --- |
| Dart frames show short or meaningless names | Flutter symbol directory from `--split-debug-info` | Firebase CLI `crashlytics:symbols:upload` |
| Native frames show addresses or missing symbols | Native Android symbol files | Crashlytics Gradle/native symbol workflow |
| The wrong release is decoded | Build-specific symbol directory or Firebase App ID mismatch | Rebuild the mapping and verify the exact app/build |

The Firebase Flutter guide checked on 2026-10-26 requires Firebase CLI 11.9.0 or newer for Flutter debug-symbol upload. The `--app` value is the Firebase Android App ID, not the Android package name.

## Make the build output reproducible

Put the version and build number in the symbol directory. A disposable CI path such as `/tmp/symbols` makes a later incident impossible to repair.

This is the release command whose output must be retained:

```bash
VERSION="1.4.0-42"
SYMBOL_DIR="build/symbols/android-$VERSION"

flutter build appbundle \
  --release \
  --obfuscate \
  --split-debug-info="$SYMBOL_DIR"
```

Do not regenerate symbols with a later source checkout and assume they will match. The mapping belongs to the exact Dart program and build configuration that produced the shipped binary. Store the directory as a protected CI artifact beside the AAB metadata, commit SHA, Flutter version, and Firebase Android App ID.

## Upload the Flutter symbols after the build

Run this after the AAB build succeeds and before deleting the workspace. The command is intentionally separate from native symbol handling.

```bash
FIREBASE_ANDROID_APP_ID="1:1234567890:android:abcdef123456"
SYMBOL_DIR="build/symbols/android-1.4.0-42"

firebase crashlytics:symbols:upload \
  --app="$FIREBASE_ANDROID_APP_ID" \
  "$SYMBOL_DIR"
```

A useful CI guard is to fail when the directory is empty instead of reporting a successful upload step with no files:

```bash
test -d "$SYMBOL_DIR" && find "$SYMBOL_DIR" -type f -print -quit | grep -q . \
  || { echo "Flutter symbol output is missing: $SYMBOL_DIR" >&2; exit 1; }
```

Keep credentials and the Firebase App ID in CI configuration. The public article only needs the upload shape; it should not contain a real project ID or service-account secret.

## Recovery checklist for an unreadable crash

1. Compare the Crashlytics app with the Firebase App ID used by the upload command. A package name that looks similar is not sufficient.
2. Match the crash’s version and build number to the symbol directory name and the shipped AAB.
3. Check whether the release used both `--obfuscate` and `--split-debug-info`.
4. Confirm the exact symbol directory survived CI cleanup and was uploaded with Firebase CLI 11.9.0+.
5. Test with a new release crash. A clean local rebuild only proves that the project still builds; it does not prove that Crashlytics can decode the already shipped artifact.

Uploading a missing symbol later helps Crashlytics process reports that still have the required raw data, but it is not a substitute for retaining build outputs. The safe policy is simple: every obfuscated Android build gets an immutable symbol artifact, an explicit Firebase App ID, and an upload log tied to the release.

Sources checked on 2026-10-26: [Firebase Flutter symbolication guide](https://firebase.google.com/docs/crashlytics/flutter/get-deobfuscated-reports) and [Flutter obfuscation documentation](https://docs.flutter.dev/deployment/obfuscate).
