# ddev-nativephp-android

A [DDEV](https://ddev.com) add-on that builds Android APKs for [NativePHP Mobile](https://nativephp.com) apps entirely inside a Docker container — no Android SDK, Java, or PHP required on your host machine.

## What it does

Adds an `android-builder` service to your DDEV project. This container has PHP 8.4 CLI, Composer, Java 17, and the Android SDK pre-installed. Your project files are shared with it via a volume mount, so `php artisan native:run` runs inside the container and produces an APK in your project directory — just like it would on a configured host.

## Requirements

- DDEV >= v1.22.0
- A Laravel project with [NativePHP Mobile v3](https://nativephp.com/mobile/docs) installed
- `native:install` must have been run at least once to create the `nativephp/android/` project skeleton

## Installation

```bash
ddev add-on get zeshanziya/ddev-nativephp-android
ddev restart
```

> **Note:** The first `ddev restart` after installation will take 5–15 minutes to build the image (downloads Android SDK ~1.5 GB). Subsequent restarts are instant — Docker caches the image layers, and Gradle dependencies are cached in a named volume.

## Usage

### Install NativePHP Android project (first time only)

```bash
ddev native-android-install
```

This runs `php artisan native:install` inside the android-builder container.

> Alternatively, run `ddev exec php artisan native:install` in the standard web container if you prefer.

### Build a release APK

```bash
ddev native-android-build
# or explicitly:
ddev native-android-build --release
```

The signed APK will be at:
```
nativephp/android/app/build/outputs/apk/release/app-release-unsigned.apk
```

### Build an AAB (for Google Play)

```bash
ddev native-android-build --bundle
```

Output: `nativephp/android/app/build/outputs/bundle/release/app-release.aab`

## Why release-only?

Debug builds require ADB to install the APK directly onto a connected device. ADB needs USB access which is not available inside Docker containers. For device testing during development, use `ddev native-jump` with the [Jump app](https://nativephp.com/mobile/docs/getting-started/jump) — it doesn't require a full build.

## Environment variables

Set these in your project's `.env` before building:

```env
NATIVEPHP_APP_ID=com.example.myapp
NATIVEPHP_APP_VERSION=1.0.0
NATIVEPHP_APP_VERSION_CODE=1
```

## How it works

```
┌─────────────────────────┐     ┌──────────────────────────────────┐
│  ddev-web               │     │  android-builder                 │
│  (Laravel / PHP-FPM)    │     │  PHP 8.4 CLI                     │
│                         │     │  Java 17 + Android SDK           │
│  /var/www/html ─────────┼─────┼▶ /var/www/html  (shared volume) │
│                         │     │                                  │
│  Browser dev / Jump     │     │  Gradle → APK/AAB output         │
└─────────────────────────┘     └──────────────────────────────────┘
```

- Project files are shared via bind mount — no copying needed
- Gradle dependency cache persists in a named Docker volume (`gradle-cache`)
- The Android SDK is baked into the image — no download on each restart

## Uninstalling

```bash
ddev add-on remove ddev-nativephp-android
ddev restart
```

Named volumes (`gradle-cache`) are removed automatically on uninstall.

## Contributing

Issues and PRs welcome. The key files are:

- `docker-compose.android-builder.yaml` — service definition and `dockerfile_inline`
- `commands/host/native-android-build` — the build command
- `commands/host/native-android-install` — the install helper command

## License

MIT
