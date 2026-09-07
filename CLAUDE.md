# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Lawnchair 16: an Android home screen app built as a **fork of AOSP Launcher3** (Android 16 / `platform_frameworks_libs_systemui` at API 37). The repo is not a normal Android app project — it is the AOSP Launcher3 tree with a Gradle build bolted on top and Lawnchair's own code layered beside it. Most directories at the root (`src`, `quickstep`, `res`, `systemUI`, `protos`, `Android.bp`, `CleanSpec.mk`, `OWNERS`) are AOSP; Lawnchair's own code lives under `lawnchair/`.

Docs for contributors live at https://docs.lawnchair.app/developers (not in this repo).

## Build & checks

Requires JDK 21. The `platform_frameworks_libs_systemui` git submodule must be initialized (`git submodule update --init --recursive`) — the build fails without it.

```bash
./gradlew assembleLawnWithQuickstepGithubDebug   # main debug build used in CI
./gradlew assembleLawnWithQuickstepNightlyRelease
./gradlew assembleLawnWithQuickstepPlayDebug
./gradlew spotlessCheck                          # CI lint gate
./gradlew spotlessApply                          # auto-fix formatting
./gradlew lint                                   # Android lint (abortOnError = true)
```

Variant names come from three flavor dimensions — `app` (`lawn`) × `recents` (`withQuickstep`) × `channel` (`github` | `nightly` | `play`) — so a variant is e.g. `lawnWithQuickstepGithubDebug`. `play` is the default channel and has a different application id (`app.lawnchair.play`); `debug` builds get a `.debug` suffix.

Formatting is enforced by Spotless: `google-java-format` (AOSP style) for Java, `ktlint` + Compose rules for `lawnchair/src/**/*.kt` only. AOSP Java under `src/`, `quickstep/`, and `compatLib/` is deliberately excluded from Kotlin/Compose rules — do not reformat it wholesale.

There is **no JVM unit-test task wired into the Gradle build.** `tests/` contains the AOSP Launcher3 test tree (built via Soong/`Android.bp`, not Gradle); only `tests/shared` is compiled, as part of the `lawn` source set. The only Gradle-driven instrumented tests are `baseline-profile/` (Macrobenchmark, `./gradlew :baseline-profile:...`, uses a `pixel7Api36` managed device) and `modules/widgetpicker`'s `androidTest`. Don't promise to "run the tests" — verify by building the affected variant.

## Source layout

Gradle source sets are declared explicitly in the root `build.gradle` (there is no `src/main/java`):

| Path | Contents |
| --- | --- |
| `src/`, `res/`, `src_plugins/`, `protos/` | AOSP Launcher3 core (`com.android.launcher3`) |
| `quickstep/` | AOSP Recents/gesture-nav overlay, compiled only into `withQuickstep` variants |
| `lawnchair/src/` | Lawnchair's own code (`app.lawnchair`), plus `lawnchair/res`, `lawnchair/aidl`, `lawnchair/protos` |
| `compose/` | Compose facade + Compose-based Launcher3 features |
| `src_flags/`, `src_shortcuts_overrides/` | Small AOSP override source sets pulled into the `lawn` flavor |
| `systemUI/`, `platform_frameworks_libs_systemui/` (submodule) | Vendored SystemUI libs, included as Gradle subprojects (`:shared`, `:animation`, `:iconloaderlib`, …) |

The app `namespace` is `com.android.launcher3` even though Lawnchair classes are `app.lawnchair` — both live in one APK.

Manifests are merged from several files: `AndroidManifest-common.xml` (main), `lawnchair/AndroidManifest.xml` (`lawn`), `quickstep/AndroidManifest.xml`, plus per-channel manifests in `github/`, `nightly/`, and `quickstep/AndroidManifest-launcher.xml`.

## Working with AOSP code

- Edits to AOSP files are marked inline with `// LC-Note:` (a deliberate behavioral change) or `// Lawnchair-TODO:` (a known hack awaiting a better solution). Follow that convention — every future rebase onto a newer Launcher3 has to reconcile these by hand, so keep diffs against AOSP minimal and clearly annotated, and prefer adding code under `lawnchair/` over editing `src/`.
- The `lawn` flavor's `LawnchairLauncher` subclasses Launcher3's `Launcher`; `LawnchairLayoutFactory` swaps AOSP views for Lawnchair ones at inflation time. That subclass-and-inflate seam is the preferred integration point.
- `prebuilts/libs/*.jar` (`framework-16.jar`, `SystemUI-core-16.jar`, `WindowManager-Shell-16.jar`) are stub jars prepended to the compile classpath by the `addFrameworkJar` helper in the root `build.gradle`, giving access to hidden/`@SystemApi` platform APIs. They are compile-only — anything used from them must be guarded at runtime.
- Hidden-API access at runtime goes through `:hidden-api` with the Rikka Refine plugin and `restriction-bypass`.

## Lawnchair subsystems

- **Preferences — two coexisting systems.** `app.lawnchair.preferences.PreferenceManager` is the older SharedPreferences-backed one; `app.lawnchair.preferences2.PreferenceManager2` is the DataStore-backed one built on the `opto` library, with `SharedPreferencesMigration` bridging them. New preferences belong in `PreferenceManager2`. The settings UI is Compose (`lawnchair/src/app/lawnchair/ui/preferences/`, navigation via `destinations/`).
- **compatLib** (`compatLib/compatLibV{Q,R,S,T,U,V,Baklava}`) abstracts per-Android-version Quickstep/Recents system APIs so QuickSwitch-style Recents integration works on Android 10–16. See `compatLib/README.md`. Adding support for a new platform version means a new submodule here plus wiring in `settings.gradle` and the root `build.gradle`.
- **Compose facade** (`compose/facade/{core,enabled,disabled}`) lets Compose-dependent features be compiled out. Only the `enabled` variant is currently in the source set; keep `core` interfaces and both implementations in sync when adding to it.
- Other notable areas under `lawnchair/src/app/lawnchair/`: `theme/` (Material 3 / wallpaper color extraction, `ThemeProvider`), `icons/` (icon packs, shapes, themed icons), `search/` (global search algorithms and providers), `smartspace/` (At a Glance + Smartspacer), `qsb/` (search bar), `gestures/`, `data/` (Room; schemas in `schemas/`), `root/` (libsu).
- DI is Dagger/Hilt via KSP, on top of Launcher3's own Dagger graph (`com.android.launcher3.dagger`, `DaggerSingletonObject`).

## Conventions

- Kotlin for new Lawnchair code; the AOSP tree is mostly Java.
- Versioning: `versionCode` is written as underscore-grouped digits (`16_00_01_05_01`) and `versionName` is derived in `build.gradle` from `version` + `releaseName` for release refs, or `Dev.(<short sha>)` / `Dev.(#<run number>)` otherwise. A release build is detected purely from CI env (`GITHUB_REF` ending in `-alpha`/`-beta`/`-rc`/`-stable`), so local builds always report a Dev version.
- Translations are managed on Crowdin — edit only `res/values/strings.xml` and `lawnchair/res/values/strings.xml`; never hand-edit other `values-*` locale files.
- Third-party licenses are gated by the `licensee` plugin (Apache-2.0 / BSD-3 / GPL-2.0+ plus an explicit allowlist); a new dependency under another license needs an entry in the `licensee` block.
- CI (`.github/workflows/ci.yml`) runs on `*-dev` branches and PRs: three APK builds plus `spotlessCheck`. The development branch for this line is `16-dev`.
