---
title: "Forter — Starting Over in Android Studio"
date: 2026-09-24
categories: [Forter]
tags: [android, android-studio, git, gradle, versioning]
---

## Why start over

A review of the prototype on `dev` turned up structural problems, not bugs:

- The collection thread lived inside `MainActivity`. A screen rotation or an automatic
  dark-mode switch destroyed the Activity and ended the session.
- No Foreground Service. The manifest declared the permissions, but there was no `<service>`.
  With a navigation app in front, the process could be frozen — a silent stall with no
  exception and no error count.
- No reconnect. Once the RFCOMM socket died, the loop logged failures until stopped by hand.
- CSV rows were held in memory and written once, in `finally`. A killed process lost the
  whole drive.

Fixing these meant moving the connection into a service and rewriting the loop around it.
Instead of patching, I decided to start from an empty project, write every line by hand,
and build on the laptop.

The [previous post](/posts/forter-build-apk/) set up phone-only builds on GitHub Actions.
Its own conclusion was that writing new classes is faster on a laptop. A rewrite is all new
classes, so the workflow was dropped.

## Archiving the prototype

Before deleting anything, the old `dev` was preserved as a branch, created on GitHub from `dev`:

```text
archive/prototype-v0
```

Naming conventions:

- `/` separates a category from a name. Git stores branches under `.git/refs/heads/`, so
  `archive/prototype-v0` is just a subdirectory. It is still one name, not a hierarchy.
- `-` separates words within a name. `_` works but is uncommon.
- `archive/` means finished work kept for reference. `backup/` implies a copy you might
  restore from. Nothing here will be restored, so `archive/`.

A consequence of the directory layout: if a branch named `archive` existed,
`archive/prototype-v0` could not be created. A path cannot be a file and a directory at once.

### Where did a branch come from?

Git does not record it. A branch is a pointer to a commit and nothing more. To check, compare
commits — the new branch's head was `45f505a`, the same as `dev`. Or use the compare view:

```text
github.com/iyabong/forter/compare/dev...archive/prototype-v0
```

"There isn't anything to compare" means the two point to the same commit.

The "12 commits ahead of, 1 commit behind main" banner is not origin information. GitHub
always compares against the default branch.

## New project

| Field | Value |
|-------|-------|
| Template | Empty Activity (Compose) |
| Package | `com.iyabong.forter` |
| Minimum SDK | API 31 |
| Build config | Kotlin DSL |
| Location | `D:\git\forter` |

- **Empty Activity** is the Compose template. **Empty Views Activity**, right next to it, is
  the XML one.
- **API 31** is where the Bluetooth permission model changed to `BLUETOOTH_SCAN` /
  `BLUETOOTH_CONNECT`. Starting there removes the legacy permission branches.
- A path with no Korean characters or spaces avoids occasional Gradle problems with the
  default `C:\Users\<name>\...` location.

The first Gradle sync took several minutes, downloading dependencies. Later syncs are
incremental.

## Connecting the phone

**USB debugging**: Settings → About phone → Software information → tap Build number seven
times → Developer options → USB debugging.

One trap: TalkBack has its own screen titled *Developer settings*. It is not the system
Developer options.

"Charging phone only" in USB settings does not block ADB. The device showed up as
`samsung SM-S938N` without changing it.

**Wireless debugging** works when the laptop and phone share a Wi-Fi network. Pair once
from the device dropdown with *Pair Devices Using Wi-Fi* and a QR code.

Security notes:

- ADB can install apps, capture the screen and inject input.
- Pairing is required, the connection is encrypted, and wireless debugging turns off when
  the phone leaves the network.
- Leave it off on public Wi-Fi, where client isolation often blocks it anyway.
- Some Korean banking apps refuse to start while developer options are on.

In the car there is no shared Wi-Fi, so testing there will be over USB.

The first build took a few minutes. The phone showed `Hello Android!`.

## Version

Two `build.gradle.kts` files:

- **Project: Forter** declares plugins for all modules with `apply false`. It applies nothing.
- **Module :app** holds the app's actual configuration. Similar to a parent and child
  `pom.xml` in Maven.

In the module file:

```kotlin
defaultConfig {
    versionCode = 1
    versionName = "0.1.0"
}

buildFeatures {
    compose = true
    buildConfig = true
}
```

- `versionCode` is the integer the system compares on update. It must increase.
- `versionName` is the string people read. The format is free, but SemVer's leading `0`
  already says "pre-release". The prototype never set a version (it kept the default `"1.0"`),
  so this is effectively the first one.
- `buildConfig = true` generates a `BuildConfig` class at build time. Recent AGP versions
  turn it off by default for build speed.

Generated class:

```kotlin
object BuildConfig {
    const val DEBUG = true
    const val APPLICATION_ID = "com.iyabong.forter"
    const val BUILD_TYPE = "debug"
    const val VERSION_CODE = 1
    const val VERSION_NAME = "0.1.0"
}
```

After sync it appears under `java (generated)` in the project tree.

In `MainActivity.kt`:

```kotlin
Greeting(
    name = "Forter",
    modifier = Modifier.padding(innerPadding)
)

// ...

Text(
    text = "Hello $name!\n${BuildConfig.VERSION_NAME}",
    modifier = modifier
)
```

The phone now shows:

```text
Hello Forter!
0.1.0
```

The version is set in one place and read from there. In the car, the screen answers
"is this the build with the fix?"

## Replacing dev without a force push

The new project was not a Git repository yet. The goal was to put it on `dev` while keeping
`dev`'s history — one commit on top that replaces everything.

```bash
git init
git remote add origin https://github.com/iyabong/forter.git
git fetch origin
git checkout -b dev
git reset origin/dev
```

`git reset origin/dev` moves the local `dev` to the remote `dev`'s last commit. The working
tree is untouched. Git now sees the new project as changes against the old one:

```text
D    .github/workflows/build.yml
D    app/src/main/java/com/iyabong/forter/BleScanner.kt
D    app/src/main/java/com/iyabong/forter/ClassicBtScanner.kt
D    app/src/main/java/com/iyabong/forter/ObdSppConnection.kt
M    app/build.gradle.kts
M    app/src/main/java/com/iyabong/forter/MainActivity.kt
...
```

Before committing, check that `local.properties` and `build/` are not in the list. The
template's `.gitignore` already excludes them.

```bash
git add -A
git commit -m "Fresh start: Hello Forter v0.1.0"
git tag v0.1.0
git push -u origin dev
git push origin v0.1.0
```

- `commit` and `tag` are local.
- `push -u origin dev` sends the branch and remembers `origin/dev` as its upstream, so the
  next push is just `git push`.
- Tags are pushed separately.

Result:

```text
   45f505a..f57c5bd  dev -> dev
 * [new tag]         v0.1.0 -> v0.1.0
```

`45f505a..f57c5bd` is a fast-forward. The prototype history is still on `dev`, directly
below the new start.

### Two warnings that don't matter

`LF will be replaced by CRLF the next time Git touches it` — Git for Windows stores LF in the
repository and checks files out as CRLF. It is a notice, not an error.

In the Android Studio terminal, PowerShell showed command arguments on a black background.
That is PSReadLine's highlighting clashing with a light theme. Switching the terminal to Git
Bash (Settings → Tools → Terminal → Shell path) removed it.

## Next

- Switch the GitHub default branch to `dev`
- Request Bluetooth permissions and connect to the adapter's saved MAC address, without a
  scan screen
- Put the connection and polling loop inside a Foreground Service from the start
