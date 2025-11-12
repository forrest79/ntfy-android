# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is the Android app for [ntfy](https://github.com/binwiederhier/ntfy) ([ntfy.sh](https://ntfy.sh)), a pub-sub
notification service. This repo (`forrest79/ntfy-android`, branch `build`) is Jakub's personal fork of the
upstream project, rebased on top of upstream history and customized to build and self-sign a **Play-flavor**
release that talks to his **private ntfy server** instead of ntfy.sh. Keep upstream-derived files mergeable when
possible; personal/build-only changes are isolated in "personal build" style commits (currently just the latest
one, `3ae6827f`). That commit's changes to `app/build.gradle` / `app/src/main/res/values/values.xml`:

- `app_base_url` → `https://ntfy.trmota.cz` (the default/hardcoded server baked into the app — see
  `values.xml`; every screen that falls back to a default server reads this string).
- `applicationId` → `cz.trmota.ntfy` (was `io.heckel.ntfy`) so it installs side-by-side with upstream ntfy and
  matches this private server's expected app ID.
- Added a `rel` signing config wired to the committed `release-keystore.jks`, and pointed the `release` build
  type at it, so `assemblePlayRelease` produces a self-signed installable APK without extra setup.
- `play` flavor's `RATE_APP_AVAILABLE` flipped to `false` (no Play Store listing to rate).

If the private server's base URL or app ID ever changes again, update `values.xml` / `app/build.gradle` together
— note the existing comment that `app_base_url` changes must also be mirrored in `google-services.json` (Firebase
config, only relevant to the `play` flavor).

Written in Kotlin, single-module Gradle project (`:app`), no Compose — classic Fragment/Activity + View XML UI.

## Build & tooling commands

Use the Gradle wrapper (`./gradlew`) for everything; there is no separate lint/format script.

Build variants are a combination of **product flavor** (`play` / `fdroid`) x **build type** (`debug` / `release`):

```
./gradlew assemblePlayDebug        # Google Play flavor, debug build (has Firebase/FCM)
./gradlew assembleFdroidDebug      # F-Droid flavor, debug build (no Firebase, uses UnifiedPush/polling)
./gradlew assemblePlayRelease      # signed release build (needs release-keystore.jks + passwords)
./gradlew installFdroidDebug       # build and install on a connected device/emulator
```

There are no unit or instrumentation tests in this repo (no `test`/`androidTest` source sets) — `TESTING.md`
documents a **manual** QA checklist instead; run through it for any change touching subscriptions, delivery, or
notifications.

```
./gradlew ktlintCheck   # if configured — verify before assuming; not wired into a Gradle task by default here
```

Room schema JSON files live in `app/schemas/io.heckel.ntfy.db.Database/` and are auto-exported on build
(`ksp { arg("room.schemaLocation", ...) }` in `app/build.gradle`) — any Room entity/migration change regenerates
these; commit the resulting schema file alongside the migration.

## Remote build container (`android-build`)

There's a Debian Trixie LXC container reachable at `ssh forrest79@android-build` set up to build
`assemblePlayRelease` outside Windows. Setup performed 2026-08-11 (all steps idempotent — safe to re-run if the
container is rebuilt):

```
sudo apt-get install -y openjdk-21-jdk-headless unzip curl wget git ca-certificates rsync
```

JDK 21, not 17, because Trixie's repos only ship OpenJDK 21/25 — Gradle 9.2.1 / AGP 9.0.0 run fine on 21 and
still compile/target the project's Java 17 bytecode (`compileOptions`/`kotlin jvmTarget` in `app/build.gradle`
are independent of the JDK running Gradle itself).

Android SDK command-line tools, installed to `~/android-sdk`:

```
mkdir -p ~/android-sdk/cmdline-tools
curl -sSL -o /tmp/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip
unzip -q /tmp/cmdline-tools.zip -d ~/android-sdk/cmdline-tools
mv ~/android-sdk/cmdline-tools/cmdline-tools ~/android-sdk/cmdline-tools/latest

export ANDROID_HOME=~/android-sdk
yes | ~/android-sdk/cmdline-tools/latest/bin/sdkmanager --sdk_root=$ANDROID_HOME --licenses
~/android-sdk/cmdline-tools/latest/bin/sdkmanager --sdk_root=$ANDROID_HOME \
  'platform-tools' 'platforms;android-36' 'build-tools;36.0.0'   # matches compileSdkVersion/targetSdkVersion 36
```

`~/.bashrc` on the container has these appended:

```
export ANDROID_HOME=$HOME/android-sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/build-tools/36.0.0
```

**Project source**: Jakub shares the working directory into the container over a 9p Windows mount at
`/mnt/share/FORREST79/Apps/NtfyAndroid` (same files as the Windows checkout, including the gitignored
`app/google-services.json` and the committed `release-keystore.jks` — no secrets need copying separately). Two
things make that mount unsuitable to build from directly, so rsync a working copy to the container's local disk
first:

- **CRLF**: files checked out on Windows have CRLF line endings, which breaks `gradlew`'s `#!/bin/sh` shebang
  (`cannot execute: required file not found`). Fix only needed on `gradlew` itself.
- **9p I/O is slow** for Gradle/Kotlin's many-small-file compilation — a local copy builds much faster.
- **`local.properties`** on the share has `sdk.dir` pointing at the Windows SDK path
  (`C:\Users\Forrest79\AppData\Local\Android\Sdk`) — needed by the Windows side, so don't overwrite it in place;
  the local copy gets its own `local.properties` instead.

```
rsync -a --delete --exclude='.gradle/' --exclude='build/' --exclude='app/build/' --exclude='.idea/' \
  /mnt/share/FORREST79/Apps/NtfyAndroid/ ~/ntfy-android/
sed -i 's/\r$//' ~/ntfy-android/gradlew
chmod +x ~/ntfy-android/gradlew
echo 'sdk.dir=/home/forrest79/android-sdk' > ~/ntfy-android/local.properties
```

### Building an APK

Re-run the rsync step above before every build to pick up source changes made on Windows (it's cheap — only
copies changed files and excludes `.gradle`/`build` caches), then:

```
cd ~/ntfy-android
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
./gradlew assemblePlayRelease --no-daemon    # or any other variant, see app/build.gradle
```

`ANDROID_HOME`/`ANDROID_SDK_ROOT`/`PATH` come from `~/.bashrc` in an interactive shell; set `JAVA_HOME` and
source `~/.bashrc` (or export `ANDROID_HOME` too) explicitly in a non-interactive one (e.g. `ssh host "cmd"`,
which doesn't load `~/.bashrc`).

Output APKs land under `~/ntfy-android/app/build/outputs/apk/<flavor>/<buildType>/`, named
`app-<flavor>-<buildType>.apk` (release variants may also get a `-unsigned`/signed pair depending on config —
check the directory listing):

- `assemblePlayRelease` → `app/build/outputs/apk/play/release/app-play-release.apk` (the one that matters —
  signed with `release-keystore.jks`, self-installable, points at `https://ntfy.trmota.cz`)
- `assemblePlayDebug` → `app/build/outputs/apk/play/debug/app-play-debug.apk`
- `assembleFdroidRelease` → `app/build/outputs/apk/fdroid/release/app-fdroid-release.apk`
- `assembleFdroidDebug` → `app/build/outputs/apk/fdroid/debug/app-fdroid-debug.apk`

To pull an APK back to Windows, copy it onto the shared mount (`cp ... /mnt/share/FORREST79/Apps/NtfyAndroid/`)
or `scp` it directly from `android-build`.

Sanity-check a release build's signature and identity before install:

```
$ANDROID_HOME/build-tools/36.0.0/apksigner verify --print-certs <apk>
$ANDROID_HOME/build-tools/36.0.0/aapt2 dump badging <apk> | grep package:
```

Verified 2026-08-11: `assemblePlayRelease` produced `app-play-release.apk`, signed by `release-keystore.jks`
(cert `O=Trmota`), `applicationId=cz.trmota.ntfy`, version `1.25.2 (63)`.

## Product flavors

- **play**: `FIREBASE_AVAILABLE=true`, uses Firebase Cloud Messaging for push wake-up (source set `app/src/play/`).
- **fdroid**: `FIREBASE_AVAILABLE=false`, `PAYMENT_LINKS_AVAILABLE=true`, no Google Play services; relies on
  UnifiedPush and/or the persistent foreground service + polling for delivery (source set `app/src/fdroid/`).

Flavor-specific Firebase glue lives under `app/src/{play,fdroid}/java/io/heckel/ntfy/firebase/`. When editing
anything related to push delivery, check whether the change needs to happen in both flavor source sets or only
common code.

## Architecture

### Delivery pipeline (the core of the app)

Notifications reach the device through one of three paths, all converging on the same repository/notification
pipeline:

1. **Instant delivery (foreground service)** — `service/SubscriberService.kt` runs a persistent foreground
   service that holds one long-lived connection per base URL (topics on the same server are multiplexed).
   `service/SubscriberServiceManager.kt` starts/stops this service (always via a `WorkManager` one-off worker,
   never directly from a `BroadcastReceiver`, because receiver-spawned processes run at low priority and get
   killed — see the manager's doc comment). The service is intentionally over-engineered to survive Android's
   process-death behavior: it's `STICKY`, has boot/auto-restart `BroadcastReceiver`s, and (on the `play` flavor)
   gets kicked awake by an FCM keepalive.
2. **Firebase (play flavor only)** — FCM wakes the app for `ntfy.sh` subscriptions without instant delivery.
3. **Polling** — `work/PollWorker.kt` (periodic `WorkManager` job) and `msg/Poller.kt` (poll logic, groups
   messages by `sequenceId` and keeps only the latest per sequence) cover non-instant subscriptions.
4. **UnifiedPush (`up/` package)** — implements the [UnifiedPush](https://unifiedpush.org) distributor spec so
   third-party apps can receive push via ntfy instead of FCM. `up/Distributor.kt` /
   `up/BroadcastReceiver.kt` handle REGISTER/UNREGISTER/MESSAGE intents per the spec;
   `up/RaiseAppToForeground*.kt` briefly raises the target app's process importance before delivering a message.

Server connections themselves are abstracted behind `service/Connection.kt` (interface), implemented by
`service/WsConnection.kt` (WebSocket, adapted from the Gotify project) and `service/JsonConnection.kt`
(HTTP long-poll/JSON streaming) — `SubscriberService` picks whichever is supported per server.

Incoming messages flow: connection/poller → `msg/NotificationParser.kt` (parse wire format) →
`db/Repository.kt` (persist) → `msg/NotificationDispatcher.kt` (build & post Android notifications, respecting
per-subscription mute/priority/dedicated-channel settings) → optional `msg/DownloadAttachmentWorker.kt` /
`msg/DownloadIconWorker.kt` for attachments/icons.

### Data layer

- `db/Database.kt` — Room database. All entities (`Subscription`, `Notification`, `User`, trusted/client
  certificates, custom headers) and DAOs live in this one file. Schema is currently at `version = 18`; every
  schema change needs a new `MIGRATION_N_N+1` object added to the builder chain and a matching exported schema
  JSON in `app/schemas/`.
- `db/Repository.kt` — the single app-wide facade over the DB + in-memory connection state
  (`ConcurrentHashMap<baseUrl, ConnectionDetails>`) + `LiveData` for UI observation. Almost all business logic
  that isn't UI or transport-specific goes through here. `Repository.getInstance(context)` is the app-wide
  singleton accessor (see `app/Application.kt`).
- `backup/Backuper.kt` — export/import of subscriptions/settings (Android's data backup or manual
  export/import flow).

### UI layer

Classic Fragment/Activity with `RecyclerView` adapters and `ViewModel` + `LiveData` (no Jetpack Compose):
- `ui/MainActivity.kt` + `ui/MainAdapter.kt` + `ui/MainViewModel.kt` — subscription list.
- `ui/DetailActivity.kt` + `ui/DetailAdapter.kt` + `ui/DetailViewModel.kt` — notification list per subscription.
- `ui/AddFragment.kt`, `ui/PublishFragment.kt`, `ui/*SettingsFragment.kt`, `ui/*CertificateFragment.kt` —
  add-subscription flow, message publishing, and settings screens (server defaults, client/trusted certs, custom
  headers).
- `ui/ShareActivity.kt` — handles Android share-sheet intents into a publish action.
- Markdown message bodies render via `util/MarkwonFactory.kt` (Markwon + Glide for GIFs).

### Cross-cutting

- `util/HttpUtil.kt` / `util/CertUtil.kt` — OkHttp client setup, including support for user-provided trusted
  server certs and mTLS client certs (stored via the DB DAOs above).
- `util/Log.kt` — app-wide logging wrapper; use this rather than `android.util.Log` directly.
- All background/scheduled work (polling, attachment/icon downloads, service auto-restart, cleanup) goes through
  `WorkManager` (`work/` package + workers embedded in `msg/`/`service/`), not raw threads or `AlarmManager`,
  except where `AlarmManager`/`SCHEDULE_EXACT_ALARM` is explicitly needed to wake the websocket retry (see
  `AndroidManifest.xml` comments).

## Notes for changes

- `AndroidManifest.xml` has extensive comments explaining *why* each permission/service/receiver exists — read
  them before adding/removing entries; several exist purely to work around specific Android OEM battery-killing
  behavior.
- When touching anything in `service/` or `up/`, prefer testing manually per `TESTING.md` (multi-delivery-path
  correctness is hard to verify by reading code alone) — instant delivery, FCM, polling, and UnifiedPush all need
  independent verification.
- `release-keystore.jks` is committed to this fork intentionally (personal build); do not treat it as leaked
  secret material to be scrubbed.
