# BLE iBeacon — ESP32-C3 with Vibration Wake

![Prototype](docs/images/prototype-project-cover.jpg)

**Version**: 1.1 — Production Ready  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino / NimBLE-Arduino 2.1.0

---

## What It Does

An ultra-low-power Apple iBeacon built on the ESP32-C3. The device spends nearly all of its life in deep sleep (~5 µA). A SW-420 vibration sensor on GPIO4 acts as the trigger — any vibration wakes the chip, which immediately starts broadcasting a standards-compliant iBeacon advertisement. Once the vibration stops and the advertising window expires, the device returns to deep sleep automatically.

A 3-minute serial configuration window opens on every cold boot (USB connect / power-on), allowing you to customise the beacon's UUID, name, major/minor values, advertising duration, and TX power. The config window is deliberately suppressed on vibration wakes so the radio stays silent while you configure.

---

## Hardware

| Signal | GPIO | Notes |
|---|---|---|
| Vibration sensor (SW-420) | GPIO4 | Active HIGH — triggers wake from deep sleep |
| Status LED | GPIO3 | Active HIGH — solid during advertising, slow blink during config |
| USB | — | Serial config at 115200 baud + power |

---

## Boot and Wake Behaviour

```
Cold boot  (USB connect or power-on)
  └── Config window opens (3 min, LED slow-blinks)
        └── After 3 min → deep sleep, wake-on GPIO4 HIGH

GPIO4 HIGH  (vibration detected)
  └── Advertise iBeacon for <duration> ms  (LED solid ON)
        └── Wait for GPIO4 LOW (stable 800 ms, up to 10 s timeout)
              └── Return to deep sleep → wait for next vibration
```

---

## Firmware Architecture

### Operating Modes

| Mode | Current draw | Duration | Exit condition |
|---|---|---|---|
| Deep sleep | ~5 µA | Indefinite | GPIO4 HIGH (vibration) |
| BLE advertising | ~15 mA avg | Configurable (default 60 s) | Duration elapsed |
| Serial config | ~80 mA | 3 min (cold boot only) | Timeout → deep sleep |

### BLE Advertisement

The firmware uses **NimBLE-Arduino** to emit a standard Apple iBeacon packet:

- Manufacturer ID: `0x4C00` (Apple iBeacon standard)
- Advertising interval: 100 – 200 ms (slots 0x06 – 0x12)
- Flags: `0x04` (BR_EDR_NOT_SUPPORTED)
- TX power: configurable (default −8 dBm ≈ 3 m range)
- Payload: Proximity UUID + Major + Minor + TX Power byte

### NVS Configuration Storage

Settings are persisted to flash using the ESP-IDF Preferences API under namespace `beacon`:

| Key | Type | Default |
|---|---|---|
| name | String | Trattoria Beacon |
| uuid | String | 8a46676d-6348-4d50-a0e6-af5a508def40 |
| manId | UShort | 0x4C00 |
| major | UShort | 1 |
| minor | UShort | 1 |
| duration | ULong | 60000 ms |

Settings survive deep sleep and power cycles. They are only changed via serial commands and take effect on the next wake.

### Vibration Detection and Sleep Logic

The SW-420 is wired to GPIO4 configured as a deep-sleep wake-up source (active HIGH). On wake:

1. BLE advertising starts immediately (LED solid ON).
2. The firmware waits for GPIO4 to go LOW and remain stable for **800 ms**.
3. If GPIO4 stays HIGH beyond **10 s**, the device sleeps anyway (debounce timeout).
4. `esp_deep_sleep_start()` is called — the CPU halts, drawing ~5 µA until the next vibration.

---

## Default Configuration

| Parameter | Default |
|---|---|
| Beacon name | Trattoria Beacon |
| Proximity UUID | 8a46676d-6348-4d50-a0e6-af5a508def40 |
| Manufacturer ID | 0x4C00 (Apple iBeacon) |
| Major | 1 |
| Minor | 1 |
| Advertising duration | 60 000 ms (1 minute) |
| TX power | −8 dBm (~3 m range) |
| Config window | 180 s (cold boot only) |

---

## Serial Configuration

Connect at **115200 baud** within 3 minutes of a cold boot (LED slow-blinks).

| Command | Description |
|---|---|
| `HELP` | Show command list |
| `CONFIG` | Show current settings |
| `SAVE` | Persist settings to NVS flash |
| `SET NAME <value>` | Beacon name (1–20 chars) |
| `SET UUID <uuid>` | Proximity UUID (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx) |
| `SET MAJOR <0-65535>` | Major value |
| `SET MINOR <0-65535>` | Minor value |
| `SET DURATION <ms>` | Advertising duration (1 – 3 600 000 ms) |
| `SET TXPOWER <dBm>` | TX power: `-8` ≈ 3 m, `0` ≈ 10 m, `4` = max |

---

## Flashing

See **QUICKSTART.txt** in this repo for step-by-step flashing and first-use instructions.

The pre-built merged binary (`beacon_firmware_v1.1_merged.bin`) flashes to address `0x0` and includes the bootloader, partition table, and application in a single file.

```bash
esptool.py --chip esp32c3 --port <PORT> --baud 460800 write_flash 0x0 beacon_firmware_v1.1_merged.bin
```

---

## Scanning / Verification

1. Install **nRF Connect** (iOS or Android).
2. Trigger the vibration sensor (tap or shake the SW-420).
3. Scan — locate the beacon by its configured name.
4. Confirm UUID, Major, Minor, RSSI, and TX power.

---

## License

Released for use without restriction. No source code is distributed with this binary release.