# Functional Specification Document
## BLE Milk Frother Controller

**Project:** DIY BLE-Controlled Milk Frother with Temperature Regulation
**Version:** 1.1
**Date:** 2026-03-23
**Author:** terenceplett-tech

---

## 1. Project Overview

This project retrofits a SIMPLETASTE Electric Milk Frother with Bluetooth Low Energy (BLE) control via an ESP32 microcontroller. An Android application allows the user to remotely control the motor and heating element independently, set a target milk temperature, and receive notifications when the target is reached — at which point the device automatically shuts off.

---

## 2. Hardware

### 2.1 Components

| Component | Description |
|-----------|-------------|
| Milk Frother | SIMPLETASTE Electric Milk Frother (motor + heating element) |
| Microcontroller | AZ-Delivery ESP32 Development Board (38-pin) |
| Temperature Sensor | TI TMP61 Linear PTC Thermistor (analog, SOT-23 package) |
| Reference Resistor | 10kΩ 1% precision resistor (voltage divider with TMP61) |
| Relay Module x2 | 5V Single-Channel Relay (one for motor, one for heater) |
| Battery | 3.7V LiPo (recommended: 2000mAh) |
| Charging Module | TP4056 LiPo charger with protection circuit |
| Voltage Regulator | 3.3V LDO (e.g., AMS1117-3.3) for ESP32 and sensor supply |

### 2.2 Pin Assignment

| ESP32 GPIO | Connected To |
|------------|--------------|
| GPIO 26 | Relay 1 — Motor control |
| GPIO 27 | Relay 2 — Heater control |
| GPIO 34 | TMP61 ADC input (middle of voltage divider; input-only pin) |
| 3.3V | TMP61 + 10kΩ reference resistor supply, Relay VCC |
| GND | TMP61 GND, Relay GND, LDO GND |

### 2.3 TMP61 Voltage Divider Circuit

The TMP61 is a linear PTC thermistor read via the ESP32 ADC. A voltage divider is formed with a 10kΩ reference resistor:

```
3.3V ──── R_ref (10kΩ) ──── GPIO 34 (ADC) ──── TMP61 ──── GND
```

- ADC reads the voltage across the TMP61.
- TMP61 resistance is calculated from the ADC reading.
- Temperature is derived using the TMP61 resistance–temperature characteristic (TI datasheet, Table 1).
- TMP61 resistance at 25°C: 10kΩ; sensitivity: ~175 Ω/°C (linear PTC).
- Measurement range suitable for milk frothing: 0°C – 100°C.

### 2.4 Wiring Overview

```
[LiPo 3.7V] ──── [TP4056 Charger] ──── [AMS1117-3.3 LDO] ──── [ESP32 3.3V]
[230V Frother Motor] ──── [Relay 1] ──── GPIO 26 (ESP32)
[230V Frother Heater] ─── [Relay 2] ──── GPIO 27 (ESP32)
[TMP61 + 10kΩ Divider] ── GPIO 34 (ESP32 ADC)
```

> ⚠️ **Safety Note:** The frother operates on mains voltage (230V AC). All mains-side wiring must be properly insulated. Relay modules must be rated for mains voltage. This modification involves risk of electric shock — proceed with caution or consult a qualified electrician.

> 🔋 **Battery Note:** The ESP32 and TMP61 run at 3.3V supplied by the LDO. The relay coils require 5V — use a relay module with an optocoupler that accepts a 3.3V control signal, or include a 5V boost converter for the relay supply only.

---

## 3. System Architecture

```
[Android App] <──BLE──> [ESP32 BLE Server] <──── [LiPo Battery + LDO]
                               │
                    ┌──────────┼──────────┐
                    │          │          │
              [Relay - Motor] [Relay - Heater] [TMP61 + 10kΩ Voltage Divider]
                    │          │                       │
             [Frother Motor] [Frother Heater]      [GPIO 34 ADC]
```

---

## 4. BLE Specification

### 4.1 Device Name
`MilkFrother`

### 4.2 GATT Service

| | Value |
|--|--|
| **Service UUID** | `4fafc201-1fb5-459e-8fcc-c5c9c331914b` |

### 4.3 GATT Characteristics

| Characteristic | UUID | Properties | Data Type | Description |
|----------------|------|------------|-----------|-------------|
| Motor Control | `beb5483e-36e1-4688-b7f5-ea07361b26a8` | Read / Write | `uint8` (0=OFF, 1=ON) | Turn frother motor on/off |
| Heater Control | `beb5483f-36e1-4688-b7f5-ea07361b26a8` | Read / Write | `uint8` (0=OFF, 1=ON) | Turn heater on/off |
| Current Temperature | `beb54840-36e1-4688-b7f5-ea07361b26a8` | Read / Notify | `float32` (°C, little-endian) | Current milk temperature; notifies every 1 second |
| Target Temperature | `beb54841-36e1-4688-b7f5-ea07361b26a8` | Read / Write | `float32` (°C, little-endian) | Desired shutoff temperature; default 65°C |
| Status | `beb54842-36e1-4688-b7f5-ea07361b26a8` | Read / Notify | `string` | Notifies with `TARGET_REACHED` when target temp is hit |

---

## 5. ESP32 Firmware

### 5.1 Libraries Required

| Library | Source |
|---------|--------|
| ESP32 BLE Arduino | Built-in (Arduino ESP32 core) |
| Arduino ESP32 ADC | Built-in (no external library needed for TMP61 ADC read) |

### 5.2 Firmware Behaviour

- On power-up: both relays are OFF, BLE advertising starts.
- On BLE client connect: characteristics become readable/writable.
- On BLE client disconnect: both relays immediately turn OFF (safety shutoff). BLE advertising restarts.
- Temperature is read every **1 second** via ADC on GPIO 34:
  - ADC voltage is converted to TMP61 resistance via the voltage divider formula.
  - Resistance is mapped to temperature using the TMP61 R-T characteristic.
  - Result is notified to connected client as `float32` (°C).
- When `currentTemp >= targetTemp` and heater is ON:
  - Heater relay → OFF
  - Motor relay → OFF
  - Status characteristic notifies `TARGET_REACHED`
  - Motor and Heater characteristics updated to `0`

### 5.3 Battery Power Management

- When no BLE client is connected, the ESP32 enters **light sleep** to reduce current draw.
- BLE advertising interval is set to **1 second** (balance between discoverability and power).
- Relay coils draw significant current — relays are only energised when actively frothing.
- Estimated active current (ESP32 + BLE + relays): ~300 mA.
- Estimated standby current (light sleep + advertising): ~3–5 mA.
- Estimated battery life on 2000 mAh LiPo:
  - Standby only: ~400 hours
  - Mixed use (5 min active / hour): ~30+ hours typical use.

### 5.3 File Structure

```
firmware/
└── milk_frother_ble/
    └── milk_frother_ble.ino
```

---

## 6. Android Application

### 6.1 Platform

| | |
|--|--|
| **Language** | Kotlin |
| **Min SDK** | API 26 (Android 8.0) |
| **Target SDK** | API 34 (Android 14) |

### 6.2 Permissions Required

| Permission | Reason |
|------------|--------|
| `BLUETOOTH` | BLE operations (API < 31) |
| `BLUETOOTH_ADMIN` | BLE scanning (API < 31) |
| `BLUETOOTH_SCAN` | BLE scanning (API 31+) |
| `BLUETOOTH_CONNECT` | BLE connection (API 31+) |
| `ACCESS_FINE_LOCATION` | Required for BLE scan on Android |

### 6.3 UI Screens

#### Main Screen
| UI Element | Function |
|------------|----------|
| Connection Status Bar | Shows "Connected" / "Disconnected" + device name |
| Scan / Connect Button | Scans for `MilkFrother` device and connects |
| Current Temperature Display | Large live readout of temperature in °C |
| Target Temperature Slider | Range: 40°C – 90°C, default 65°C |
| Motor Toggle Button | ON/OFF — sends to Motor Control characteristic |
| Heater Toggle Button | ON/OFF — sends to Heater Control characteristic |
| Status Text | Displays notifications (e.g. "Target temperature reached!") |

### 6.4 BLE Client Behaviour

- Scan filters by device name `MilkFrother`.
- On connect: subscribe to Current Temperature and Status notifications.
- On temperature notify: update UI in real time.
- On Status notify `TARGET_REACHED`:
  - Show in-app notification / toast: "Target temperature reached! Device stopped."
  - Update Motor and Heater toggle buttons to OFF state.
- On disconnect: disable all controls, show reconnect prompt.

### 6.5 File Structure

```
android/MilkFrotherApp/
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/milkfrother/
│       │   ├── MainActivity.kt       # UI + BLE orchestration
│       │   └── BleManager.kt         # BLE scan, connect, read/write/notify
│       └── res/
│           ├── layout/activity_main.xml
│           └── values/
│               ├── strings.xml
│               └── colors.xml
├── build.gradle
└── settings.gradle
```

---

## 7. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | User can turn the frother motor ON/OFF independently via the Android app |
| FR-02 | User can turn the heater ON/OFF independently via the Android app |
| FR-03 | App displays the current milk temperature in real time (updated every 1 second) |
| FR-04 | User can set a target temperature between 40°C and 90°C |
| FR-05 | When current temperature reaches the target, the heater and motor automatically turn OFF |
| FR-06 | App displays a notification when the target temperature is reached |
| FR-07 | All relays turn OFF immediately if BLE connection is lost |
| FR-08 | App automatically reconnects or prompts user to reconnect after disconnection |

---

## 8. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | Temperature update latency must be ≤ 2 seconds |
| NFR-02 | BLE range must be sufficient for normal kitchen use (≥ 5 metres) |
| NFR-03 | Device must default to a safe (OFF) state on power-up and on disconnect |
| NFR-04 | App must support Android 8.0 and above |
| NFR-05 | Device must operate for a minimum of 8 hours of standby on a fully charged 2000 mAh LiPo |
| NFR-06 | Battery voltage must not drop below 3.0V during operation (LiPo protection threshold) |

---

## 9. Out of Scope

- iOS application
- Cloud connectivity or remote control (non-local BLE only)
- Custom PCB or enclosure design
- OTA firmware updates

---

## 10. Open Questions

- Should the motor be allowed to run without the heater (e.g. for cold frothing)?
  - Current design: yes, they are independent.
- Should there be a minimum temperature threshold before auto-shutoff triggers?
  - Prevents false shutoff if sensor reads high at startup.
- Should target temperature persist after disconnect/reboot on the ESP32?
  - Could store in NVS (non-volatile storage).
- Should battery voltage be exposed as a BLE characteristic so the app can display battery level?
  - Requires an ADC pin connected to a voltage divider on the LiPo output.
- What relay module will be used — confirm it supports 3.3V control signal from ESP32?
  - Some relay modules require 5V on the IN pin; an optocoupler-based module or level shifter may be needed.
- Should the TMP61 be mounted directly in the milk jug or on the frother body?
  - Direct immersion gives more accurate milk temperature but requires waterproofing the sensor and leads.
