---
title: "Forter — Initializing the ELM327 and Reading the First PIDs"
date: 2026-09-27
categories: [Forter]
tags: [android, kotlin, elm327, obd2, bluetooth]
---

> This post was compiled by Claude.

## Goal

v0.1.3 opened an SPP socket, sent `ATZ`, and got `ELM327 v2.3` back. v0.1.4 goes one step
further: initialize the adapter properly, then read engine RPM and vehicle speed once and
show them on screen. Still one connect, one pass, one close. No polling loop yet.

## Is my adapter an ELM327?

Not exactly. The Vgate iCar Pro 2s is an **ELM327-compatible** adapter. ELM327 was a chip
(really, firmware on a PIC microcontroller) made by ELM Electronics, and its command set
became the de facto standard for cheap OBD2 adapters. Most adapters on the market speak
it, which is why apps like Torque Pro work with any of them. The `v2.3` string the adapter
returns is best read as "I support commands up to this version", not as proof of a
genuine chip. For Forter it does not matter: if it speaks ELM327, the code works.

## Where the commands are defined

ELM Electronics closed in 2022, so there is no official download page anymore. The full
datasheet (`ELM327DS.pdf`, 82 pages) survives in a few places, such as SparkFun's CDN and
the ELMduino repository. Only two sections matter for this project:

- **AT Command Summary**: one table listing every AT command
- **AT Command Descriptions**: one paragraph per command, alphabetical

PID definitions (what `010C` means and how to decode it) are not in the ELM datasheet.
They come from SAE J1979, which is paywalled, so the Wikipedia "OBD-II PIDs" page is the
practical reference.

I put both links and a command table in the repository `README.md`, so the answer to
"what did `ATS0` do again?" lives next to the code.

## How the commands are shaped

- `AT` + command: talks to the adapter itself. `AT` is fixed (it comes from old Hayes
  modem commands). What follows is usually a letter plus a value: `E0` turns echo off,
  `E1` turns it on. Case and spaces are ignored, so `AT E0` and `ATE0` are the same.
- Hex digits only: forwarded to the car's ECU. `01` is the mode (current data), the next
  two digits are the PID. `010C` is RPM, `010D` is speed.
- Every command ends with `\r`, and every response ends with the `>` prompt.

## The init sequence

| Command | Meaning |
|---------|---------|
| `ATZ` | Reset to defaults |
| `ATE0` | Echo off, so the command does not come back mixed into the response |
| `ATL0` | Linefeeds off |
| `ATS0` | Spaces off: `41 0C 1A F8` becomes `410C1AF8` |
| `ATH0` | Headers off: data only, no ECU address |
| `ATSP0` | Automatic protocol search |

`ATZ` comes first because I cannot know what state the adapter is in. Another app may
have changed its settings. Reset, then apply my own.

## Decoding: `Pid`

`Pid` turns a response string into a number. It knows nothing about Bluetooth, sockets,
or the UI.

```kotlin
object Pid {
    fun parseRpm(response: String): Int? {
        val data = dataBytes(response, "410C") ?: return null
        if (data.size < 2) return null
        return (data[0] * 256 + data[1]) / 4
    }

    fun parseSpeed(response: String): Int? {
        val data = dataBytes(response, "410D") ?: return null
        return data.firstOrNull()
    }

    // "410C1AF8" -> strip prefix, split into pairs, parse hex -> [0x1A, 0xF8]
    private fun dataBytes(response: String, prefix: String): List<Int>? {
        val line = response.lines()
            .map { it.trim() }
            .firstOrNull { it.startsWith(prefix) } ?: return null
        return line.removePrefix(prefix).chunked(2).mapNotNull { it.toIntOrNull(16) }
    }
}
```

- `4x` in the response means "answer to mode 0x", so `410C` is the reply to `010C`.
- RPM uses two bytes: `(A*256 + B) / 4`. `1AF8` is 6904, divided by 4 is 1726 rpm.
- Speed is one byte, already in km/h.
- It searches for the line that starts with the prefix instead of trusting the first
  line. The first OBD request after `ATSP0` can be preceded by `SEARCHING...`, and a
  hybrid may have more than one ECU answer.

It is an `object` because there is no state. The same input always gives the same output,
so there is no reason to create instances. In Java this would be a class full of `static`
methods, like `Math`. A side benefit: it can be unit-tested without a car.

## Talking to the adapter: `Elm327`

```kotlin
class Elm327(private val conn: SppConnection) {

    suspend fun init() {
        conn.send("ATZ", timeoutMs = 5000)   // reset can be slow
        for (cmd in listOf("ATE0", "ATL0", "ATS0", "ATH0", "ATSP0")) {
            val r = conn.send(cmd)
            check(r.contains("OK")) { "$cmd failed: $r" }
        }
    }

    suspend fun raw(command: String, timeoutMs: Long = 3000): String =
        conn.send(command, timeoutMs)

    suspend fun rpm(): Int? = Pid.parseRpm(conn.send("010C"))

    suspend fun speed(): Int? = Pid.parseSpeed(conn.send("010D"))
}
```

`check` throws if its condition is false, so a failed init step lands in the caller's
`catch` and shows up on screen.

The layering is now:

- `SppConnection` moves bytes
- `Elm327` decides which commands to send
- `Pid` interprets what comes back

### The primary constructor

`class Elm327(private val conn: SppConnection)` replaces all of this Java:

```java
public class Elm327 {
    private final SppConnection conn;

    public Elm327(SppConnection conn) {
        this.conn = conn;
    }
}
```

The parentheses after the class name are the primary constructor. Putting `val` in front
of a parameter makes it a property as well. Without `val`, the value is only available
during construction and `init()` could not use it.

## Wiring it up

```kotlin
scope.launch {
    val conn = SppConnection(context, device.address)
    try {
        status = "연결 중: ${device.name}"
        conn.connect()
        val elm = Elm327(conn)

        status = "초기화 중"
        elm.init()

        status = "프로토콜 탐색 중"
        val supported = elm.raw("0100", timeoutMs = 10000)
        val rpm = elm.rpm()
        val speed = elm.speed()

        status = "0100: $supported\nRPM: $rpm\n속도: $speed km/h"
    } catch (e: Exception) {
        status = "실패: ${e.message}"
    } finally {
        conn.close()
    }
}
```

`0100` goes first with a 10-second timeout. With `ATSP0`, the adapter searches for the
car's protocol on the first real OBD request, which can take a few seconds. Letting `0100`
absorb that wait keeps the RPM and speed requests fast. `0100` also returns a bitmap of
supported PIDs 01–20, which will be useful when adding more.

## Package name: `obd`, not `obd2`

OBD-I was manufacturer-specific and predates the mid-90s, so there is nothing to
distinguish OBD-II from in this app. The package also holds ELM327 command handling, not
just OBD-II PIDs, so the broader name fits. (Kotlin package names cannot contain hyphens,
so `obd-ii` was never an option.)

## Mistakes along the way

- A parameter typed as `respnse` made every use of `response` in the body unresolved.
- `toIntOrNull(1)` instead of `toIntOrNull(16)`. Half-typed, but a wrong radix would
  have compiled and silently returned `null` for everything.
- I first pasted only the `init()` function without the surrounding class, so `conn`
  was unresolved. The snippet was a fragment, not a whole file.
- Twice, keystrokes meant for the terminal went into the editor: a stray `d` after
  `testImplementation(libs.junit)` and a `git` inside `dependencies {`. The fix is simple:
  click the terminal before typing.
- Opening `README.md` made Android Studio create `.idea/markdown.xml`, which showed up in
  `git status`. Checking status before committing caught it. Android Studio's commit
  dialog (`Ctrl+K`) is easier here: uncheck the file and commit.

## Result at the cafe

I wrote this session at a cafe, away from the car. The app shows `Forter v0.1.4-...`.
With the adapter unpowered, tapping `Android-Vlink`
goes to `연결 중` and then:

```text
실패: read failed, socket might closed or timeout, read ret: -1
```

Same as v0.1.3: the failure is caught and the app stays responsive. This fails at
`connect()`, so the new init and PID code has not run yet. That only happens in the car.

## Next

- Test in the car with the engine running. The Forte is a hybrid, so RPM can read `0`
  at idle when the engine stops. That is not a bug.
- Record the actual `0100` response in the README.
- Tag `v0.1.4` on the commit that worked in the car.
- v0.1.5: a 1-second polling loop, counting timeouts as failures so a dropped connection
  is detected instead of stalling silently.
