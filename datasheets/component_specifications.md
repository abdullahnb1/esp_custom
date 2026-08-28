# Component Selection and Technical Specifications

This directory contains component selection sheets, pinout references, and application circuit diagrams for the custom ESP32 expansion board.

---

## 1. Integrated IMU (Inertial Measurement Unit)
### Selected Component: **MPU-6050 / ICM-42688-P (or LSM6DSOX)**
* **Type:** 6-Axis MotionTracking (3-Axis Gyroscope + 3-Axis Accelerometer).
* **Supply Voltage:** 2.375V – 3.46V (Logic & VDD = 3.3V).
* **Interface:** I2C (Standard Mode 100kHz, Fast Mode 400kHz).
* **I2C Address:** `0x68` (AD0 = GND) or `0x69` (AD0 = 3.3V).
* **Key Pins:**
  - `VDD` (Pin 13): +3.3V with 100nF decoupling capacitor.
  - `GND` (Pin 18): System Ground.
  - `SDA` (Pin 24): I2C Serial Data (connected to ESP32 GPIO21 with 4.7kΩ pull-up).
  - `SCL` (Pin 23): I2C Serial Clock (connected to ESP32 GPIO22 with 4.7kΩ pull-up).
  - `INT` (Pin 12): Hardware Interrupt output (connected to ESP32 GPIO19).
  - `VLOGIC` (Pin 8): +3.3V with 10nF cap.
  - `C_REGOUT` (Pin 10): 2.2nF capacitor to GND.

---

## 2. 3S Logic BMS & Telemetry Fuel Gauge
### Selected Component Architecture:
#### A. Precision Telemetry & Monitoring: **INA3221 (Triple-Channel I2C Voltage/Current Monitor)** or **TI BQ76920 / INA226**
* **Type:** 3-Channel High-Side Current & Bus Voltage Monitor with I2C interface.
* **Operating Voltage:** 2.7V to 5.5V (+3.3V logic from ESP32 rail).
* **Sensed Bus Range:** 0V to 26V (fully covers 3S Li-Po/Li-Ion: 9.0V – 12.6V).
* **Channels:**
  - **CH1:** Measures Cell 1 Voltage & Pack Total Voltage.
  - **CH2:** Measures Cell 2 Midpoint Voltage.
  - **CH3:** Measures Total 3S Discharge Current across $10\text{ m}\Omega$ precision shunt resistor ($R_{SENSE}$).
* **Interface:** I2C (`0x40` default address).
* **Pins:**
  - `SDA`, `SCL` shared on system I2C bus (GPIO21 / GPIO22).
  - `CRIT`, `WARN`, `PV` alarm interrupt pins connected to ESP32 GPIO34 (Input-only, active alert).

#### B. 3S Hardware Battery Protection & Balancing: **BM3451 / HY2213 Series 3S Circuit**
* Over-charge protection (4.25V/cell cutoff).
* Over-discharge protection (2.80V/cell cutoff).
* Over-current and short-circuit protection with Dual N-Channel Power MOSFETs (e.g. AOD4184A / IRF8736).
* Passive cell balancing ($68\text{ }\Omega$ bleeder resistors per cell).

---

## 3. 4x Servo Motor Outputs
### Interface & Power Architecture:
* **Connector Type:** 4x 3-Pin standard 2.54mm pitch headers (`GND`, `V_SERVO`, `PWM`).
* **Power Source (`V_SERVO`):** High-efficiency 5.0V / 6.0V 5A Synchronous Buck Converter (e.g., TPS54531 / XL4015 step-down from 3S 11.1V).
* **Signal Lines:**
  - **Servo 1:** ESP32 GPIO13 (LEDC Channel 0, 50Hz, 1ms–2ms pulse).
  - **Servo 2:** ESP32 GPIO14 (LEDC Channel 1, 50Hz).
  - **Servo 3:** ESP32 GPIO27 (LEDC Channel 2, 50Hz).
  - **Servo 4:** ESP32 GPIO26 (LEDC Channel 3, 50Hz).
* **Protection:** 100Ω series damping resistor on each PWM signal line; bulk 470µF low-ESR electrolytic capacitor across `V_SERVO` rail to absorb inductive kickback.

---

## 4. 2x DC Motor Dual Driver
### Selected Component: **TB6612FNG (Dual H-Bridge Driver)** or **2x DRV8871**
* **Type:** High-efficiency Dual MOSFET H-Bridge IC.
* **Motor Supply ($V_M$):** 4.5V to 15V (connects directly to 3S Battery $V_{BAT}$ 11.1V–12.6V).
* **Logic Supply ($V_{CC}$):** 2.7V to 5.5V (+3.3V from ESP32).
* **Output Current:** 1.2A continuous per channel, 3.2A peak.
* **Control Truth Table:**
  | IN1 | IN2 | PWM | Mode |
  | :---: | :---: | :---: | :---: |
  | H | L | H | Forward |
  | L | H | H | Reverse |
  | L | L | H | Short Brake |
  | H | H | H | Short Brake |
  | X | X | L | Stop / Standby |
* **ESP32 GPIO Pin Assignment:**
  - `AIN1`: GPIO32
  - `AIN2`: GPIO33
  - `PWMA`: GPIO25 (Motor A Speed PWM)
  - `BIN1`: GPIO16
  - `BIN2`: GPIO17
  - `PWMB`: GPIO18 (Motor B Speed PWM)
  - `STBY`: GPIO5 (Hardware Standby, active high)

---

## 5. Codeable Integrated RGB LEDs (Status & Diagnostics)
### Selected Component: **WS2812B-2020 / SK6812 (Mini 2.0x2.0mm Addressable RGB LEDs)**
* **Protocol:** Single-wire NZR communication, 800 kbps, 24-bit True Color (GRB / RGB).
* **Supply Voltage:** +5V rail (or +3.3V compatible SK6812-MINI).
* **Topology:** Daisy-chained serial data loop:
  $$\text{ESP32 GPIO4} \xrightarrow{330\Omega} \text{DIN}_{\text{LED1}} \rightarrow \text{DOUT} \rightarrow \text{DIN}_{\text{LED2}} \rightarrow \dots \rightarrow \text{DIN}_{\text{LED8}}$$
* **LED Distribution on PCB Layout:**
  - **LED 1 (Index 0):** Adjacent to **Servo 1 Header**
  - **LED 2 (Index 1):** Adjacent to **Servo 2 Header**
  - **LED 3 (Index 2):** Adjacent to **Servo 3 Header**
  - **LED 4 (Index 3):** Adjacent to **Servo 4 Header**
  - **LED 5 (Index 4):** Adjacent to **IMU Sensor** (Motion / Attitude state)
  - **LED 6 (Index 5):** Adjacent to **3S BMS / Battery Port** (Charge / SOC / Fault state)
  - **LED 7 (Index 6):** Adjacent to **Motor A Output Terminal** (Direction / Speed / Stall)
  - **LED 8 (Index 7):** Adjacent to **Motor B Output Terminal** (Direction / Speed / Stall)

---

## 6. Enable (EN / Reset) & Boot Push Buttons
### Selected Components: **SMD Tactile Switches (e.g. C&K PTS645 / Panasonic EVQ / TS-1187A)**
* **Package:** 3x4x2.0mm SMD or 3x6x2.5mm SMD or 4.5x4.5mm SMD.
* **Actuation Force:** 160gf – 250gf.
* **Circuits:**
  - **EN Switch (`SW_EN`):** Connected between `ESP32_EN` (Pin 3) and `GND`.
    - Pull-up: $10\text{k}\Omega$ to $+3.3\text{V}$.
    - Power-on Delay Cap: $1\mu\text{F}$ or $100\text{nF}$ ceramic to `GND` ($\tau = 10\text{ms}$).
  - **BOOT Switch (`SW_BOOT`):** Connected between `ESP32_GPIO0` (Pin 25) and `GND`.
    - Pull-up: $10\text{k}\Omega$ to $+3.3\text{V}$.
    - Debounce / RF Glitch Filter: $10\text{nF}$ ceramic to `GND`.
