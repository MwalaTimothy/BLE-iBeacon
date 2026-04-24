# ESP32-C3 BLE Gateway Firmware v2.1

**Production-Ready iBeacon Detection & KDS Integration**

**Version**: 2.1  
**Target**: ESP32-C3 (LOLIN C3 Mini)  
**Framework**: Arduino / NimBLE-Arduino 2.1.0  
**Companion**: [BLE-iBeacon System](https://github.com/MwalaTimothy/BLE-iBeacon)  
**Author**: MwalaTimothy | **Updated**: April 24, 2026

---

## Introduction

This **always-on Bluetooth Low Energy gateway** detects nearby iBeacon transmitters (from companion beacon devices), maintains a real-time tracking list, and reports detections to a **Kitchen Display System (KDS)** server via HTTP/JSON POST requests every 10 seconds (configurable).

**Key Capabilities:**
-  Continuous BLE scanning for Apple iBeacon advertisements
- Real-time beacon tracking (RSSI, signal strength, seen-count)
- Automatic stale beacon cleanup (30-second TTL)
- WiFi-based HTTP reporting to KDS server
- Serial configuration interface (115200 baud)
- NVS flash persistence across power cycles
- LED status indicator (GPIO3)

**Typical Use Case:** Track items (plates, trays, orders) moving through a restaurant kitchen using vibration-triggered beacon transmitters.

---

##  Architecture & Data Flow

### System Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    ESP32-C3 Gateway (Always-On)                  │
│                                                                  │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │  BLE Scanner    │  │  Beacon Tracking │  │ HTTP Reporting   ││
│  │  (NimBLE API)   │→→│  (Max 100 items) │→→│ (KDS Server)     ││
│  └─────────────────┘  └──────────────────┘  └──────────────────┘│
│         ▲                      │                      │           │
│         │                      ▼                      ▼           │
│    [iBeacon Packets]   [Internal State]       [WiFi HTTP POST]   │
│    (from beacons)      (RSSI, timestamps)     (JSON to /beacons) │
│                                                                  │
│  Serial Interface: 115200 baud (configuration)                  │
│  LED Status: GPIO3 (WiFi connectivity indicator)                │
│  Power: 5V USB (continuous operation)                           │
└──────────────────────────────────────────────────────────────────┘
```

### Operational Flow

```
1. INITIALIZATION (Setup Phase)
   ├─ Load configuration from NVS flash
   ├─ Initialize GPIO3 LED (low = off)
   ├─ Show 180-second serial config window
   ├─ Attempt WiFi connection (if credentials saved)
   ├─ Sync NTP time from pool.ntp.org
   └─ Start continuous BLE scanning

2. RUNTIME (Main Loop)
   ├─ BLE Scanning (every 5 seconds)
   │  ├─ Receive iBeacon advertisements
   │  ├─ Parse UUID, Major, Minor, RSSI
   │  ├─ Filter by target Major (65535 = wildcard)
   │  ├─ Add new or update existing beacon
   │  └─ Log per-packet if BLE DEBUG ON
   │
   ├─ Beacon Cleanup (every 5 seconds)
   │  └─ Remove beacons not seen for 30s (mark as LOST)
   │
   ├─ WiFi Health Check (every 15 seconds)
   │  └─ Reconnect if disconnected + credentials exist
   │
   ├─ HTTP Reporting (every 10 seconds)
   │  ├─ Build JSON: {gateway_id, major, minors_detected[], rssi[], timestamp}
   │  ├─ POST to KDS endpoint: http://IP:PORT/beacons
   │  ├─ Log HTTP response code (200, 202, 4xx, 5xx)
   │  └─ Print beacon summary to serial (always, WiFi or not)
   │
   └─ Serial Input Processing (continuous)
      ├─ Read commands from serial monitor
      ├─ Update configuration variables
      └─ Optionally persist to NVS

3. LED BLINKING
   ├─ Config window (180s): Slow blink (1s period)
   ├─ WiFi connected: Solid ON
   └─ WiFi disconnected: OFF

4. FALLBACK BEHAVIOR
   └─ No WiFi? Print beacon report to Serial only (every 10s)
```

---

## 📋 Source Code Implementation Details

### Key Data Structures

```cpp
// Beacon tracking entry (one per detected beacon)
struct BeaconInfo {
  uint16_t major;          // Zone/group identifier
  uint16_t minor;          // Item/object identifier
  int rssi;                // Signal strength (dBm) — updated on each detection
  uint32_t lastSeenMs;     // Last detection timestamp
  uint32_t firstSeenMs;    // Initial detection timestamp
  uint32_t seenCount;      // Total detections since first seen
  bool wasReportedBefore;  // Report state flag (reserved for future use)
};

// Global tracking array
BeaconInfo trackedBeacons[MAX_TRACKED_BEACONS];  // MAX = 100
int trackedBeaconCount = 0;  // Current count
```

### Default Configuration (defines)

```cpp
#define DEFAULT_GATEWAY_ID "gateway-001"
#define DEFAULT_BEACON_UUID "8a46676d-6348-4d50-a0e6-af5a508def40"
#define DEFAULT_MANUFACTURER_ID 0x004C              // Apple iBeacon
#define DEFAULT_MAJOR 0x0001                        // Zone ID
#define DEFAULT_KDS_IP "192.168.1.100"
#define DEFAULT_KDS_PORT 8080
#define DEFAULT_SCAN_INTERVAL_MS 5000               // 5 seconds
#define DEFAULT_REPORT_INTERVAL_MS 10000            // 10 seconds
#define BEACON_TIMEOUT_MS 30000                     // 30 seconds
#define MAX_TRACKED_BEACONS 100
```

### Core Functions

#### 1. **Beacon Detection** (`scanResultCallback`)
```cpp
// Invoked by NimBLE when iBeacon packet received
// Parses manufacturer data (25 bytes)
// Validates iBeacon marker (0x02 0x15 at bytes 2-3)
// Extracts Major, Minor, RSSI
// Filters by target UUID and Major
// Adds or updates beacon in tracking array
```

**Logic:**
- Check manufacturer data length ≥ 25 bytes
- Extract manufacturer ID (bytes 0-1) — must be 0x004C
- Check iBeacon marker (bytes 2-3) — must be 0x02 0x15
- Extract Major (bytes 20-21), Minor (bytes 22-23), RSSI from advertisement
- Apply UUID + Major filters
- Search tracking array for match (Major + Minor)
- If found: update RSSI, lastSeenMs, ++seenCount
- If not found: add new entry (if array < 100), log "NEW"

#### 2. **Beacon Cleanup** (`cleanupStaleBeacons`)
```cpp
// Runs every 5 seconds (static timer in loop)
// For each tracked beacon:
//   if (now - lastSeenMs > 30000 ms) → remove & log "LOST"
// Removal: shift array to maintain contiguity
```

#### 3. **HTTP Reporting** (`reportToKDS`)
```cpp
// Runs every 10 seconds (configurable)
// Builds JSON payload from tracked beacon array
// POSTs to KDS server
// If WiFi connected: send to server + print to serial
// If no WiFi: print to serial only (fallback mode)

// Payload format:
{
  "gateway_id": "gateway-001",
  "major": 1,
  "minors_detected": [1, 5, 12],
  "rssi": [-62, -74, -81],
  "timestamp": "2026-04-25T10:30:00Z"
}
```

#### 4. **Configuration Management** (NVS Flash)
```cpp
// loadConfiguration() → Read from flash on boot
// saveConfiguration() → Persist all settings to flash
// Stored settings:
//   - Gateway ID
//   - Beacon UUID
//   - Target Major
//   - KDS IP + port
//   - Scan/report intervals
//   - WiFi SSID + password (unencrypted)
```

#### 5. **WiFi Management**
```cpp
// connectWiFiBlocking() → Attempt connection with timeout
// connectConfiguredWiFi() → Use saved SSID/password
// syncNtpTime() → Sync clock from pool.ntp.org
// Auto-reconnect: 15-second retry if disconnected
```

---

## 🔧 Hardware Requirements

| Component | GPIO | Mode | Notes |
|-----------|------|------|-------|
| ESP32-C3 (LOLIN Mini) | — | MCU | 160MHz RISC-V core |
| USB-C | — | Power + Serial | 5V input, CH340 USB-to-serial |
| WiFi Status LED | GPIO3 | OUTPUT (active HIGH) | Optional; indicates connectivity |

**Power Supply:** 5V USB, continuous operation (no sleep mode)

---

## ⚙️ Configuration Commands (Serial Interface)

### Entry Requirements
- **Baud Rate:** 115200
- **Connection Window:** 180 seconds after cold boot
- **Timeout:** Auto-exit if no commands entered after 180s

### Command Reference

#### Status & Inspection
| Command | Purpose |
|---------|---------|
| `HELP` | Show all available commands |
| `CONFIG` | Display current settings |
| `WIFI SCAN` | List nearby WiFi networks |
| `WIFI STATUS` | Show WiFi connection details |
| `BEACONS` | List all tracked beacons (with RSSI, uptime, count) |

#### WiFi Management
| Command | Purpose |
|---------|---------|
| `WIFI CONNECT` | Connect using saved SSID/password |
| `WIFI DISCONNECT` | Disconnect from WiFi |
| `WIFI CLEAR` | Erase saved credentials |
| `AT+WIFI=SSID,PASSWORD` | Set credentials + connect + save (one-liner) |

#### BLE Debugging
| Command | Purpose |
|---------|---------|
| `BLE DEBUG ON` | Print every detected iBeacon packet to serial |
| `BLE DEBUG OFF` | Disable per-packet output |

#### Configuration (SET commands)
| Command | Example | Range |
|---------|---------|-------|
| `SET GW_ID <id>` | `SET GW_ID kitchen-1` | String (no spaces) |
| `SET UUID <uuid>` | `SET UUID 8a46676d-...` | UUID format |
| `SET MAJOR <val>` | `SET MAJOR 1` | 0-65535 (65535=ANY) |
| `SET KDS_IP <ip>` | `SET KDS_IP 192.168.1.50` | IPv4 address |
| `SET KDS_PORT <p>` | `SET KDS_PORT 8080` | 1-65535 |
| `SET SCAN_MS <ms>` | `SET SCAN_MS 5000` | 100-60000 |
| `SET REPORT_MS <ms>` | `SET REPORT_MS 10000` | 1000-600000 |
| `SET WIFI_SSID <s>` | `SET WIFI_SSID Office` | String |
| `SET WIFI_PASS <p>` | `SET WIFI_PASS pass123` | String |

#### Persistence & Control
| Command | Purpose |
|---------|---------|
| `SAVE` | Persist all settings to NVS flash |
| `CLEAR BEACONS` | Empty tracking array |

### Typical Configuration Session

```
[Serial Monitor @ 115200]

> CONFIG
========== GATEWAY CONFIGURATION ==========
Gateway ID:        gateway-001
Beacon UUID:       8a46676d-6348-4d50-a0e6-af5a508def40
Target Major:      1
KDS Server:        192.168.1.100:8080
Scan Interval:     5000 ms
Report Interval:   10000 ms
WiFi SSID:         <not set>
WiFi Password:     <not set>
WiFi Status:       ✗ Disconnected
Tracked Beacons:   0

> WIFI SCAN
Found 5 network(s):
  1. Office (RSSI: -35 dBm, Channel: 6)
  2. Guest (RSSI: -52 dBm, Channel: 1)
  3. TP-Link (RSSI: -67 dBm, Channel: 11)

> AT+WIFI=Office,SecurePassword123
[INFO] Connecting to WiFi: Office
[INFO] WiFi connected successfully
[INFO] IP Address: 192.168.1.50
[INFO] RSSI: -45 dBm

> SET KDS_IP 192.168.1.100
[INFO] KDS server IP set to: 192.168.1.100

> SET MAJOR 1
[INFO] Target major set to: 1
[INFO] Beacon tracking list cleared (major changed)

> SAVE
[INFO] Configuration saved to NVRAM

[Gateway running... listening for beacons]
[Automatic exit from config window after 180s]
```

---

## JSON Payload Specification

### Request Format

```http
POST http://KDS_IP:KDS_PORT/beacons HTTP/1.1
Host: KDS_IP:KDS_PORT
Content-Type: application/json
Content-Length: <length>

{
  "gateway_id": "gateway-001",
  "major": 1,
  "minors_detected": [1, 5, 12],
  "rssi": [-62, -74, -81],
  "timestamp": "2026-04-25T10:30:00Z"
}
```

### Response Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 OK | Data received & processed | Log success |
| 202 Accepted | Data queued for processing | Log success |
| 4xx | Client error (e.g., malformed JSON) | Log warning + retry |
| 5xx | Server error (e.g., unavailable) | Log error + retry |

---

## Getting Started

### Prerequisites

- **Hardware:** ESP32-C3 (LOLIN C3 Mini recommended)
- **Tools:** Arduino IDE 1.8.19+ or PlatformIO
- **Libraries:**
  - NimBLE-Arduino 2.1.0+
  - WiFi (built-in)
  - HTTPClient (built-in)
  - Preferences (built-in, NVS flash)

### Flashing the Firmware

#### Option A: Arduino IDE (Recommended)

1. **Install Board Support**
   - File → Preferences → Additional Board Manager URLs
   - Add: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - Tools → Board Manager → Search "esp32" → Install latest

2. **Install NimBLE Library**
   - Sketch → Include Library → Manage Libraries
   - Search "NimBLE-Arduino" → Install v2.1.0+

3. **Configure & Flash**
   - Select Board: `LOLIN C3 Mini`
   - Select Port: Your COM port
   - Copy source code → new sketch → Save
   - Sketch → Upload (or Ctrl+U)

#### Option B: esptool.py (Advanced)

```bash
# Install
pip install esptool

# Flash pre-built binary
esptool.py --chip esp32c3 --port COM3 --baud 460800 \
  write_flash 0x0 gateway_firmware_v2.1.bin
```

### First Boot Verification

1. Open Serial Monitor (115200 baud)
2. Type `CONFIG` to see loaded defaults
3. Type `AT+WIFI=YourSSID,YourPassword` to connect
4. Type `SAVE` to persist
5. Wait for "Configuration window closed" message
6. Bring a beacon near the gateway
7. Type `BEACONS` to confirm detection
8. Watch serial for HTTP POST output

---

## 📈 Beacon Tracking Rules

### Addition Rules
- **Trigger:** New (Major, Minor) pair detected
- **Action:** Add to array if count < 100
- **Log:** "NEW - Major:1 Minor:5 RSSI:-62 dBm"

### Update Rules
- **Trigger:** Existing beacon detected again
- **Action:** Update RSSI, lastSeenMs, ++seenCount
- **Log:** Silent (no output unless BLE DEBUG ON)

### Removal Rules
- **Trigger:** Beacon not seen for 30 seconds
- **Check:** Every 5 seconds
- **Action:** Remove from array, log "LOST"
- **Effect:** No more reports for that beacon

### Filtering Rules
- **UUID:** Must match configured beacon UUID
- **Major:** Must match target (unless 65535 wildcard)
- **Minor:** No filter (all minors accepted for matching major)

### Special Cases
- **Tracking Full (100 beacons):** New beacons ignored, warning logged
- **Major Changed:** Entire tracking list cleared automatically
- **Target Major = 65535:** Track ALL major values (wildcard mode)

---

## 🐛 Troubleshooting

### Beacons not detected
**Symptoms:** BEACONS list always empty  
**Checklist:**
1. ✅ Enable `BLE DEBUG ON` to see per-packet output
2. ✅ Verify beacon UUID matches configured UUID
3. ✅ Check target Major matches beacon's Major (or use 65535)
4. ✅ Move beacon within 3-10m of gateway
5. ✅ Verify beacon is powered and advertising

**Solution:**
```
BLE DEBUG ON                    # See what's being received
SET UUID <correct-uuid>         # If UUID mismatch
SET MAJOR 65535                 # Track all majors temporarily
SAVE
```

### WiFi connects but no HTTP POST
**Symptoms:** Serial says "WiFi connected" but no POST messages  
**Checklist:**
1. KDS server IP/port are correct
2. KDS server is running and listening
3. Firewall allows port access
4. Network routing between gateway and KDS

**Solution:**
```
WIFI STATUS                     # Verify connected + get IP
# From computer: ping 192.168.1.100 (KDS IP)
# From computer: curl -X POST http://192.168.1.100:8080/beacons
# Check KDS server logs for inbound requests
```

### WiFi keeps disconnecting
**Symptoms:** WiFi STATUS shows disconnected; auto-reconnecting  
**Checklist:**
1. WiFi signal strength (check RSSI)
2. Gateway in range of router
3. Router doesn't have MAC filtering enabled
4. SSID/password are correct

**Solution:**
```
WIFI STATUS              # Check RSSI (goal: -40 to -50 dBm)
WIFI DISCONNECT
WIFI CONNECT             # Manual reconnect
# Or move gateway closer to router
```

### LED not blinking
**Symptoms:** GPIO3 LED unresponsive  
**Checklist:**
1. LED polarity correct (longer leg = positive)
2. LED in range during config window (first 180s)
3. GPIO3 pin not damaged

**Solution:**
- Replace LED with known-good unit
- Verify GPIO3 electrically with multimeter

### Serial monitor shows garbled text
**Symptoms:** Unreadable characters in serial output  
**Cause:** Incorrect baud rate  
**Solution:** Verify baud rate = 115200 in serial monitor

---

## Security Considerations

**Important:** This implementation is suitable for **trusted networks only**.

- **WiFi Credentials:** Stored unencrypted in NVS flash
- **iBeacon UUID:** Visible in BLE advertisements (not a secret)
- **HTTP POST:** Unencrypted (no HTTPS/TLS)
- **KDS Server Auth:** Not implemented in firmware (add at server level)

### Recommendations
1. Use WPA2/WPA3 WiFi networks only
2. Add API key validation in KDS server
3. Restrict KDS port access via firewall to gateway MAC
4. Consider HTTPS/TLS at server level
5. Monitor gateway logs for unauthorized access

---

## Integration with Beacon Firmware

### Companion Project: BLE-iBeacon Beacon (v1.1)

The gateway pairs with a **vibration-triggered beacon transmitter**:

| Parameter | Must Match |
|-----------|-----------|
| Beacon UUID | ✅ Yes — Configure same UUID on both |
| Manufacturer ID | ✅ Yes — Both use 0x004C (Apple) |
| Major value | ✅ Yes — Coordinator with beacon's Major |
| Minor value | ✅ Tracked (no match required) |

### Configuration Example

**Beacon Side (set once, survives power cycles):**
```
SET UUID 8a46676d-6348-4d50-a0e6-af5a508def40
SET MAJOR 1
SET MINOR 5
SAVE
```

**Gateway Side (must match UUID & Major):**
```
SET UUID 8a46676d-6348-4d50-a0e6-af5a508def40
SET MAJOR 1
SAVE
```

**Result:** Whenever beacon (1:5) is detected within range, gateway reports it to KDS.

---

## 📝 Advanced Topics

### Wildcard Major Tracking

To track **all beacons regardless of Major**:
```
SET MAJOR 65535
SAVE
```
Now beacon Major values 0, 1, 2, ..., 65534 are all tracked.

### Increase Scan Duration

For deeper BLE discovery in noisy environments:
```
SET SCAN_MS 10000    # 10-second scan cycles
SAVE
```

### Reduce Report Frequency

To lower KDS server load:
```
SET REPORT_MS 30000  # Report every 30 seconds
SAVE
```

### Enable BLE Packet-Level Debug

To inspect every iBeacon received:
```
BLE DEBUG ON
# Observe output:
# [2026-04-25 10:30:15] [BLE-DBG] iBeacon mfg:004C major:1 minor:5 rssi:-62
BLE DEBUG OFF
```

---

## 📄 License

Released for use without restriction. Designed for integration with the [BLE-iBeacon](https://github.com/MwalaTimothy/BLE-iBeacon) project.

---


---

**Version:** 2.1  
**Last Updated:** April 24, 2026  
**Maintainer:** MwalaTimothy  
