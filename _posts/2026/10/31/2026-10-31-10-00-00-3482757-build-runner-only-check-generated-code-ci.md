---
layout: post
title: "build_runner --only-check - Catch Stale Flutter Generated Code in CI"
description: "Use build_runner --only-check to fail Flutter CI when json_serializable or other generated Dart files no longer match their sources."
date: 2026-10-31
tags: [build_runner, testing, performance, Flutter, CI]
comments: true
share: true
---
![build_runner checking Flutter generated files in a CI pipeline](/assets/images/shared-preferences-async-migration-cache-paths.png)

The key idea in the figure is the boundary between source models, generated files, and a CI decision.

The practical choice is simple: if your Flutter repository commits generated `.dart` files, run `dart run build_runner build --only-check` in CI and fail the job when the outputs are stale. It fits projects using `json_serializable`, `freezed`, or similar builders. It does not replace code generation for repositories that intentionally ignore generated files.

## The production failure this catches

Generated files can lag behind their annotations after a model change. A pull request may compile locally because the author’s watcher regenerated `user.g.dart`, while the branch contains only the old output. Reviewers then see a harmless-looking model diff, and the mismatch appears later on a clean checkout or in a package consumer.

`build_runner` 2.16.0 added `--only-check`. The command builds the expected outputs without writing them; if an output differs from the file on disk, it logs the difference and exits with failure. That is a better CI boundary than running a normal build and silently changing the workspace.

| Repository policy | CI command | Result |
| --- | --- | --- |
| Commit generated files | `dart run build_runner build --only-check` | Fails when output is missing or stale |
| Do not commit generated files | `dart run build_runner build` | Generates files for analysis/build |
| Local development | `dart run build_runner watch --delete-conflicting-outputs` | Keeps outputs current while editing |

The official `build_runner` README also separates package-path outputs from private `.dart_tool/build` artifacts. Do not commit the artifact tree just to make this check pass.

## A reproducible setup

Keep the generator versions in `pubspec.lock`, commit the generated source files, then add this job step:

```yaml
- name: Check generated Dart files
  run: dart run build_runner build --only-check
```

For a package with `json_serializable`, a minimal local reproduction is:

```bash
dart run build_runner build
# Change a field or JsonKey annotation, but do not commit the new .g.dart.
dart run build_runner build --only-check
echo $? # non-zero: the checked-in output is stale
```

The useful test is a clean checkout. Change an annotated model, run the generator once, commit the resulting `.g.dart`, then change the model again without regenerating. The check must fail. Regenerate and commit both files; the same command should pass without modifying the working tree.

## Traps around the flag

`--only-check` is a consistency check, not a repair command. Do not add `--delete-conflicting-outputs` expecting CI to fix the branch; a repair would hide the missing commit. Run the normal build locally, inspect the generated diff, and commit it deliberately.

If generated files are ignored by Git, a clean CI checkout will not contain the files that `--only-check` is meant to compare. Use a normal generation step in that policy, or change the repository policy before introducing this gate. Also pin the Dart and generator toolchain: different analyzer or builder versions can produce legitimate formatting or output changes.

### Short release checklist

- [ ] Generated source files are tracked, not only present in a developer’s workspace.
- [ ] `pubspec.lock` and the Dart/Flutter toolchain are reproducible in CI.
- [ ] Local generation passes before the check runs.
- [ ] CI uses `--only-check`, so it never rewrites the pull request workspace.
- [ ] A model annotation change has a test that proves stale output fails.

This workflow is most valuable when generated code is part of the published package or application source. If generation happens entirely inside CI and generated files are build artifacts, keep the normal build step and validate the final compile instead.

Sources checked on 2026-10-31: [build_runner 2.16.0 changelog](https://github.com/dart-lang/build/blob/master/build_runner/CHANGELOG.md), [build_runner README](https://github.com/dart-lang/build/blob/master/build_runner/README.md), and [Flutter JSON serialization guide](https://docs.flutter.dev/data-and-backend/serialization/json).
