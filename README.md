# Beacon Firmware - ESP32-C3 iBeacon with Vibration Wake

![Project Cover / Prototype](docs/images/prototype-project-cover.jpg)

Prototype build and project cover photo.

**Version**: 1.1  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino  
**BLE Library**: NimBLE-Arduino 2.1.0  
**Status**: Production Ready

---

## Overview

Ultra-low-power iBeacon firmware for the ESP32-C3. The device spends nearly all its time in deep sleep (~5 uA). A SW-420 vibration sensor on GPIO4 wakes the device, which then broadcasts a standard Apple iBeacon advertisement for a configurable duration before returning to sleep.

A 3-minute serial configuration window is available on cold boot only (not on vibration wake), so the radio stays silent while you configure the device.

---

## Hardware

| Signal | GPIO | Notes |
|---|---|---|
| Vibration sensor (SW-420) | GPIO4 | Active HIGH - triggers wake |
| Status LED | GPIO3 | Active HIGH - solid during advertising, slow blink during config |

---

## Boot / Wake Behaviour
