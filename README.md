# SpoofX Manager

LSPosed module that spoofs Android `Build.*` fields and selected system
properties for targeted apps, with native maps stealth, plus a Manager
APK with a Material 3 native UI for managing per-package profiles.

This repository started life as a Zygisk + Magisk module
(`local_cxa_atexit_finalize_impl` + Dobby hooks). It has been ported to
the modern [libxposed](https://github.com/libxposed/api) API
([API 102](https://github.com/libxposed/api/releases/tag/102.0.0); compile pin `102.0.0` in `gradle/libs.versions.toml`) and now ships
as a regular Android APK that LSPosed loads as a Native Hook module.

The legacy zygisk path (Magisk system overlays, `customize.sh`,
`service.sh`, the Gradle-free `build.py` `SpoofX.zip` packager, the
Dobby submodule) has been removed; see the git history for context.

## Layout

```
lsmodule/                Android APK (Manager + LSPosed module)
├── build.gradle.kts     :lsmodule build script (libxposed api/service; see gradle/libs.versions.toml)
└── src/main/
    ├── AndroidManifest.xml
    ├── cpp/             libspoofx_native.so — hook entry, spoof, maps stealth
    ├── java/com/thoittxl/spoofx/
    │   ├── SpoofXModule.kt       libxposed entry: applyProfile, then Build.* reflection
    │   ├── SpoofXNative.kt       JNI bridge to libspoofx_native.so
    │   ├── SpoofXApplication.kt  manager Application: binds XposedService
    │   ├── MainActivity.kt       Material 3 root activity + bottom nav
    │   ├── DeviceProfile.kt      Build.* + system-property field carrier
    │   └── ui/                     Fragments per bottom-nav tab
    ├── resources/META-INF/xposed/  module.prop, java_init.list, native_init.list, scope.list
    └── res/                        Material 3 theme, drawables, layouts, vi/en strings
```

## Build

For Codex operating from the Termux checkout, all Gradle and APK builds run in
GitHub Actions. Do not install the Android toolchain or run Gradle in Termux.

`.github/workflows/android.yml` (`Android CI`) runs the LSPosed safety gate,
release host-JVM tests, and the release APK build on pushes and pull requests to
`dev-apk`, `dev-next`, or `dev-codex`. It can also be started manually for an
already-pushed ref through `workflow_dispatch`. Successful runs upload
`SpoofXManager*-release` for three days.

The workflow configures JDK 21, the hosted Android SDK, and recursive
submodules on `ubuntu-latest`. The NDK (`29.0.14206865`) and CMake are
provisioned by AGP auto-download driven by `ndkVersion` in
`lsmodule/build.gradle.kts` — the workflow has no explicit
`sdkmanager` install step for them. A local Termux inspection is not
build proof; use the GitHub Actions run result and artifact.

## Install

1. Install an LSPosed/libxposed framework that implements API ≥ 102.
2. `adb install lsmodule-debug.apk` (or sideload the release APK).
3. Open the LSPosed manager → enable **SpoofX** → pick the apps to scope.
4. Open SpoofX, create a device profile, and assign it to a target app.

The manager UI is a native Material 3 Android app (Vietnamese first,
English fallback). Bottom navigation hosts six tabs: Home, Profiles,
Add, Alerts, Advanced, Settings. Profiles are persisted locally and
mirrored to remote preferences when the libxposed service is bound.

## Storage

| Key | Where | Purpose |
|---|---|---|
| `_config` | local `spoofx_local` prefs + `XposedService.getRemotePreferences("profiles")` | Profile JSON blob (`PACKAGES_<KEY>` arrays + `PACKAGES_<KEY>_DEVICE` objects) |
| `_ignorelist` | same | Newline-separated package names to skip spoofing |

The manager writes both local + remote prefs (so the UI works even
without LSPosed bound). Hooked target processes only read the remote
prefs via `XposedInterface.getRemotePreferences`.

## Hook surface

`lsmodule/src/main/cpp/spoofx_native/` registers six hooks via the libxposed Native API
(captured at `native_init` time), declared in the `kHooks[]` table in
`hook_install.cpp`:

- `__system_property_read_callback`
- `__system_property_get`
- `__system_property_foreach` (optional — enumeration filter; self-disables if unresolvable)
- `open` / `openat` (selective `/proc/self/maps` filtering + path-spoof registry for `/proc/version`/`osrelease`)
- `uname` (optional — libc wrapper around `SYS_uname`; gated on the `spoof_kernel_strings` profile toggle)

The binder `ioctl` trampoline (`lsmodule/src/main/cpp/binder_hook/`,
`dispatcher.cpp`) is also installed eagerly in `native_init`; it is a
pass-through tail-call until the `binder_hook` profile toggle activates a
handler.

`dl_iterate_phdr` is **called** internally to enumerate the module's own
PT_LOAD segments (input to the maps filter) but is **not** itself
hooked yet — see the roadmap.

Java side, `SpoofXModule.onPackageLoaded` reads the per-package profile
from remote prefs and calls `SpoofXNative.applyProfile` **first**: the native
side validates the profile and installs the property + maps-cleaning hooks
atomically — if it rejects the profile, `Build.*` is never touched. Only after
`applyProfile` returns `true` does `applyBuildFields` reflect the profile into
`Build.*`, so the native side and the framework end up with exactly the same
values the framework will see.

## Roadmap

- Hook `dl_iterate_phdr` itself to hide `libspoofx_native.so` from
  module-list enumeration.
- Build out the remaining bottom-nav tabs (Profiles list, Profile
  editor with Build.* form, Settings, etc.) — PR 7 ships only the Home
  tab; the others render placeholders.
- Optional: hook `IPackageManager.hasSystemFeature` /
  `getSystemAvailableFeatures` in `onSystemServerStarting` to restore the
  Pixel-exclusive feature parity that the legacy zygisk Magisk overlay
  used to provide. This requires a dedicated system-server lifecycle and
  architecture review before the current fail-closed guard can change.
  (Sysconfig XML data was removed in this repo
  because LSPosed cannot mount over `/system/etc/sysconfig/`.)

## Acknowledgments

- [libxposed/api](https://github.com/libxposed/api) — modern Xposed
  module API (current upstream **102.0.0**).
- [libxposed/service](https://github.com/libxposed/service) — module-app
  ↔ framework IPC (scope, remote prefs, running targets / hot reload in 102).
- [LSPosed](https://github.com/LSPosed/LSPosed) — the framework
  hosting the module.
- [Material Components for Android](https://github.com/material-components/material-components-android)
  — the native UI toolkit.

## License

This project is licensed under the **GNU General Public License v3.0
or later** (GPL-3.0-or-later).  See the [`LICENSE`](LICENSE) file at
the repository root for the full text.

The hidden-api stub files vendored from
[AppManager](https://github.com/MuntashirAkon/AppManager) under
[`hidden-api/`](hidden-api/) are individually licensed under the
**Apache License 2.0** and retain their original
`SPDX-License-Identifier` headers.  Apache-2.0 is one-way compatible
with GPL-3.0; the project as a whole ships under GPL-3.0 while the
vendored stubs preserve their upstream attribution.

Third-party component attributions (Maven dependencies, vendored
sources) live in the [`NOTICE`](NOTICE) file at the repository root.
That file also contains the placeholder for the
MorpheApp/morphe-patches attribution that will land with Phase 2 of
the multi-purpose-profile rollout — until then, no Morphe code is
shipped in this repository.
