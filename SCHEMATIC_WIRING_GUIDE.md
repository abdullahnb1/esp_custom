# Custom ESP32 Robotics Board: Schematic & Wiring Guide

This document outlines the complete schematic design, pin mapping, power architecture, and KiCad wiring diagrams for the custom ESP32 board.

---

## 1. Complete ESP32-WROOM-32 Pin Allocation Matrix

All GPIO assignments are designed to avoid internal flash lines (GPIO6–11) and boot strapping conflicts (GPIO0, 2, 12, 15):

| ESP32 Pin | GPIO | Function / Net Name | Target Subsystem | Peripheral Type / Notes |
| :---: | :---: | :---: | :---: | :--- |
| **Pin 3** | `EN` | `ESP_EN` | **Reset / Enable Button** | 10kΩ Pull-up to 3.3V + 1µF/100nF RC delay to GND + Reset Tact Switch (SW_EN) |
| **Pin 25** | `GPIO0` | `ESP_BOOT` | **Boot Button (Strapping)** | 10kΩ Pull-up to 3.3V + 10nF filter to GND + Boot Tact Switch (SW_BOOT) |
| **Pin 26** | `GPIO4` | `LED_DATA` | **Status RGB LEDs** | WS2812B Serial Data Line (via 330Ω series resistor) |
| **Pin 29** | `GPIO5` | `MOTOR_STBY` | **DC Motor Driver** | TB6612FNG Standby Enable (Active HIGH, 10kΩ pull-down) |
| **Pin 16** | `GPIO13` | `SERVO_PWM1` | **Servo 1** | LEDC Timer Channel 0 (50Hz RC PWM, 100Ω series resistor) |
| **Pin 13** | `GPIO14` | `SERVO_PWM2` | **Servo 2** | LEDC Timer Channel 1 (50Hz RC PWM, 100Ω series resistor) |
| **Pin 11** | `GPIO27` | `SERVO_PWM3` | **Servo 3** | LEDC Timer Channel 2 (50Hz RC PWM, 100Ω series resistor) |
| **Pin 10** | `GPIO26` | `SERVO_PWM4` | **Servo 4** | LEDC Timer Channel 3 (50Hz RC PWM, 100Ω series resistor) |
| **Pin 8** | `GPIO32` | `MOTOR_AIN1` | **DC Motor Driver** | Motor A Direction Bit 1 |
| **Pin 9** | `GPIO33` | `MOTOR_AIN2` | **DC Motor Driver** | Motor A Direction Bit 2 |
| **Pin 14** | `GPIO25` | `MOTOR_PWMA` | **DC Motor Driver** | Motor A Speed (LEDC PWM 20kHz) |
| **Pin 27** | `GPIO16` | `MOTOR_BIN1` | **DC Motor Driver** | Motor B Direction Bit 1 |
| **Pin 28** | `GPIO17` | `MOTOR_BIN2` | **DC Motor Driver** | Motor B Direction Bit 2 |
| **Pin 30** | `GPIO18` | `MOTOR_PWMB` | **DC Motor Driver** | Motor B Speed (LEDC PWM 20kHz) |
| **Pin 38** | `GPIO19` | `IMU_INT` | **IMU Sensor** | Hardware Motion Interrupt (Active LOW/HIGH) |
| **Pin 33` | `GPIO21` | `I2C_SDA` | **IMU & 3S BMS** | System I2C Data (4.7kΩ pull-up to +3.3V) |
| **Pin 36` | `GPIO22` | `I2C_SCL` | **IMU & 3S BMS** | System I2C Clock (4.7kΩ pull-up to +3.3V) |
| **Pin 6** | `GPIO34` | `BMS_ALERT` | **3S BMS / INA3221**| Over-current / Under-voltage Alarm (Input Only) |
| **Pin 7** | `GPIO35` | `VBAT_SENSE` | **Analog Backup** | Resistor divider to monitor total pack voltage ($100\text{k}\Omega / 20\text{k}\Omega$) |

---

## 2. Power Architecture and Voltage Rails

```
 3S Li-Po Battery (9.0V - 12.6V)
       │
       ▼
 [ 3S BMS / Protection Board ] ─────────► [ V_BATT (11.1V-12.6V) ]
                                                │         │
 ┌──────────────────────────────────────────────┘         │
 │                                                        ▼
 │                                            [ DC Motor Driver (VM) ]
 ▼
[ Step-Down Buck Converter (TPS54531 / XL4015) ]
 5.0V / 6.0V Output @ 5A
       │
       ├─────────────────────────────────► [ 4x Servo V_SERVO Power Headers ]
       ├─────────────────────────────────► [ 8x WS2812B Addressable RGB LEDs ]
       │
       ▼
[ LDO Linear Regulator (AMS1117-3.3) ]
 +3.3V Logic Output @ 800mA
       │
       ├─────────────────────────────────► [ ESP32-WROOM-32 VDD ]
       ├─────────────────────────────────► [ MPU-6050 / ICM-42688 IMU VDD ]
       ├─────────────────────────────────► [ INA3221 BMS Fuel Gauge VS ]
       ├─────────────────────────────────► [ TB6612FNG Motor Driver VCC ]
       └─────────────────────────────────► [ EN & BOOT Pull-up Resistors ]
```

---

## 3. Subsystem Wiring Diagrams for KiCad

### A. Subsystem 1: Enable (EN / Reset) & Boot Buttons Circuit

The ESP32 requires specific RC timing on `EN` to ensure the +3.3V power rail is stable before booting. `GPIO0` controls bootloader entry mode (LOW at startup = UART download mode, HIGH = Flash boot).

```
                      +3.3V                              +3.3V
                        │                                  │
                      [10k]                              [10k]
                     (R_EN)                             (R_BOOT)
                        │                                  │
   ESP32 Pin 3 (EN) ────┼──────────────┐    ESP32 Pin 25 ──┼──────────────┐
                        │              │       (GPIO0)     │              │
                     [1µF / 100nF]  [SW_EN]              [10nF]       [SW_BOOT]
                     (C_EN)         (Reset)             (C_BOOT)        (Boot)
                        │              │                   │              │
   GND ─────────────────┴──────────────┴─── GND ───────────┴──────────────┴─── GND
```

#### How it Operates:
1. **Power-On Reset:** The $10\text{k}\Omega$ resistor ($R_{EN}$) and $1\mu\text{F}$ capacitor ($C_{EN}$) form an RC circuit with a time constant $\tau = R \times C = 10\text{ms}$. This holds `EN` low until $V_{DD}$ reaches a stable 3.0V+, preventing brownouts and flash corruption.
2. **Manual Reset (SW_EN):** Pressing `SW_EN` immediately pulls `EN` to GND. Releasing it reboots the ESP32.
3. **Firmware Upload (SW_BOOT):** Hold `SW_BOOT` down $\rightarrow$ Tap `SW_EN` $\rightarrow$ Release `SW_BOOT`. The ESP32 enters bootloader ROM flashing mode.
4. **Runtime User Button:** After the ESP32 finishes booting, `GPIO0` can be read in firmware as a general-purpose programmable input button with software debouncing!

---

### B. Auto-Programming Transistor Circuit (Optional for USB-UART bridge)

If using a USB-to-UART bridge (CH340K, CP2102, FT231X) with `DTR` and `RTS` control lines:

```
        DTR ────────[10k]────────── Base Q2 (NPN S8050)
         │                           │
         │                        Collector Q2 ───► ESP32 GPIO0 (BOOT)
         │                           │
         └────────── Collector Q1  Emitter Q2 ───── RTS
                      │
   ESP32 EN ◄─────── Emitter Q1
                      │
        RTS ────────[10k]────────── Base Q1 (NPN S8050)
```

---

### C. Subsystem 2: Integrated IMU (MPU-6050 / ICM-42688)

```
                     +3.3V
                       │
             ┌─────────┴─────────┐
             │                   │
           [4.7k]              [4.7k]
             │                   │
             ├────── I2C_SDA ────┼───────────────────► ESP32 GPIO21 (Pin 33)
             │                   │
             │       I2C_SCL ────┴───────────────────► ESP32 GPIO22 (Pin 36)
             │
   ┌─────────┴─────────────────────────────────────────┐
   │                    MPU-6050                       │
   │                                                   │
   │  [13] VDD ──────────── +3.3V (with 100nF + 10nF) │
   │  [8]  VLOGIC ───────── +3.3V                     │
   │  [18] GND ──────────── GND                        │
   │  [9]  AD0 ──────────── GND (sets I2C address 0x68)│
   │  [24] SDA ──────────── I2C_SDA                    │
   │  [23] SCL ──────────── I2C_SCL                    │
   │  [12] INT ──────────── IMU_INT ─────────────────► ESP32 GPIO19 (Pin 38)
   │  [10] REGOUT ───────── [2.2nF Ceramic] ── GND    │
   │  [20] CPOUT ────────── [2.2nF Ceramic] ── GND    │
   └───────────────────────────────────────────────────┘
```

---

### D. Subsystem 3: 3S BMS Telemetry (INA3221 Triple High-Side Sensor)

```
   From 3S LiPo Pack:
   [Cell 3 (+)] ─────────► INA3221 CH1_IN+ ──[10mΩ Shunt]──► CH1_IN- (To System V_BATT)
   [Cell 2 Tap] ─────────► INA3221 CH2_IN+ (Midpoint ~7.4V Sense)
   [Cell 1 Tap] ─────────► INA3221 CH3_IN+ (Low-cell ~3.7V Sense)
   [Pack Ground] ────────► Common System Ground (GND)

   INA3221 Logic Connections:
   ┌───────────────────────────────────────────────────┐
   │                    INA3221                        │
   │                                                   │
   │  V_S ───────────────── +3.3V (with 100nF cap)     │
   │  GND ───────────────── System GND                 │
   │  SDA ───────────────── I2C_SDA (ESP32 GPIO21)     │
   │  SCL ───────────────── I2C_SCL (ESP32 GPIO22)     │
   │  A0 ────────────────── GND (I2C Address 0x40)     │
   │  CRIT / WARN ───────── BMS_ALERT ───────────────► ESP32 GPIO34 (Pin 6)
   └───────────────────────────────────────────────────┘
```

---

### E. Subsystem 4: 4x Servo Motor Headers

```
   V_SERVO Rail (+5V / +6V from Buck Converter)
   ──────────────────┬─────────────────┬─────────────────┬─────────────────┐
                     │                 │                 │                 │
   [470µF/16V]       │                 │                 │                 │
   Capacitor         ▼                 ▼                 ▼                 ▼
   (Filter)       [Pin 2]           [Pin 2]           [Pin 2]           [Pin 2]
                ┌─────────┐       ┌─────────┐       ┌─────────┐       ┌─────────┐
                │ SERVO 1 │       │ SERVO 2 │       │ SERVO 3 │       │ SERVO 4 │
                └─────────┘       └─────────┘       └─────────┘       └─────────┘
                  ▲     ▲           ▲     ▲           ▲     ▲           ▲     ▲
    ESP32 GPIO13 ─┤     │           │     │           │     │           │     │
   [100Ω Res] ────┘     │           │     │           │     │           │     │
    ESP32 GPIO14 ───────┼───────────┤     │           │     │           │     │
   [100Ω Res] ──────────┼───────────┘     │           │     │           │     │
    ESP32 GPIO27 ───────┼─────────────────┼───────────┤     │           │     │
   [100Ω Res] ──────────┼─────────────────┼───────────┘     │           │     │
    ESP32 GPIO26 ───────┼─────────────────┼─────────────────┼───────────┤     │
   [100Ω Res] ──────────┼─────────────────┼─────────────────┼───────────┘     │
                        │                 │                 │                 │
   GND ─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

---

### F. Subsystem 5: Dual DC Motor Driver (TB6612FNG)

```
   3S Battery Rail (V_BATT 9V-12.6V) ──► TB6612 VM (Pin 24, with 100µF/25V + 100nF)
   +3.3V Logic Rail ──────────────────► TB6612 VCC (Pin 20, with 100nF)
   GND ───────────────────────────────► TB6612 GND / PGND

   Control Inputs:
   ESP32 GPIO32 ──────────────────────► AIN1 (Pin 17)
   ESP32 GPIO33 ──────────────────────► AIN2 (Pin 16)
   ESP32 GPIO25 (PWM) ────────────────► PWMA (Pin 23)

   ESP32 GPIO16 ──────────────────────► BIN1 (Pin 14)
   ESP32 GPIO17 ──────────────────────► BIN2 (Pin 13)
   ESP32 GPIO18 (PWM) ────────────────► PWMB (Pin 12)
   ESP32 GPIO5  ──────────────────────► STBY (Pin 19, with 10kΩ pull-down to GND)

   Motor Outputs:
   TB6612 AO1 (Pin 1, 2) ─────────────► Screw Terminal Motor A (+)
   TB6612 AO2 (Pin 5, 6) ─────────────► Screw Terminal Motor A (-)
   TB6612 BO1 (Pin 7, 8) ─────────────► Screw Terminal Motor B (+)
   TB6612 BO2 (Pin 11, 10) ───────────► Screw Terminal Motor B (-)
```

---

### G. Subsystem 6: Codeable Status RGB LEDs (WS2812B / SK6812)

```
   +5V Rail (from Buck Converter) ────┬──────────────┬──────────────┬──────────────┬──────────────┐
                                     │              │              │              │              │
                                  [100nF]        [100nF]        [100nF]        [100nF]        [100nF]
                                     │              │              │              │              │
   ESP32 GPIO4 ──[330Ω]──► DIN [LED 1] DOUT ──► DIN [LED 2] DOUT ──► ... ──► DIN [LED 8]
                            (Servo 1)             (Servo 2)                    (Motor B)
                                     │              │              │              │              │
   GND ──────────────────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

#### Diagnostic LED Role Map:
* **LED 1 (Index 0):** Servo 1 status indicator.
* **LED 2 (Index 1):** Servo 2 status indicator.
* **LED 3 (Index 2):** Servo 3 status indicator.
* **LED 4 (Index 3):** Servo 4 status indicator.
* **LED 5 (Index 4):** IMU sensor heartbeat / calibration state.
* **LED 6 (Index 5):** 3S Battery state (Green: >11.5V, Yellow: 10.5V–11.5V, Flashing Red: <10.0V cutoff).
* **LED 7 (Index 6):** Motor A status (Green = Forward, Red = Reverse, Blue = PWM Speed/Braking).
* **LED 8 (Index 7):** Motor B status (Green = Forward, Red = Reverse, Blue = PWM Speed/Braking).
