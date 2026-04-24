# BLE iBeacon System — ESP32-C3

**Version**: Beacon v1.1 / Gateway v2.1 — Production Ready  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino / NimBLE-Arduino 2.1.0

---

## System Overview

This project is a two-device BLE proximity system built on the ESP32-C3.

| Device | Role |
|---|---|
| **Beacon** | Vibration-triggered iBeacon transmitter. Deep sleeps at ~5 µA, wakes on physical impact. |
| **Gateway** | Always-on BLE scanner. Listens for beacons, tracks them, and reports detections to a Kitchen Display System (KDS) over WiFi via HTTP POST. |

The beacon identifies a physical object (e.g. a plate or tray). The gateway sits in a fixed location and detects whenever that object comes within range, then pushes the event data to a server for display or logging.

---

## How It Works

```
[ Physical object vibrates ]
        |
        v
[ Beacon wakes from deep sleep ]
[ Broadcasts Apple iBeacon advertisement ]
[ Major = zone ID, Minor = item ID ]
        |
        v  (BLE advertisement)
[ Gateway (always on) picks up the advertisement ]
[ Tracks: Major, Minor, RSSI, seen-count, timestamps ]
[ Every <report_interval> ms: POST /beacons to KDS server ]
        |
        v  (HTTP POST, JSON)
[ KDS server receives beacon event ]
[ Displays or logs item arrival ]
```

---

## Beacon Device

### Hardware

| Signal | GPIO | Notes |
|---|---|---|
| Vibration sensor (SW-420) | GPIO4 | Active HIGH — triggers wake from deep sleep |
| Status LED | GPIO3 | Active HIGH — solid during advertising, slow blink during config |
| USB | — | Serial config at 115200 baud + power |

### Boot and Wake Behaviour

```
Cold boot  (USB connect or power-on)
  +-- Config window opens (3 min, LED slow-blinks)
       +-- After 3 min -> deep sleep, wake-on GPIO4 HIGH

GPIO4 HIGH  (vibration detected)
  +-- Advertise iBeacon for <duration> ms  (LED solid ON)
       +-- Wait for GPIO4 LOW (stable 800 ms, up to 10 s timeout)
             +-- Return to deep sleep -> wait for next vibration
```

### Default Configuration

| Parameter | Default |
|---|---|
| Beacon name | Trattoria Beacon |
| Proximity UUID | 8a46676d-6348-4d50-a0e6-af5a508def40 |
| Manufacturer ID | 0x4C00 (Apple iBeacon) |
| Major | 1 |
| Minor | 1 |
| Advertising duration | 60 000 ms (1 minute) |
| TX power | -8 dBm (~3 m range) |
| Config window | 180 s (cold boot only) |

### Power Profile

| Mode | Current draw | Duration |
|---|---|---|
| Deep sleep | ~5 µA | Indefinite (until vibration) |
| BLE advertising | ~15 mA avg | Configurable (default 60 s) |
| Serial config | ~80 mA | 3 min (cold boot only) |

### Beacon Serial Commands

Connect at 115200 baud within 3 minutes of a cold boot.

| Command | Description |
|---|---|
| `HELP` | Show command list |
| `CONFIG` | Show current settings |
| `SAVE` | Persist settings to NVS flash |
| `SET NAME <value>` | Beacon name (1-20 chars) |
| `SET UUID <uuid>` | Proximity UUID (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx) |
| `SET MAJOR <0-65535>` | Major value |
| `SET MINOR <0-65535>` | Minor value |
| `SET DURATION <ms>` | Advertising duration (1 - 3 600 000 ms) |
| `SET TXPOWER <dBm>` | TX power: -8 = ~3 m, 0 = ~10 m, 4 = max |

---

## Gateway Device

### Hardware

| Signal | GPIO | Notes |
|---|---|---|
| WiFi status LED | GPIO3 | Slow blink = config window; solid = WiFi connected; off = WiFi disconnected |
| USB | — | Serial config at 115200 baud + power (always on) |

The gateway requires a stable 5 V USB supply. It runs continuously — no deep sleep.

### Architecture

The gateway performs three tasks concurrently in the main loop:

1. **BLE scanning** — continuously scans for Apple iBeacon advertisements, filters by UUID and Major value, tracks up to 100 beacons with RSSI and seen-count.
2. **Beacon housekeeping** — any beacon not seen for 30 s is removed from the tracking list.
3. **HTTP reporting** — every `<report_interval>` ms, POSTs a JSON payload to the KDS server endpoint `POST /beacons`.

### JSON Payload (sent to KDS)

```json
{
  "gateway_id": "gateway-001",
  "major": 1,
  "minors_detected": [1, 2, 5],
  "rssi": [-62, -74, -81],
  "timestamp": "2026-04-25T10:30:00Z"
}
```

Timestamps are sourced from NTP (`pool.ntp.org`) once WiFi is available.

### Default Configuration

| Parameter | Default |
|---|---|
| Gateway ID | gateway-001 |
| Beacon UUID | 8a46676d-6348-4d50-a0e6-af5a508def40 |
| Target Major | 1 (set to 65535 to track all majors) |
| KDS server IP | 192.168.1.100 |
| KDS server port | 8080 |
| Scan interval | 5 000 ms |
| Report interval | 10 000 ms |
| Beacon timeout | 30 000 ms (stale removal) |
| Config window | 180 s (cold boot only) |

### Beacon Tracking

- Tracks up to **100 beacons** simultaneously.
- Each entry stores: Major, Minor, RSSI, first-seen time, last-seen time, seen-count.
- Stale entries (not seen for 30 s) are pruned every 5 s.
- Setting a new Major value via serial clears the tracking list.
- Set Major to **65535** to track beacons of any major value.

### Gateway Serial Commands

Connect at 115200 baud within 3 minutes of a cold boot.

| Command | Description |
|---|---|
| `HELP` | Show all commands |
| `CONFIG` | Show current configuration |
| `SAVE` | Save configuration to NVS flash |
| `WIFI SCAN` | Scan available WiFi networks |
| `WIFI STATUS` | Show WiFi connection details |
| `WIFI CONNECT` | Connect using saved credentials |
| `WIFI DISCONNECT` | Disconnect from WiFi |
| `WIFI CLEAR` | Clear saved WiFi credentials |
| `BLE DEBUG ON` | Print each detected iBeacon packet to serial |
| `BLE DEBUG OFF` | Disable per-packet debug output |
| `BEACONS` | List all currently tracked beacons |
| `CLEAR BEACONS` | Clear beacon tracking list |
| `SET GW_ID <id>` | Set gateway identifier string |
| `SET UUID <uuid>` | Set beacon UUID filter |
| `SET MAJOR <val>` | Target major (0-65534; 65535 = ANY) |
| `SET KDS_IP <ip>` | KDS server IP address |
| `SET KDS_PORT <p>` | KDS server port (1-65535) |
| `SET SCAN_MS <ms>` | BLE scan interval (100-60 000 ms) |
| `SET REPORT_MS <ms>` | Report interval (1 000-600 000 ms) |
| `SET WIFI_SSID <s>` | WiFi SSID |
| `SET WIFI_PASS <p>` | WiFi password |
| `AT+WIFI=<s>,<p>` | Set credentials + connect + save in one step |

---

## Flashing

See **QUICKSTART.txt** for step-by-step first-use and flashing instructions.

The pre-built merged binary (`beacon_firmware_v1.1_merged.bin`) flashes to address `0x0`.

```
esptool.py --chip esp32c3 --port <PORT> --baud 460800 write_flash 0x0 beacon_firmware_v1.1_merged.bin
```

---

## Scanning / Verification

1. Install **nRF Connect** (iOS or Android).
2. Trigger the vibration sensor on the beacon (tap or shake the SW-420).
3. Scan — locate the beacon by its configured name.
4. Confirm UUID, Major, Minor, RSSI, and TX power.
5. On the gateway serial monitor, type `BEACONS` to confirm detection.
6. Watch the gateway serial output for the next HTTP report to the KDS server.

---

## License

Released for use without restriction. No source code is distributed with this binary release.
