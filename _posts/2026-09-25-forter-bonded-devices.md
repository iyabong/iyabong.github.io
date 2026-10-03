---
title: "Forter — Listing Bonded Bluetooth Devices"
date: 2026-09-25
categories: [Forter]
tags: [android, kotlin, jetpack-compose, bluetooth, signing]
---

> This post was drafted by Claude from my work session, then reviewed by me.

## Goal

v0.1.2: show the phone's bonded Bluetooth devices and let me pick the OBD2 adapter.
Reading the bonded list is not a scan, so `BLUETOOTH_CONNECT` from v0.1.1 is all it needs.

## Package layout

Before writing code, I settled the layout: split by feature, grow folders only when needed.

```text
com.iyabong.forter
├── MainActivity.kt
├── Permissions.kt          # all runtime permissions in one place
├── bluetooth/              # bonded devices, SPP socket
├── obd/                    # ELM327 commands, PID parsing
├── trip/                   # polling loop, foreground service
├── data/                   # Room, CSV, upload
└── ui/
    ├── theme/
    └── DeviceSelectScreen.kt
```

The data flows one way: `bluetooth → obd → trip → data`. A few naming rules came with it:

- Full words for packages: `bluetooth`, not `bt`. `obd` stays because it is the real name.
- No prefix the package already gives: `bluetooth/BondedDevices.kt`, not `BluetoothBondedDevices.kt`.
- Screens keep full names (`DeviceSelectScreen`), because they are called side by side where
  the package is not visible. `SelectScreen` alone says nothing.
- `Permissions.kt` sits at the root, not under `bluetooth/`. Location and notification
  permissions will join it later.

## Permissions

```kotlin
object Permissions {
    val required: Array<String> = arrayOf(
        Manifest.permission.BLUETOOTH_CONNECT,
    )

    fun isGranted(context: Context, permission: String): Boolean =
        ContextCompat.checkSelfPermission(context, permission) ==
            PackageManager.PERMISSION_GRANTED

    fun missing(context: Context): List<String> =
        required.filterNot { isGranted(context, it) }
}
```

A Kotlin `object` compiles to a Java singleton (private constructor, static `INSTANCE`).
Permissions are plain `String` constants on Android; there is no `Permission` type.

## Reading bonded devices

The result has three states, expressed as a sealed interface so the screen cannot miss one.

```kotlin
sealed interface BondedResult {
    data object NoPermission : BondedResult
    data object BluetoothOff : BondedResult
    data class Devices(val list: List<BondedDevice>) : BondedResult
}

@SuppressLint("MissingPermission") // checked on the first line
fun loadBondedDevices(context: Context): BondedResult {
    if (!Permissions.isGranted(context, Manifest.permission.BLUETOOTH_CONNECT)) {
        return BondedResult.NoPermission
    }
    val adapter = context.getSystemService(BluetoothManager::class.java)?.adapter
    if (adapter == null || !adapter.isEnabled) return BondedResult.BluetoothOff

    return BondedResult.Devices(
        adapter.bondedDevices
            .map { BondedDevice(it.name ?: "(no name)", it.address, it.type) }
            .sortedBy { it.name }
    )
}
```

The file is `BondedDevices.kt`, yet no class has that name. Kotlin allows it; top-level
functions compile into a class called `BondedDevicesKt`.

## Screen

`DeviceSelectScreen` only takes state and callbacks. `MainActivity` owns the state and
reloads it after the permission dialog or a refresh.

```kotlin
var result by remember { mutableStateOf(loadBondedDevices(context)) }

val permissionLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { result = loadBondedDevices(context) }
```

Having used Vue and React, the pattern was recognizable: props down, events up, and
`remember` / `LaunchedEffect` map closely to `useState` / `useEffect`.

One trap: `Modifier.clickable({ onSelect(device) })` does not compile. Inside the
parentheses, the lambda goes to the first parameter. Outside, as a trailing lambda, it goes
to the last one, `onClick`:

```kotlin
modifier = Modifier.clickable { onSelect(device) }
```

## Result

The adapter showed up as `Android-Vlink`, type **Classic**. Tapping it logged the
selection. The list also has `FORTE` — the car's head unit, not the adapter, so the two are
easy to confuse.

## Signing keys

This explained an old annoyance. Every APK built by GitHub Actions had to be uninstalled
before the next one would install. Each runner is a fresh VM, so it generated a new debug
key every time, and Android refuses an update signed by a different key.

I backed up my laptop's `debug.keystore` to a private `vault` repository. If I bring CI back,
the key goes into Actions secrets so every build is signed the same way.

A small habit too: `versionName` includes the commit hash, so I build after committing.
Otherwise the app shows the previous hash.

## Next

- Open an RFCOMM socket to the selected MAC address
- Send `ATZ` and read the ELM327 response
