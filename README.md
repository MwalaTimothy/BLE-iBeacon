# BLE iBeacon Firmware — ESP32-C3

Ultra-low-power iBeacon firmware for the **ESP32-C3 (LOLIN C3 Mini)**.  
The device spends nearly all its time in deep sleep (~5 µA). A SW-420 vibration sensor on GPIO4 wakes the device, which then broadcasts a standard Apple iBeacon advertisement for a configurable duration before returning to sleep.

**Version**: 1.1  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino / NimBLE-Arduino 2.1.0

---

## Hardware

| Signal | GPIO | Notes |
|---|---|---|
| Vibration sensor (SW-420) | GPIO4 | Active HIGH — triggers wake |
| Status LED | GPIO3 | Active HIGH — solid during advertising, slow blink during config |

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
| TX power | -8 dBm (~3 m range) |
| Config window | 180 s (cold boot only) |

---

## Flashing the Pre-built Binary

The `beacon_firmware_v1.1_merged.bin` is a **single merged image** (bootloader + partition table + app) that flashes to address **0x0**.

### Option A — esptool.py (any OS)

```
pip install esptool
esptool.py --chip esp32c3 --port <PORT> --baud 460800 write_flash 0x0 beacon_firmware_v1.1_merged.bin
```

Replace `<PORT>` with your serial port (e.g. `COM18` on Windows, `/dev/ttyUSB0` on Linux/macOS).

### Option B — ESP Flash Download Tool (Windows GUI)

1. Download [Flash Download Tools](https://www.espressif.com/en/support/download/other-tools) from Espressif.
2. Select **ESP32-C3** chip.
3. Add `beacon_firmware_v1.1_merged.bin` at address **0x0**.
4. Set baud to **460800**, select your COM port, click **START**.

### Option C — ESP Web Flasher (Chrome / Edge)

1. Open https://esp.huhn.me in Chrome or Edge.
2. Click **Connect** and select your ESP32-C3.
3. Add the merged binary at offset **0x0** and click **Program**.

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

---

## Serial Configuration

Connect at **115200 baud** within 3 minutes of a cold boot (LED slow-blinks).

| Command | Description |
|---|---|
| HELP | Show command list |
| CONFIG | Show current settings |
| SAVE | Persist settings to NVS flash |
| SET NAME <value> | Beacon name |
| SET UUID <uuid> | Proximity UUID |
| SET MAJOR <0-65535> | Major value |
| SET MINOR <0-65535> | Minor value |
| SET DURATION <ms> | Advertising duration (1 - 3 600 000 ms) |
| SET TXPOWER <dBm> | TX power (e.g. -8 for ~3 m range) |

---

## License

Released for use without restriction. No source code is distributed with this binary release.