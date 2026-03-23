# Functional Specification Document
## BLE Milk Frother Controller

**Project:** DIY BLE-Controlled Milk Frother with Temperature Regulation
**Version:** 1.2
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

## 9. Validation

This section defines the test procedures to verify that all functional and non-functional requirements have been met before the device is considered complete.

---

### 9.1 Hardware Validation

#### HW-V-01 — Power Supply Output
| | |
|--|--|
| **Objective** | Confirm LDO outputs stable 3.3V under load |
| **Equipment** | Multimeter |
| **Procedure** | 1. Fully charge LiPo. 2. Power on device. 3. Measure voltage at ESP32 3.3V pin and TMP61 VCC. |
| **Pass Criteria** | Voltage reads 3.28V – 3.32V with ESP32 active and BLE advertising. |

#### HW-V-02 — Relay Motor Control
| | |
|--|--|
| **Objective** | Confirm Relay 1 (GPIO 26) switches correctly |
| **Equipment** | Multimeter in continuity or voltage mode |
| **Procedure** | 1. Send Motor ON command via BLE. 2. Measure relay output contacts. 3. Send Motor OFF command. 4. Measure again. |
| **Pass Criteria** | Relay closes (continuity / voltage passes) on ON; opens on OFF. |

#### HW-V-03 — Relay Heater Control
| | |
|--|--|
| **Objective** | Confirm Relay 2 (GPIO 27) switches correctly |
| **Equipment** | Multimeter |
| **Procedure** | Same as HW-V-02 but using Heater toggle in app. |
| **Pass Criteria** | Relay closes on ON; opens on OFF. |

#### HW-V-04 — TMP61 Temperature Accuracy
| | |
|--|--|
| **Objective** | Confirm TMP61 ADC reading matches actual temperature |
| **Equipment** | Reference thermometer (e.g. kitchen probe), cup of water at known temperatures |
| **Procedure** | 1. Prepare water at 40°C, 60°C, and 80°C (verify with reference thermometer). 2. Immerse TMP61 sensor. 3. Read temperature shown in app. |
| **Pass Criteria** | App reading is within ±3°C of reference thermometer at all three points. |

#### HW-V-05 — Safety Shutoff on Power Loss
| | |
|--|--|
| **Objective** | Confirm relays default to OFF on power-up |
| **Equipment** | Multimeter |
| **Procedure** | 1. With relays previously ON, remove and reapply battery power. 2. Immediately measure relay output contacts. |
| **Pass Criteria** | Both relays are OPEN (OFF) within 1 second of power-up, before any BLE connection. |

---

### 9.2 BLE Communication Validation

#### BLE-V-01 — Device Discovery
| | |
|--|--|
| **Objective** | Confirm device is discoverable under the correct name |
| **Procedure** | 1. Open Android app. 2. Tap Scan. 3. Check scan results. |
| **Pass Criteria** | Device `MilkFrother` appears in the scan list within 5 seconds. |
| **Requirement** | FR-08, NFR-02 |

#### BLE-V-02 — BLE Connection
| | |
|--|--|
| **Objective** | Confirm app connects successfully to the ESP32 |
| **Procedure** | 1. Tap `MilkFrother` in scan results. 2. Observe connection status bar. |
| **Pass Criteria** | Status bar shows "Connected — MilkFrother" within 5 seconds. |

#### BLE-V-03 — BLE Range
| | |
|--|--|
| **Objective** | Confirm BLE connection holds at ≥ 5 metres |
| **Procedure** | 1. Connect to device. 2. Move phone 5 metres away (through typical kitchen environment). 3. Observe connection and temperature updates. |
| **Pass Criteria** | Connection remains stable and temperature updates continue without interruption. |
| **Requirement** | NFR-02 |

#### BLE-V-04 — Disconnect Safety Shutoff
| | |
|--|--|
| **Objective** | Confirm relays turn OFF immediately when BLE disconnects |
| **Procedure** | 1. Connect app and turn ON both motor and heater. 2. Disable Bluetooth on phone (or move out of range). 3. Observe relay state with multimeter. |
| **Pass Criteria** | Both relays open (OFF) within 2 seconds of BLE disconnection. |
| **Requirement** | FR-07, NFR-03 |

---

### 9.3 Functional Validation

#### FN-V-01 — Independent Motor Control
| | |
|--|--|
| **Objective** | Confirm motor can be turned ON/OFF without affecting heater |
| **Procedure** | 1. Turn heater ON, motor OFF. 2. Confirm only Relay 1 state changes when toggling motor. 3. Confirm heater relay is unaffected. |
| **Pass Criteria** | Motor relay toggles; heater relay remains unchanged. |
| **Requirement** | FR-01 |

#### FN-V-02 — Independent Heater Control
| | |
|--|--|
| **Objective** | Confirm heater can be turned ON/OFF without affecting motor |
| **Procedure** | 1. Turn motor ON, heater OFF. 2. Toggle heater ON and OFF. 3. Confirm motor relay is unaffected. |
| **Pass Criteria** | Heater relay toggles; motor relay remains unchanged. |
| **Requirement** | FR-02 |

#### FN-V-03 — Real-Time Temperature Display
| | |
|--|--|
| **Objective** | Confirm app displays live temperature updates |
| **Procedure** | 1. Connect app. 2. Observe temperature value on screen. 3. Warm the TMP61 sensor by hand. 4. Observe temperature change in app. |
| **Pass Criteria** | Temperature value updates within 2 seconds of sensor change. Reading increases as sensor is warmed. |
| **Requirement** | FR-03, NFR-01 |

#### FN-V-04 — Target Temperature Setting
| | |
|--|--|
| **Objective** | Confirm target temperature can be set across the full range |
| **Procedure** | 1. Move slider to minimum (40°C). 2. Move slider to maximum (90°C). 3. Set to 65°C. 4. Confirm value is accepted by firmware (read back characteristic). |
| **Pass Criteria** | Slider accepts and sends all values 40–90°C. Firmware characteristic read-back matches sent value. |
| **Requirement** | FR-04 |

#### FN-V-05 — Automatic Shutoff at Target Temperature
| | |
|--|--|
| **Objective** | Confirm device shuts off motor and heater when target is reached |
| **Procedure** | 1. Set target temperature to 5°C above current room temperature. 2. Turn ON motor and heater. 3. Warm the TMP61 sensor until the reading exceeds the target. |
| **Pass Criteria** | Both relays open (OFF) automatically. Motor and Heater buttons in app update to OFF state. |
| **Requirement** | FR-05 |

#### FN-V-06 — Target Reached Notification
| | |
|--|--|
| **Objective** | Confirm app notifies user when target temperature is reached |
| **Procedure** | Perform same steps as FN-V-05. Observe app UI after shutoff triggers. |
| **Pass Criteria** | App displays toast or status message "Target temperature reached! Device stopped." within 2 seconds of shutoff. |
| **Requirement** | FR-06 |

#### FN-V-07 — Full Frothing Cycle (End-to-End)
| | |
|--|--|
| **Objective** | Validate the complete real-world use case from start to finish |
| **Procedure** | 1. Fill frother with cold milk (~5°C). 2. Connect app. 3. Set target temperature to 65°C. 4. Turn ON motor and heater. 5. Wait for device to reach target and auto-shutoff. 6. Check milk quality. |
| **Pass Criteria** | Milk reaches 65°C (±3°C verified by reference thermometer). Device shuts off automatically. Milk is frothed. App notification is received. |
| **Requirement** | FR-01 through FR-06 |

---

### 9.4 Battery Validation

#### BAT-V-01 — Standby Battery Life
| | |
|--|--|
| **Objective** | Confirm device operates in standby for ≥ 8 hours |
| **Equipment** | Fully charged 2000mAh LiPo, multimeter or battery monitor |
| **Procedure** | 1. Fully charge battery. 2. Power on device (no BLE connection, no relays active). 3. Leave for 8 hours. 4. Measure battery voltage. |
| **Pass Criteria** | Battery voltage remains above 3.0V after 8 hours of standby. |
| **Requirement** | NFR-05, NFR-06 |

#### BAT-V-02 — Active Operation Battery Life
| | |
|--|--|
| **Objective** | Confirm device operates through a realistic frothing session without power failure |
| **Procedure** | 1. Fully charge battery. 2. Connect app. 3. Run a full frothing cycle (motor + heater ON for ~3 minutes). 4. Repeat 5 times. 5. Measure battery voltage after last cycle. |
| **Pass Criteria** | Device completes all 5 cycles. Battery voltage does not fall below 3.0V. |
| **Requirement** | NFR-06 |

#### BAT-V-03 — Low Battery Protection
| | |
|--|--|
| **Objective** | Confirm TP4056 protection circuit prevents over-discharge |
| **Equipment** | Adjustable bench power supply (simulate depleted battery) |
| **Procedure** | 1. Gradually reduce supply voltage below 3.0V. 2. Observe device behaviour. |
| **Pass Criteria** | TP4056 protection circuit disconnects load before LiPo reaches 2.5V. Device shuts down gracefully (relays OFF). |
| **Requirement** | NFR-06 |

---

### 9.5 Validation Sign-Off

| Test ID | Description | Result | Date | Notes |
|---------|-------------|--------|------|-------|
| HW-V-01 | Power supply output | ☐ Pass / ☐ Fail | | |
| HW-V-02 | Relay motor control | ☐ Pass / ☐ Fail | | |
| HW-V-03 | Relay heater control | ☐ Pass / ☐ Fail | | |
| HW-V-04 | TMP61 temperature accuracy | ☐ Pass / ☐ Fail | | |
| HW-V-05 | Safety shutoff on power-up | ☐ Pass / ☐ Fail | | |
| BLE-V-01 | Device discovery | ☐ Pass / ☐ Fail | | |
| BLE-V-02 | BLE connection | ☐ Pass / ☐ Fail | | |
| BLE-V-03 | BLE range | ☐ Pass / ☐ Fail | | |
| BLE-V-04 | Disconnect safety shutoff | ☐ Pass / ☐ Fail | | |
| FN-V-01 | Independent motor control | ☐ Pass / ☐ Fail | | |
| FN-V-02 | Independent heater control | ☐ Pass / ☐ Fail | | |
| FN-V-03 | Real-time temperature display | ☐ Pass / ☐ Fail | | |
| FN-V-04 | Target temperature setting | ☐ Pass / ☐ Fail | | |
| FN-V-05 | Automatic shutoff at target | ☐ Pass / ☐ Fail | | |
| FN-V-06 | Target reached notification | ☐ Pass / ☐ Fail | | |
| FN-V-07 | Full frothing cycle (E2E) | ☐ Pass / ☐ Fail | | |
| BAT-V-01 | Standby battery life | ☐ Pass / ☐ Fail | | |
| BAT-V-02 | Active operation battery life | ☐ Pass / ☐ Fail | | |
| BAT-V-03 | Low battery protection | ☐ Pass / ☐ Fail | | |

---

## 10. Out of Scope

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
