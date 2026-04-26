# Beacon Firmware — ESP32-C3 iBeacon with Vibration Wake

<!-- Add prototype photo to docs/images/prototype-project-cover.jpg -->

**Version**: 1.1  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino  
**BLE Library**: NimBLE-Arduino 2.1.0  
**Status**: Production Ready

---

## Overview

Ultra-low-power iBeacon firmware for the ESP32-C3. The device spends nearly all its time in deep sleep (~5 µA). A SW-420 vibration sensor on GPIO4 wakes the device, which then broadcasts a standard Apple iBeacon advertisement for a configurable duration before returning to sleep.

A 3-minute serial configuration window is available on cold boot only (not on vibration wake), so the radio stays silent while you configure the device.

---

## Hardware

| Signal | GPIO | Notes |
|---|---|---|
| Vibration sensor (SW-420) | GPIO4 | Active HIGH — triggers wake |
| Status LED | GPIO3 | Active HIGH — solid during advertising, slow blink during config |
| Serial / Power | USB | Configuration and power |

---

## LED Status

| Pattern | Meaning |
|---|---|
| Slow blink (1 s / 100 ms on) | Config window open (cold boot only) |
| Solid ON | Advertising iBeacon |
| OFF | Deep sleep |

---

## Boot / Wake Behaviour

```
Cold boot (USB connect or power-on)
  +-- Config window opens (3 min, LED slow blink)
       +-- After 3 min -> deep sleep, wake on GPIO4 HIGH

GPIO4 HIGH (vibration)
  +-- Advertise iBeacon for <duration> ms (LED solid ON)
       +-- Wait for GPIO4 LOW (stable 800 ms, up to 10 s)
            +-- deep sleep -> wait for next GPIO4 HIGH
```

> `CONFIG_ON_COLD_BOOT_ONLY = true` prevents the config window from opening on every vibration wake.

---

## Quick Start

1. Connect the device via USB.
2. The LED begins slow-blinking — the config window is open (3 minutes).
3. Open a serial monitor at **115200 baud**.
4. Type `HELP` to see all commands.
5. Configure as needed (see [Serial Commands](#serial-commands) below).
6. Type `SAVE` to persist settings to flash.
7. After the 3-minute window the device enters deep sleep automatically.
8. Trigger the vibration sensor (tap or shake) to verify advertising.

> **Note:** The config window opens on **cold boot only**. Vibration wakes go straight to advertising — no serial prompt.

---

## Serial Commands

Connect at **115200 baud**. The config window is active for 3 minutes after a cold boot (LED slow-blinks).

| Command | Description |
|---|---|
| `HELP` | Show command list |
| `CONFIG` | Show current settings |
| `SAVE` | Persist settings to NVS flash |
| `SET NAME <value>` | Beacon name (shown in BLE advertisement) |
| `SET UUID <uuid>` | Proximity UUID (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) |
| `SET MAJOR <0-65535>` | Major value |
| `SET MINOR <0-65535>` | Minor value |
| `SET DURATION <ms>` | Advertising duration in ms (1–3,600,000) |
| `SET TXPOWER <dBm>` | TX power, e.g. `-8` for ~3 m range |

---

## Default Configuration

| Parameter | Default |
|---|---|
| Beacon name | Trattoria Beacon |
| Proximity UUID | `8a46676d-6348-4d50-a0e6-af5a508def40` |
| Manufacturer ID | `0x4C00` (Apple iBeacon) |
| Major | 1 |
| Minor | 1 |
| Advertising duration | 60,000 ms (1 minute) |
| TX power | -8 dBm (~3 m range) |
| Config window | 180 s (cold boot only) |

---

## Build and Upload

```bash
# Build
pio run --environment lolin_c3_mini

# Upload
pio run --environment lolin_c3_mini --target upload --upload-port COM18

# Serial monitor
pio device monitor --environment lolin_c3_mini --port COM18 --baud 115200
```

---

## Normal Operation

1. Device is in deep sleep (~5 µA).
2. SW-420 detects vibration → GPIO4 goes HIGH → device wakes.
3. LED turns solid ON; iBeacon advertises for `<duration>` ms.
4. Device waits for vibration to stop (GPIO4 LOW, stable 800 ms).  
   If GPIO4 stays HIGH, the device sleeps after 10 s anyway.
5. Device returns to deep sleep; LED turns OFF.
6. Repeat from step 1.

---

## Scan / Verify

1. Install **nRF Connect** on iOS or Android.
2. Trigger the vibration sensor (shake or tap).
3. Scan and locate the beacon by its configured name.
4. Confirm UUID, Major, Minor, RSSI, and TX power.

---

## Project Structure

```
src/
  main.cpp                Active firmware
  ble_provisioning.cpp    Reserved (BLE GATT provisioning — not active)
  ble_provisioning.h
  provisioning_config.h
platformio.ini
QUICKSTART.txt
README.md
docs/images/
IMPLEMENTATION.txt        Detailed architecture notes
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Config window won't open | Power-cycle (USB disconnect + reconnect). Cold boot is the only trigger. |
| Serial commands ignored | Config window already expired (3 min). Power-cycle to reopen. |
| No beacon visible in nRF Connect | Confirm vibration sensor triggered GPIO4 HIGH. Check TX power and duration settings. |

---

## Dependencies

| Library | Version | Purpose |
|---|---|---|
| [NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino) | 2.1.0 | BLE stack |
| [ArduinoJson](https://arduinojson.org/) | ^7.0.0 | JSON config parsing |
