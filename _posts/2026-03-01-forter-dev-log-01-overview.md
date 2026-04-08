---
title: "Forter Dev Log (1) - Project Overview"
date: 2026-03-01
categories: [Forter]
tags: [Android, Kotlin, OBD2, BLE, ELM327, Cloudflare Workers, Upstash Redis]
---

## Background

I had been using **Torque Pro** combined with the Vgate iCar Pro 2s OBD2 scanner to monitor my vehicle's condition.

Torque Pro was released in 2012. At the time, its web server upload feature was sufficient — REST APIs, JSON payloads, and auth tokens were not yet common in mobile app design. However, by today's standards, the integration capabilities are significantly limited.

**OBD2 / BLE Control**
- Torque Pro delegates BLE handling entirely to the ELM327 adapter. BLE connection state, reconnection logic, and disconnect handling cannot be managed at the app level.
- OBD2 connect/disconnect events can be broadcast as intents, but connection timing cannot be controlled.
- The app occasionally crashes on BLE reconnection with no way to handle it.

**API Call Limitations**
- The web server upload feature only supports simple GET requests.
- Setting request headers, POST body, or auth tokens is not supported.
- Transmission interval, retry logic, and filtering conditions cannot be customized.
- Initialization requests fire even when OBD is not connected, generating unnecessary API calls.

To address these constraints, I decided to build a native Android app that gives full control over BLE management and API communication.

**Forter** is an Android native app that replaces Torque Pro and streams collected vehicle data to a self-hosted backend in real time. The name is a portmanteau of **Forte + Porter**, inspired by the target vehicle, KIA Forte.

---

## Target Vehicle

| Field | Value |
|-------|-------|
| Manufacturer | KIA |
| Model | Forte LPi Hybrid |
| Year | 2012 |
| OBD2 Scanner | Vgate iCar Pro 2s (BLE) |

---

## System Architecture

```text
[KIA Forte LPi Hybrid]
        │
   OBD2 Port (SAE J1962)
        │
[Vgate iCar Pro 2s]
        │  BLE (ELM327 AT Command)
[Forter Android App]
        │  HTTP
[Cloudflare Workers]
        │
[Upstash Redis]
```

---

## Tech Stack

| Layer | Technology | Note |
|-------|-----------|------|
| App | Android Native (Kotlin) | Jetpack Compose |
| BLE | Nordic BLE Library 2.7.4 | |
| OBD2 | ELM327 AT Command | SAE J1979 |
| Backend | Cloudflare Workers | |
| DB | Upstash Redis | |
| Automation | MacroDroid | Auto-launch on BT connect |

Android Native was chosen for the stability of BLE Foreground Service. Flutter and React Native were considered but excluded due to limitations in background BLE handling.

---

## Development Environment

| Field | Value |
|-------|-------|
| IDE | Android Studio Panda 2025.3.1 |
| Language | Kotlin |
| Minimum SDK | API 31 (Android 12.0) |
| Test Device | Galaxy S26 |

API 31 was set as the minimum SDK because Android 12 introduced a revised BLE permission model (`BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`).

---

## Source Control

- Repository: [https://github.com/iyabong/forter](https://github.com/iyabong/forter)
- Branch strategy: `main` (production) / `dev` (development)

---

## Progress

| Task | Status |
|------|--------|
| Cloudflare Workers + Upstash Redis backend | ✅ Done |
| MacroDroid BT connection automation | ✅ Done |
| Android Studio project setup | ✅ Done |
| BLE permissions (AndroidManifest.xml) | ✅ Done |
| Nordic BLE library integration | ✅ Done |
| BLE scan & Vgate connection | 🔄 In Progress |
| ELM327 command communication | ⬜ Planned |
| Data parsing & Worker transmission | ⬜ Planned |

---

## Next Steps

Starting with BLE scan implementation. The goal is to discover the Vgate iCar Pro 2s device, establish a GATT connection, and send the ELM327 initialization command sequence.
