---
title: "Forter — Starting Over in Android Studio"
date: 2026-09-24
categories: [Forter]
tags: [android, android-studio, git, gradle, versioning, bluetooth, permissions]
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

In `MainActivity.kt`, the greeting now reads the version:

```kotlin
Text(
    text = "Hello $name!\nv${BuildConfig.VERSION_NAME}",
    modifier = modifier
)
```

```text
Hello Forter!
v0.1.0
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

`LF will be replaced by CRLF the next time Git touches it` appeared on `git add`. Git for
Windows stores LF in the repository and checks files out as CRLF. It is a notice, not an error.

## Version from Git

A hand-edited `versionCode` gets forgotten. Two ways to automate it:

- **Per build** — every ▶ bumps the number. It grows meaninglessly (dozens of builds a day),
  and the configuration changes on every build, so Gradle's caches stop helping.
- **Per commit** — the same code always carries the same number. The screen identifies the
  exact source.

Per commit, then. At the top of the module `build.gradle.kts`, above `android { }`:

```kotlin
val gitCommitCount = providers.exec {
    commandLine("git", "rev-list", "--count", "HEAD")
}.standardOutput.asText.get().trim().toInt()

val gitHash = providers.exec {
    commandLine("git", "rev-parse", "--short", "HEAD")
}.standardOutput.asText.get().trim()
```

```kotlin
defaultConfig {
    versionCode = gitCommitCount
    versionName = "0.1.0-$gitHash"
}
```

This is Kotlin, not Groovy — the file is a Kotlin script (`.kts`). The Groovy equivalent
would use `def` and single quotes.

- `providers.exec { }` runs an external command. It is configuration-cache compatible, unlike
  calling a process directly.
- `commandLine(...)` is the command, exactly as typed in a terminal.
- `.get()` is where it actually runs. A Gradle `Provider` holds a value to be computed later.
- `.trim()` drops the trailing newline.

Gradle has to find `git` on the `PATH`. Git for Windows adds it by default.

The two commands on their own:

```bash
$ git rev-parse --short HEAD
f57c5bd

$ git rev-list --count HEAD
15
```

- `rev-parse` turns any revision name — `HEAD`, `dev`, `v0.1.0`, `HEAD~1` — into a commit
  hash. `--short` abbreviates it to seven characters.
- `rev-list` walks back from `HEAD` through parent commits. `--count` prints only the number.
  15 is the prototype's 14 commits plus the fresh start.

The phone showed `v0.1.0-f57c5bd`. One catch: uncommitted changes still carry the previous
commit's hash. After committing this change, the next build showed `v0.1.0-7a6e0e9`.

`0.1.0` stays manual. Deciding that a feature is complete enough for `0.2.0` is a human call.

## Generated sources

After sync, `BuildConfig` appears under `java (generated)`:

```java
public final class BuildConfig {
  public static final boolean DEBUG = Boolean.parseBoolean("true");
  public static final String APPLICATION_ID = "com.iyabong.forter";
  public static final String BUILD_TYPE = "debug";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "0.1.0";
}
```

Android Studio warns *Generated source files should not be edited*. The real path is
`app/build/generated/source/buildConfig/debug/...` — inside `build/`, regenerated on every
build, never committed. Change the values in `build.gradle.kts` instead.

It is Java because AGP generates Java. Kotlin reads it directly.

## Project layout

The **Android** view groups files by role rather than showing the disk layout:

```text
app/
├─ manifests/AndroidManifest.xml   permissions, activities, services
├─ kotlin+java/
│   ├─ com.iyabong.forter/         source
│   │   ├─ ui.theme/               Color.kt, Type.kt, Theme.kt
│   │   └─ MainActivity.kt
│   ├─ com.iyabong.forter          (androidTest) runs on the device
│   └─ com.iyabong.forter          (test) runs on the JVM
├─ java (generated)                BuildConfig — do not edit
└─ res/                            icons, strings, XML themes

Gradle Scripts/
├─ build.gradle.kts (Project)      shared plugin declarations
├─ build.gradle.kts (Module :app)  SDK levels, version, dependencies
├─ settings.gradle.kts             project name, modules, repositories
├─ libs.versions.toml              version catalog
├─ gradle.properties               Gradle JVM options
├─ gradle-wrapper.properties       Gradle version
├─ local.properties                local SDK path — not committed
└─ proguard-rules.pro              R8 rules (unused for now)
```

For a Spring developer:

| Android | Spring |
|---|---|
| `kotlin+java/` | `src/main/java` |
| `res/` | `src/main/resources` |
| `AndroidManifest.xml` | app configuration (like the old `web.xml`) |
| `build.gradle.kts (Module)` | module `pom.xml` |
| `libs.versions.toml` | parent pom's `dependencyManagement` |

**Android** for daily work, **Project** when the real path matters — `.gitignore`, root
files, `build/`. **Packages** is rarely useful.

## IDE settings

A few things worth changing on a modest laptop:

| What | Where |
|---|---|
| UI font (menus, project tree) | Settings → Appearance & Behavior → Appearance → Use custom font |
| Whole-IDE zoom | same screen → Zoom, or View → Appearance → Zoom IDE |
| Parameter name hints (`value =`, `contract =`) | Settings → Editor → Inlay Hints → Kotlin → Parameter names |
| Documentation popup font | ⋮ in the popup → Adjust Font Size |
| Documentation popup on hover | Settings → Editor → Code Editing → Quick Documentation → Show on mouse move (Ctrl+Q still works) |

Android Studio also flagged that Microsoft Defender's real-time scanning slows builds, and
offers *Exclude folders* for the project and Gradle caches.

### Terminal

PowerShell in the IDE terminal showed command arguments on a black background. Android
Studio eventually explained it on startup: PSReadLine 2.0.0 is outdated.

```powershell
Install-Module PSReadLine -MinimumVersion 2.0.3 -Scope CurrentUser -Force
```

Answer `Y` to the NuGet provider prompt and `A` to the PSGallery prompt, then open a new
terminal tab — the old tab keeps the old module loaded. The black background was gone, and
profile load time dropped from 21,146 ms to 3,782 ms.

Git Bash is still the default here. Guides, documentation and my own blog posts use bash
syntax, and the same commands work on a Linux server or a Mac:

```text
Settings → Tools → Terminal → Shell path
C:\Program Files\Git\bin\bash.exe
```

## Bluetooth permission

Since Android 12, connecting to a Bluetooth device requires a runtime permission. Two parts:
declare it in the manifest, then ask the user.

Forter will connect to the adapter by its MAC address with no discovery, so
`BLUETOOTH_CONNECT` is enough. `BLUETOOTH_SCAN` is not needed.

`AndroidManifest.xml`, above `<application>`:

```xml
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

`MainActivity.kt`:

```kotlin
@Composable
fun PermissionScreen(modifier: Modifier = Modifier) {
    val context = LocalContext.current
    var granted by remember {
        mutableStateOf(
            ContextCompat.checkSelfPermission(
                context, Manifest.permission.BLUETOOTH_CONNECT
            ) == PackageManager.PERMISSION_GRANTED
        )
    }
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { result -> granted = result }

    Column(modifier) {
        Text("Hello Forter!\nv${BuildConfig.VERSION_NAME}")
        Text(if (granted) "블루투스 권한: 허용됨" else "블루투스 권한: 없음")
        if (!granted) {
            Button(onClick = {
                launcher.launch(Manifest.permission.BLUETOOTH_CONNECT)
            }) {
                Text("권한 요청")
            }
        }
    }
}
```

`setContent` calls `PermissionScreen(modifier = Modifier.padding(innerPadding))` in place of
the template's `Greeting`.

### Three classes named Manifest

`Unresolved reference 'permission'` on `Manifest.permission.BLUETOOTH_CONNECT` means the
wrong `Manifest` is in scope. There are three:

| Class | What it is |
|---|---|
| `android.Manifest` | Android's permission constants — the right one |
| `java.util.jar.Manifest` | JAR file metadata. Alt+Enter offered it first |
| `com.iyabong.forter.Manifest` | Generated for the app. Same package, so it wins with no import |

An explicit `import android.Manifest` beats the same-package class.

The other import that trips people up: `var granted by remember { }` needs
`androidx.compose.runtime.getValue` and `setValue`. Without them `by` shows a red underline,
and Alt+Enter does not always offer the fix.

Typing the code by hand instead of pasting it is what surfaced all of this — two typos
(`BLUTOOTH`, `launncher`) and the wrong import. Pasted code would have compiled, and I would
not have learned that three `Manifest` classes exist.

### Result

```text
Hello Forter!
v0.1.0-7a6e0e9
블루투스 권한: 없음
[ 권한 요청 ]
```

The system dialog reads *Allow Forter to find, connect to, and determine the relative
position of nearby devices?* That sentence describes the whole Nearby devices group. The app
only receives what the manifest declares — `BLUETOOTH_CONNECT`.

After **Allow**, the button disappears and the status reads `허용됨`. On later launches it
shows `허용됨` immediately.

To test again: long-press the app icon → App info → Permissions → Nearby devices → Don't allow.

## Default branch

All work happens on `dev`, so it is now the GitHub default branch: Settings → General →
Default branch → ⇄ → `dev` → Update.

- The repository page opens on `dev` instead of the old code on `main`
- A fresh clone checks out `dev`
- Pull requests target `dev` by default

`main` still holds the prototype. It stays for now as a future home for stable releases.
Nothing is lost if it goes — the same code is on `archive/prototype-v0`.

## Next

- List bonded devices on screen — `BLUETOOTH_CONNECT` is enough to read them, no scan —
  and find the adapter's name and MAC address
- Connect to that address over RFCOMM
- Put the connection and polling loop inside a Foreground Service from the start
