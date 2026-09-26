---
title: "Forter — Opening an SPP Connection to the Adapter"
date: 2026-09-26
categories: [Forter]
tags: [android, kotlin, jetpack-compose, bluetooth, coroutines, elm327]
---

> This post was compiled by Claude.

## Goal

v0.1.3: tap the adapter in the bonded list, open an RFCOMM socket, send `ATZ`, and show
the ELM327 response on screen. One connect, one command, one close. No service, no
reconnect, no polling yet.

## Reading `MainActivity` first

Before adding code, I stopped to understand when each line actually runs. The file reads
top to bottom, but the lines do not run at the same time.

- `setContent { }` runs inside `onCreate`, but it only stores the lambda. The lambda runs
  later, when the view is attached to the window (after `onResume`).
- `remember { }` runs on the first composition only, then returns the stored value.
- `LaunchedEffect(Unit) { }` is scheduled during composition and runs once after the first
  frame.
- Event lambdas (`onRefresh`, `onSelect`) run when the user does something.
- Changing a `mutableStateOf` value re-runs the parts that read it.

`@Composable` marks a function the Compose compiler rewrites so the runtime can track where
it was called, remember values, and re-run it. `ForterTheme`, `Column`, and `Text` are all
functions, not classes. The capital letter is a Compose naming convention.

That was enough. The goal of Forter is collecting car data, not mastering Android, so I
decided to learn Compose only as far as the app needs and spend the depth on the
connection, the service, and the data.

## `SppConnection`

```kotlin
class SppConnection(
    private val context: Context,
    private val address: String,
) {
    private var socket: BluetoothSocket? = null

    @SuppressLint("MissingPermission")
    suspend fun connect() = withContext(Dispatchers.IO) {
        val adapter = context.getSystemService(BluetoothManager::class.java).adapter
        val s = adapter.getRemoteDevice(address).createRfcommSocketToServiceRecord(SPP_UUID)
        s.connect()
        socket = s
    }

    suspend fun send(command: String, timeoutMs: Long = 3000): String = withContext(Dispatchers.IO) {
        val s = socket ?: error("Not connected")
        s.outputStream.write("$command\r".toByteArray())
        s.outputStream.flush()

        val input = s.inputStream
        val buffer = StringBuilder()
        withTimeout(timeoutMs) {
            while (true) {
                if (input.available() > 0) {
                    val c = input.read().toChar()
                    if (c == '>') break
                    buffer.append(c)
                } else {
                    delay(10)
                }
            }
        }
        buffer.toString().replace("\r", "\n").trim()
    }

    fun close() {
        runCatching { socket?.close() }
        socket = null
    }

    companion object {
        private val SPP_UUID: UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
    }
}
```

- Socket connect and read block, so both run on `Dispatchers.IO`. On the main thread the
  UI would freeze.
- `00001101-...` is the standard Serial Port Profile UUID.
- ELM327 ends every response with `>`. Read until it arrives, or give up after 3 seconds.

## Wiring it to the screen

`suspend` functions cannot be called from a plain click lambda, so `MainActivity` gets a
coroutine scope and a `status` state.

```kotlin
val scope = rememberCoroutineScope()
var status by remember { mutableStateOf("기기를 선택하세요") }

onSelect = { device ->
    scope.launch {
        val conn = SppConnection(context, device.address)
        try {
            status = "연결 중: ${device.name}"
            conn.connect()
            val response = conn.send("ATZ")
            status = "응답:\n$response"
        } catch (e: Exception) {
            status = "실패: ${e.message}"
        } finally {
            conn.close()
        }
    }
}
```

## Mistakes along the way

- A missing `BluetoothManager` import failed the build. The editor showed no error in the
  file itself; the build output did.
- I typed `"/r"` instead of `"\r"`. It compiles, but the response would never split into
  lines.
- A mismatched brace put `companion object` inside `onCreate`.

## Result at home

With the adapter powered off, tapping it shows:

```text
실패: read failed, socket might closed or timeout, read ret: -1
```

That is the expected failure. The exception is caught, the app stays responsive, and the
reason is on screen. The version reads `v0.1.3-e45c6ea`.

## Next

- Test in the car with ACC on and other OBD apps closed. Expect `ELM327 v1.5`.
- Tag `v0.1.3` once the car test passes.
- Move the connection into a Foreground Service.
