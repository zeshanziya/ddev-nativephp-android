# ddev-nativephp-android

A [DDEV](https://ddev.com) add-on that builds signed Android APKs for [NativePHP Mobile](https://nativephp.com) apps entirely inside a Docker container — no Android SDK, Java, or PHP required on your host machine.

## What it does

Adds an `android-builder` service to your DDEV project containing PHP 8.4 CLI, Composer, Java 17 (JDK), and the Android SDK. Your project files are shared via a volume mount, so artisan commands run inside the container and produce output (APKs, keystores) directly in your project directory.

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

### 1. Install NativePHP Android project (first time only)

```bash
ddev native-android-install
```

This runs `php artisan native:install` inside the android-builder container.

### 2. Generate signing credentials (first time only)

```bash
ddev native-android-credentials
```

This runs `php artisan native:credentials android` using the JDK `keytool` inside the container. It will:
- Prompt for a keystore filename, alias, and passwords
- Write the keystore to `credentials/app-release-key.jks` in your project
- Add `ANDROID_KEYSTORE_*` variables to your `.env`

> **Tips:**
> - Press Enter to accept the default keystore filename (`app-release-key.jks`)
> - The key password must be **at least 6 characters**
> - Press Enter at the key password prompt to reuse the keystore password

After generation, make sure the path in `.env` includes the directory:

```env
ANDROID_KEYSTORE_FILE=credentials/app-release-key.jks
```

> **⚠️ Back up your keystore!** The `credentials/` folder is in `.gitignore` and will not be committed. If you lose the keystore, you cannot publish updates to the Play Store. Store it securely in a password manager or encrypted cloud storage.

### 3. Build a signed APK

```bash
ddev native-android-build --release
```

The signed APK will be at:
```
nativephp/android/app/build/outputs/apk/release/app-release.apk
```

### 4. Build an AAB (for Google Play)

```bash
ddev native-android-build --bundle
```

Output: `nativephp/android/app/build/outputs/bundle/release/app-release.aab`

### 5. Install on your Android device

Transfer the APK to your phone (USB, Google Drive, WhatsApp, etc.), then:

1. Enable **Install unknown apps** in Android Settings → Security
2. Open the APK on your phone and tap Install

## Required .env variables

```env
NATIVEPHP_APP_ID=com.example.myapp
NATIVEPHP_APP_VERSION=1.0.0
NATIVEPHP_APP_VERSION_CODE=1

ANDROID_KEYSTORE_FILE=credentials/app-release-key.jks
ANDROID_KEYSTORE_PASSWORD=your-keystore-password
ANDROID_KEY_ALIAS=your-key-alias
ANDROID_KEY_PASSWORD=your-key-password
```

## Why release-only?

Debug builds require ADB to push the APK directly to a connected device. ADB needs USB access which is not available inside Docker containers. For device testing during development, use `ddev native-jump` with the [Jump app](https://nativephp.com/mobile/docs/getting-started/jump) — it doesn't require a full build.

## How it works

```
┌─────────────────────────┐     ┌──────────────────────────────────┐
│  ddev-web               │     │  android-builder                 │
│  (Laravel / PHP-FPM)    │     │  PHP 8.4 CLI + Composer          │
│                         │     │  Java 17 JDK (keytool)           │
│  /var/www/html ─────────┼─────┼▶ /var/www/html  (shared volume) │
│                         │     │  Android SDK + Gradle            │
│  Browser dev / Jump     │     │  → signed APK/AAB output         │
└─────────────────────────┘     └──────────────────────────────────┘
```

- Project files are shared via bind mount — no copying needed
- Gradle dependency cache persists in a named Docker volume (`gradle-cache`)
- The Android SDK and JDK are baked into the image — no download on each restart

## Available commands

| Command | Description |
|---|---|
| `ddev native-android-install` | Run `native:install` inside the android-builder container |
| `ddev native-android-credentials` | Generate Android signing keystore via `keytool` |
| `ddev native-android-build --release` | Build a signed release APK |
| `ddev native-android-build --bundle` | Build a signed AAB for Play Store |

## Uninstalling

```bash
ddev add-on remove ddev-nativephp-android
ddev restart
```

The `gradle-cache` named volume is removed automatically on uninstall.

## Contributing

Issues and PRs welcome. Key files:

- `docker-compose.android-builder.yaml` — service definition with `dockerfile_inline`
- `commands/host/native-android-build` — builds signed APK/AAB via `native:package`
- `commands/host/native-android-credentials` — generates keystore via `native:credentials`
- `commands/host/native-android-install` — runs `native:install`

## License

MIT
