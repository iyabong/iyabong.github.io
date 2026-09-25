---
title: "Forter — Starting Over in Android Studio"
date: 2026-09-24
categories: [Forter]
tags: [android, android-studio, git, gradle, bluetooth]
---

> This post was drafted by Claude from my work session, then reviewed by me.

## Why start over

It has been months since I started Forter, and I had lost track of the old code and how it
flowed. Rather than re-reading it piece by piece, I decided to start from an empty project
and rebuild it by hand — understanding the structure as I go to make it mine again.

## Archive the prototype

The old `dev` was kept as a branch before anything was deleted:

```text
archive/prototype-v0
```

`/` separates a category from a name, `-` separates words. `archive/` rather than `backup/`,
since it is kept for reference, not for restoring.

## New project

- Template: **Empty Activity** (Compose)
- Package: `com.iyabong.forter`
- Minimum SDK: **API 31** — where `BLUETOOTH_CONNECT` begins, so no legacy permission code
- Location: `D:\git\forter` — no Korean characters or spaces in the path

Built on the laptop and installed over USB debugging. First result: `Hello Android!`

## Version on screen

In `app/build.gradle.kts`:

```kotlin
buildFeatures {
    compose = true
    buildConfig = true   // off by default in recent AGP
}
```

Then `BuildConfig.VERSION_NAME` can be shown on screen. In the car, that answers
"is this the build with the fix?"

`BuildConfig` is generated under `app/build/` on every build. Change values in
`build.gradle.kts`, never in the generated file.

### From Git

Instead of bumping by hand, the version comes from Git — per commit, not per build, so the
same code always shows the same number:

```kotlin
val gitCommitCount = providers.exec {
    commandLine("git", "rev-list", "--count", "HEAD")
}.standardOutput.asText.get().trim().toInt()

val gitHash = providers.exec {
    commandLine("git", "rev-parse", "--short", "HEAD")
}.standardOutput.asText.get().trim()

android {
    defaultConfig {
        versionCode = gitCommitCount
        versionName = "0.1.0-$gitHash"
    }
}
```

- `rev-list --count HEAD` — number of commits
- `rev-parse --short HEAD` — current commit hash, 7 characters

The phone shows `v0.1.0-f57c5bd`. The `0.1.0` part stays manual.

## Replace dev without a force push

The new project went onto `dev` as one commit on top of the old history:

```bash
git init
git remote add origin https://github.com/iyabong/forter.git
git fetch origin
git checkout -b dev
git reset origin/dev      # move the branch only; files untouched
git add -A
git commit -m "Fresh start: Hello Forter v0.1.0"
git tag v0.1.0
git push -u origin dev
git push origin v0.1.0
```

```text
   45f505a..f57c5bd  dev -> dev
```

A fast-forward. The prototype history stays below the new start. `dev` is now the GitHub
default branch.

## Bluetooth permission

Forter connects to the adapter by MAC address with no scan, so only `BLUETOOTH_CONNECT` is
needed.

```xml
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

```kotlin
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { result -> granted = result }

Button(onClick = { launcher.launch(Manifest.permission.BLUETOOTH_CONNECT) }) {
    Text("권한 요청")
}
```

One trap: there are three classes named `Manifest`. Alt+Enter first offered
`java.util.jar.Manifest`. The right one is `android.Manifest`. Typing the code by hand is
what surfaced this.

Also: denying twice — including turning the permission off in Settings — makes the denial
permanent. The request button then does nothing. Handling that is left for later.

## Next

- List bonded devices and find the adapter's MAC address
- Connect over RFCOMM
- Move the connection into a Foreground Service
