# Tsumiru Classic

A patch stack on top of [Suwayomi-Tsumiru](https://github.com/Suwayomi/Suwayomi-Tsumiru)
that brings the app back to **Android 6.0.1 (API 23)** — one level below the
upstream minimum of API 24 (Android 7.0).

Verified end-to-end on a Quanta Ellipsis QTASUN1 (Android 6.0.1, arm64):
install, launch, TLS connection to a Suwayomi server over HTTPS, UI login,
and library loading all work.

> This is an unofficial fork. All credit for the app itself goes to the
> Suwayomi project. Licensed under MPL-2.0, same as upstream.

## Security notice

**Use at your own risk.** Keeping an old device usable means accepting
risks that upstream no longer carries:

- This fork **downgrades several plugins** to older versions and targets
  **Android 6, which has been out of OS security support for years**.
  Vulnerabilities fixed upstream after those versions stay unfixed here.
- The bundled ISRG root certificates are a **snapshot**; they do not
  update themselves. If the Let's Encrypt hierarchy changes, old builds
  may stop verifying servers until a new Classic build is made.
- Release APKs are signed with a **local, project-specific key** (not
  the upstream key, not a Play Store key). Only install builds from this
  repository's Releases, or build from source and sign with your own key.
- There is **no warranty and no security support**. You are responsible
  for your own security: prefer a dedicated device and trusted networks,
  and do not reuse credentials you would not risk on an unpatched
  Android 6 device.

## Why the stock app fails on API 23

1. **minSdk 24** — five plugins declare API 24 as their floor, so the APK
   will not even install on Android 6.
2. **Flutter engine `LocalizationPlugin`** calls
   `Configuration.getLocales()` (API 24+) without a guard → crash at startup.
3. **arm64 `libflutter.so`** references four libc symbols that only exist
   from API 24 (`__fwrite_chk`, `__write_chk`, `getifaddrs`, `freeifaddrs`,
   all `@LIBC_N`) → `UnsatisfiedLinkError` on Android 6.
4. **Android 6's platform trust store predates the ISRG roots**, so Dart
   cannot verify modern Let's Encrypt chains
   (`CERTIFICATE_VERIFY_FAILED: unable to get local issuer certificate`).

## What this fork changes (source level, branch `classic`)

- `pubspec.yaml` / `pubspec.lock`
  - Downgrades for the five API-24 plugins:
    `flutter_local_notifications 20.1.0`, `flutter_secure_storage 10.3.1`,
    `app_links 6.4.1`, plus `dependency_overrides` for
    `url_launcher_android 6.3.16` and `shared_preferences_android 2.4.7`.
  - `wakelock_plus` is vendored under `third_party/` with its manifest
    floor lowered to API 21 (the published 1.8.0 references
    `flutter.minSdkVersion` = 24).
  - Registers the `assets/certs/isrg_roots.pem` asset.
- `android/app/build.gradle` — minSdk via a `pocMinSdkVersion = 23`
  variable. A literal `23` does not survive: Flutter's
  MinSdkVersionMigration rewrites literal 16–23 back to
  `flutter.minSdkVersion` on every build.
- `android/gradle.properties` — `android.builtInKotlin=false`
  (the downgraded `shared_preferences_android` applies the Kotlin plugin
  the old way, which conflicts with AGP 9's built-in Kotlin mode).
- `lib/main.dart` + `assets/certs/isrg_roots.pem` — at startup the app
  loads the ISRG roots (X1, X2, Root YE, Root YR, official PEMs from
  letsencrypt.org) into `SecurityContext.defaultContext`, fixing TLS
  verification on Android 6. Harmless on newer Android versions.

No application feature code is changed; the downgraded plugin APIs were
verified compatible with every call site in the app.

## Binary post-patch (required after `flutter build apk`)

The Flutter engine problems (items 2 and 3 above) cannot be fixed from
Dart/Gradle, so the built APK is patched once, without a rebuild:

1. **apktool** decode → patch the `LocalizationPlugin` smali
   (R8 name in v1.6.4: `k10`) so both locale paths fall back to
   `Configuration.locale` when `SDK_INT < 24`.
2. Add a tiny `poc.App` `Application` subclass that preloads the shim,
   and point the manifest's `android:name` at it.
3. **`libpocshim.so`** (C source kept with the build notes): implements
   the fortify pair as checked wrappers and `getifaddrs`/`freeifaddrs`
   over NETLINK_ROUTE. Injected into the APK as `STORED`.
4. **`libflutter.so` (arm64)**: clear the `@LIBC_N` version requirement
   for the four symbols in `.gnu.version` (found by name via
   `readelf --dyn-syms`, not by fixed index) and
   `patchelf --add-needed libpocshim.so`.
5. Transplant the patched `classes*.dex` + `AndroidManifest.xml` back
   into the release APK, `zipalign -p`, re-sign with your own key.

The armeabi-v7a build needs only steps 1–2 and 5 (its engine binary has
no `LIBC_N` references).

## Building

```bash
# Flutter is pinned by upstream .fvmrc (3.47.9 for v1.6.4)
flutter pub get
flutter build apk --release --split-per-abi
# then apply the binary post-patch above before installing
```

Signing: create your own keystore and `android/key.properties`
(both are gitignored and intentionally not part of this fork).
Reusing the same key across builds keeps `adb install -r` upgrades
working.

## Keeping up with upstream

The patch stack is deliberately small and layered:

1. `git merge` (or rebase onto) the new upstream tag on `main`, then
   replay the `classic` commits.
2. Re-check the plugin floor table — upstream may raise a plugin back
   to an API-24 minimum; if a call site also changed, the downgrade may
   need a new target version.
3. Rebuild and re-run the binary post-patch. The smali class name and
   the engine symbol set are per-Flutter-version: verify both again
   (the patch steps locate them by name, and the smali patch must be
   re-checked against the new decode).
4. Smoke-test on a real API 23 device: install, launch, HTTPS connect,
   login, open a chapter.

If upstream ever fixes the `getLocales()` guard or the engine drops the
`LIBC_N` references, the corresponding post-patch step can simply be
dropped.

## Status

- v1.6.4 + Classic patches (see Releases):
  - arm64-v8a: **fully working** on a real Android 6.0.1 device.
  - armeabi-v7a: built with the same patch stack including the TLS fix.
  - universal (arm64 + armv7 + x86_64 in one APK): built with the same
    patch stack; the x86_64 engine binary carries the same `LIBC_N`
    references, so it receives the same shim treatment (x86_64 shim
    included). Not yet device-tested — there is no x86_64 Android 6
    device in the test pool.
