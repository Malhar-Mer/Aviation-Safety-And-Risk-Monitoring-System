# Aviation Safety & Risk Monitoring System (ASRAM)

> **Educational prototype — not certified aviation equipment.** ASRAM demonstrates sensor acquisition, local alert handling, telemetry transport, a TFT display endpoint, and laptop-based monitoring. It must not be used for aircraft control, real-flight warnings, or operational safety decisions.

![ASRAM complete hardware wiring diagram](docs/ASRAM_Circuit_Diagram.svg)

## 1. Project overview

The Aviation Safety & Risk Monitoring System (ASRAM) is an embedded monitoring prototype built around an **ESP32**. It collects environmental and motion data, evaluates configured demonstration alert conditions, and sends telemetry to a laptop for monitoring and storage. An **Arduino UNO with an MCUFRIEND-style TFT shield** acts as a separate local display endpoint.

The current retained sensor set is:

- **DHT11** — ambient temperature and relative humidity.
- **BMP280** — barometric pressure and sensor temperature; altitude is a pressure-derived estimate.
- **MPU6050** — three-axis acceleration, gyroscope readings, and accelerometer-derived roll/pitch.
- **IR obstacle sensor** — digital object-detection event input (active LOW in the current firmware).

The ESP32 has two independent data outputs:

1. **USB serial to laptop:** newline-delimited JSON telemetry at **115200 baud**.
2. **UART2 to UNO TFT:** compact comma-separated/keyed display packet at **9600 baud**.

The intended software path is:

`ESP32 sensors → ESP32 JSON telemetry → Python serial/commissioning tools → local API → SQLite → Streamlit dashboard/reports`

The UNO TFT receives its display feed separately from UART2. Optional local AI/LLM explanation features belong to the laptop software layer; they are not part of the sensor circuit and must not override the deterministic monitoring state.

## 2. Current hardware scope

| Part | Role | Active in current design? |
|---|---|---:|
| ESP32 development board | Main sensor controller, alert logic, telemetry | Yes |
| DHT11 | Temperature and humidity | Yes |
| BMP280 | Pressure and sensor temperature | Yes |
| MPU6050 | Acceleration and gyroscope | Yes |
| IR obstacle module | Binary object/proximity event | Yes |
| Arduino UNO | TFT display controller only | Yes |
| MCUFRIEND-style TFT shield | Local display/HMI | Yes |
| Laptop | Serial commissioning, API, database, dashboard | Yes |
| LDR | Retired from active scope | **No** |
| GPS module | Retired from active scope | **No** |
| HC-SR04 ultrasonic sensor | Retired from active scope | **No** |

Do not re-add GPS, LDR, or HC-SR04 to the active wiring, firmware, telemetry contract, or acceptance checklist. Historical files that mention them are legacy material only.

## 3. Complete circuit diagram

The diagram above follows the pin definitions in the Milestone 13 ESP32 sketch. It shows the sensor inputs, local indicators, acknowledgement button, USB telemetry connection, and the one-way UART2 display connection.

**Important:** This is a reference wiring diagram derived from the current sketch, not a claim that every physical wire has been inspected. Check the exact ESP32 board, sensor breakout boards, IR module, buzzer, and TFT shield before applying power.

## 4. ESP32 pin connection table

### Sensors

| Component | Module pin | ESP32 connection | Notes |
|---|---|---|---|
| DHT11 | DATA | GPIO 4 | Use a 3.3 V-compatible signal. A bare DHT11 may need a 4.7–10 kΩ pull-up from DATA to 3V3; many breakout modules already include one. |
| DHT11 | VCC | 3V3, if supported by the module | Confirm the voltage marking/specification on the actual module. |
| DHT11 | GND | GND | Common ground. |
| BMP280 | SDA | GPIO 18 | Dedicated I²C bus 1 in the M13 sketch (`BMPWire`). |
| BMP280 | SCL | GPIO 22 | Dedicated I²C bus 1 in the M13 sketch. |
| BMP280 | VCC/VIN | 3V3 when the breakout supports it | Prefer 3.3 V logic and pull-ups. Do not assume every breakout accepts 5 V. |
| BMP280 | GND | GND | Common ground. Firmware tries I²C address `0x76`, then `0x77`. |
| MPU6050 | SDA | GPIO 21 | I²C bus 0 (`Wire`). |
| MPU6050 | SCL | GPIO 23 | I²C bus 0 (`Wire`). |
| MPU6050 | VCC | 3V3, if supported by the board | Confirm the breakout-board power requirements. Keep ESP32-side I²C at 3.3 V. |
| MPU6050 | GND | GND | Common ground. Sketch uses address `0x68`. |
| IR module | OUT | GPIO 27 | Current firmware treats **LOW as object detected**. |
| IR module | VCC | 3V3 only if the module supports 3.3 V | If the module needs 5 V, its OUT signal must be level-shifted/divided to a safe ESP32 input voltage. |
| IR module | GND | GND | Common ground. |

### Local indicators and acknowledgement

| Component | ESP32 pin | Connection detail |
|---|---:|---|
| Green LED | GPIO 32 | GPIO32 → 220–330 Ω series resistor → LED anode; LED cathode → GND. Indicates healthy SAFE state in the sketch. |
| Red LED | GPIO 33 | GPIO33 → 220–330 Ω series resistor → LED anode; LED cathode → GND. Indicates active alert or monitoring degradation. |
| Buzzer control | GPIO 14 | Firmware drives this pin HIGH while an alert is active and not acknowledged. Use a suitable transistor/MOSFET driver if the buzzer current exceeds the ESP32 GPIO capability. |
| Acknowledge push-button | GPIO 19 | One side to GPIO19, the other side to GND. The sketch configures `INPUT_PULLUP`, so pressed = LOW. |

**Buzzer driver recommendation:** for a bare buzzer or a buzzer that draws more current than a GPIO can safely provide, use an NPN transistor (for example, 2N2222/BC547) or logic-level MOSFET as a low-side switch. Add a base/gate resistor as appropriate. If the buzzer is an inductive type, fit a flyback diode across it. Select the buzzer supply from its own rating; do not power a high-current buzzer directly from GPIO14.

### ESP32 ↔ Arduino UNO TFT

| Signal | From | To | Protocol |
|---|---|---|---|
| Display data | ESP32 GPIO 17 (UART2 TX) | Arduino UNO D10 (software-serial RX in the documented display arrangement) | UART, 9600 baud, 8-N-1, one-way |
| Ground reference | ESP32 GND | UNO GND | Must be common |
| UART2 RX | ESP32 GPIO 16 | **Not connected in the one-way display link** | Reserved/configured as UART2 RX in the sketch; no UNO return data is required by this link. |

The UNO remains the display controller, with the MCUFRIEND-style TFT shield mounted on it. Confirm that **D10 is actually available** with the exact shield and the currently installed TFT sketch before connecting the UART wire. Preserve and back up the working UNO TFT sketch.

The ESP32-to-UNO direction is 3.3 V logic. Do not connect a 5 V UNO TX signal directly to an ESP32 input. A UNO-to-ESP32 return path is not part of the current one-way design; use a suitable level shifter or divider before adding one.

### ESP32 ↔ laptop

- Connect the ESP32 USB port directly to the laptop.
- USB serial is reserved for JSON telemetry at **115200 baud**.
- The firmware emits one UTF-8 JSON object followed by a newline for each telemetry packet.
- Windows COM port numbers can change. Check Device Manager before each run; `COM5` was reported previously but is not guaranteed.

## 5. Power and grounding rules

1. Power the ESP32 from its USB connection during bench commissioning unless the project has a separately reviewed regulated supply arrangement.
2. Feed sensor breakout boards from the ESP32 **3V3** rail only when their board specifications permit 3.3 V operation. If a module requires a different supply, keep its signal output within the ESP32's 3.3 V input limits.
3. Join the grounds of the ESP32, all sensor modules, the IR module, the indicator/driver circuit, and the UNO. The ESP32-to-UNO UART link requires a common ground.
4. Power the UNO/TFT from the UNO USB port or a suitable regulated supply. Do not attempt to power the UNO/TFT shield from the ESP32 3V3 pin.
5. Never apply 5 V directly to an ESP32 GPIO. This includes the IR sensor output and any future UNO-to-ESP32 signal.
6. Check the polarity of every module and LED before powering the circuit. Use current-limiting resistors for discrete LEDs.
7. Do not connect or disconnect sensor wiring while the circuit is powered.

## 6. Firmware reference

The current M13 reference sketch uses the following assignments and intervals:

| Function | Configuration |
|---|---|
| DHT11 data | GPIO 4; sample every 2500 ms |
| BMP280 I²C | SDA GPIO 18, SCL GPIO 22; sample every 2000 ms |
| MPU6050 I²C | SDA GPIO 21, SCL GPIO 23; sample every 100 ms |
| IR input | GPIO 27, active LOW |
| Green LED / red LED | GPIO 32 / GPIO 33 |
| Buzzer / acknowledge button | GPIO 14 / GPIO 19 (`INPUT_PULLUP`) |
| UNO TFT output | UART2 TX GPIO 17 → UNO D10; 9600 baud; packet every 500 ms |
| Laptop telemetry | USB serial; 115200 baud; JSON every 2000 ms |
| JSON node identity | `ESP32_ENV`; schema version `1` |

The UNO packet retains these fields in the firmware: `TS, IR, T, H, P, ALT, AX, AY, AZ, MT, GX, GY, GZ, ROLL, PITCH, S`.

The USB JSON stream includes sensor status, environmental readings, motion readings, IR state, data validity, alert flags, acknowledgement state, sequence number, timestamps, and time-synchronization state.

### Demonstration alert conditions

These are configurable prototype thresholds, **not aviation operating limits**:

| Condition | Firmware behavior |
|---|---|
| Temperature ≥ 40 °C for 5 seconds | Temperature warning becomes active |
| Temperature ≥ 45 °C while warning is active | Critical-temperature flag becomes active |
| Temperature < 38 °C | Temperature warning clears |
| Absolute roll or pitch ≥ 30° for 1 second | Tilt alert becomes active |
| Both absolute roll and pitch < 25° | Tilt alert clears |
| IR input LOW | Object-detection event contributes to active alert |
| Acknowledge button pressed during active alert | Alert is marked acknowledged; buzzer is muted while acknowledged |

The firmware status values are: `0 = SAFE`, `1 = ALERT`, `2 = ACKNOWLEDGED`, and `3 = MONITORING DEGRADED`. A sensor failure/degraded state must not be interpreted as SAFE.

## 7. Data and software flow

1. ESP32 reads the four retained sensors.
2. ESP32 computes local alert flags and drives the green LED, red LED, and buzzer control output.
3. ESP32 sends newline-delimited JSON over USB serial to the laptop.
4. The Python commissioning/ingestion tools validate the packet structure, sensor statuses, and sequence information.
5. The local API, SQLite database, and Streamlit dashboard provide the software monitoring and history path where configured.
6. In parallel, ESP32 UART2 sends display data to the UNO TFT.
7. Optional local LLM explanations are advisory software output only. Deterministic backend logic remains authoritative for the monitoring state.

## 8. Commissioning and test procedure

> Back up the currently working ESP32 and UNO sketches before changing firmware. The M13 sketch must be compiled using the exact ESP32 board package and libraries installed on the target computer before upload.

1. Inspect the board labels and verify every pin in Sections 4–5.
2. Confirm that BMP280 and MPU6050 are on their separate I²C buses as listed above; do not wire both to the same pins when using this M13 reference sketch.
3. Check that the IR module's OUT pin cannot exceed 3.3 V and that LOW corresponds to detection.
4. Verify the UNO TFT shield pin map and ensure D10 is available for the documented serial receiver.
5. Compile the ESP32 sketch in Arduino IDE. Do not upload if compilation fails or the wiring does not match.
6. Upload the firmware and connect the ESP32 to the laptop by USB.
7. Close Arduino Serial Monitor or any other program that has opened the ESP32 COM port.
8. Run the read-only commissioning utility from the project environment (replace `COM5` with the port shown in Device Manager):

   ```powershell
   py -3.11 tools_serial_commission.py --esp32-port COM5 --baud 115200 --duration 20 --min-packets 3
   ```

9. Require at least three consecutive complete packets with all four retained sensor statuses healthy and no schema/sequence errors before proceeding.
10. Apply controlled bench stimuli: change temperature/humidity in a safe manner, move/tilt the MPU6050, and trigger the IR sensor. Confirm the corresponding telemetry fields change plausibly.
11. Verify that missing or failed sensor data produces a degraded/incomplete status rather than a healthy reading.
12. Test the TFT link independently at 9600 baud and confirm that the display updates without breaking the existing TFT functions.
13. Connect the validated live stream to the local API/database/dashboard only after the serial commissioning gate passes.

## 9. Software endpoints (when the local services are running)

- Streamlit dashboard: `http://127.0.0.1:8501`
- Local API base: `http://127.0.0.1:8080`
- Health check: `GET http://127.0.0.1:8080/health`
- Telemetry ingestion endpoint documented in the earlier service setup: `POST /v1/telemetry`

The API and dashboard must be started using the configuration supplied in the repository. Keep bearer tokens and Wi-Fi credentials private; never commit secrets to GitHub.

## 10. Troubleshooting

| Symptom | Checks |
|---|---|
| BMP280 not detected | Check 3V3/GND, SDA GPIO18, SCL GPIO22; confirm address `0x76` or `0x77`; inspect the breakout's voltage requirements. |
| MPU6050 not ready | Check 3V3/GND, SDA GPIO21, SCL GPIO23, and address `0x68`; inspect I²C wiring and module orientation. |
| DHT11 read error | Check DATA on GPIO4, common ground, module power, and pull-up resistor if using a bare sensor. |
| IR state is reversed | The active firmware expects LOW = detected. Check the module output logic and sensitivity adjustment. |
| ESP32 resets or readings jump | Check USB supply quality, loose ground wires, wiring length, and sensor power. |
| No JSON in Python tool | Check the selected COM port and 115200 baud; close Serial Monitor; ensure USB output is JSON-only. |
| UNO TFT does not update | Check GPIO17 → UNO D10, 9600 baud, common ground, packet format, and whether D10 conflicts with the installed TFT shield/sketch. |
| ESP32 input may receive 5 V | Stop and add a suitable level shifter or divider before reconnecting. |

## 11. Verification status and limitations

- The project reference material records sensor initialization and live bench readings for the retained sensor set.
- The Milestone 12 software-focused test run recorded 9/9 tests passing in its build environment. This does not prove the M13 sketch compiles on the target PC or that every physical wire is correct.
- The M13 source itself states that it has not been compiled against the exact board package and library versions installed on the user's PC. Treat target-machine compilation and physical verification as pending until completed.
- The UART2-to-UNO link and TFT pin compatibility must be rechecked on the actual assembly.
- The BMP280 altitude value is derived from pressure using a fixed sea-level reference; it is not GPS altitude.
- A digital IR module reports a detection state, not a calibrated distance.
- MPU6050 roll/pitch are calculated from accelerometer axes; they are not a calibrated aircraft attitude solution.

## 12. Safety and engineering boundary

ASRAM is an educational laboratory prototype. It is not certified aviation equipment and is not validated for use during real flight, aircraft control, pilot warning, navigation, or operational safety decisions. Thresholds are demonstration values. Real aviation use would require qualified engineering review, validated and calibrated sensors, redundancy, environmental and failure testing, formal hazard analysis, and applicable certification.

---

**Project:** Aviation Safety & Risk Monitoring System (ASRAM)  
**Controller:** ESP32  
**Display endpoint:** Arduino UNO + MCUFRIEND-style TFT  
**Active sensors:** DHT11, BMP280, MPU6050, IR obstacle sensor  
**Primary telemetry:** USB JSON, 115200 baud  
**Display telemetry:** UART2, 9600 baud
