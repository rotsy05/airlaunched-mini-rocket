# Air-Launched Mini Rocket — Embedded Avionics System
---

## Overview

A 4-person team designed and built a mini rocket intended to be air-launched from an autonomous UAV. This repository contains the full embedded avionics software running onboard the rocket — from pre-launch standby to parachute deployment and touchdown detection.

The rocket was successfully tested via ground-based launch ramp with nominal flight, autonomous parachute deployment, and full data recovery.

| | |
|---|---|
| **Apogee reached** | ~300m |
| **Sensor polling rate** | 13ms |
| **Deployment safeties** | 3 independent triggers |
| **Team** | 4 members |

**Team structure:**
- 2 members — mechanical design & 3D printing
- 2 members (including myself) — electronics & embedded software *(hardware + software)*

---

## System Architecture

```
┌──────────────────────────────────────────────┐
│              ESP32  (MicroPython)             │
│                                              │
│  SX1262  LoRa  ──►  Remote launch trigger   │
│  BMP280  I2C   ──►  Barometric altimetry    │
│  MOSFET        ──►  Ignition circuit        │
│  Servo   PWM   ──►  Parachute release       │
│  Flash         ──►  Flight data logging     │
└──────────────────────────────────────────────┘
```

**Protocols:** I2C · SPI · PWM · LoRa (868 MHz)

---

## Hardware

| Component | Protocol | Role |
|-----------|----------|------|
| ESP32 | — | Main microcontroller — runs full avionics state machine |
| BMP280 | I2C | Barometric pressure & temperature — altitude estimation |
| SX1262 | SPI | Long-range radio module — remote launch authorization |
| MOSFET | — | Ignition circuit control |
| Servo | PWM | Parachute mechanical release |
| Custom PCB | — | In-house design & soldering |

---

## Flight Software — State Machine

### `STANDBY` → Awaiting launch authorization
The system listens for a remote launch command via LoRa radio (868.5 MHz). Security is handled at the operational level — protocol details are intentionally not documented here.

### `LAUNCHED` → Apogee detection
After ignition, the BMP280 is polled every ~13ms via I2C. Altitude is derived from pressure using the **standard atmosphere barometric formula**:

```
altitude = 44330 × (1 − (P / P₀)^0.1903)
```

Where `P₀` is the reference pressure measured at ground level before launch.

Apogee is detected by computing **vertical speed** over a rolling window of 15 barometric measurements (~195ms). Three independent safety triggers ensure parachute deployment:

| Trigger | Condition | Priority |
|--------|-----------|----------|
| Speed-based | Vertical speed < 3 m/s | Primary |
| Altitude ceiling | Altitude > 300m | Safety |
| Backup timer | Fixed delay after launch | Safety |

### `DEPLOYED` → Descent monitoring
A dedicated second thread (`_thread`) handles continuous flash writes, fully decoupled from the main flight loop via a deque-based log buffer. Post-deployment, the system continues logging pressure, altitude, and temperature at full rate.

Data is logged in a structured pipe-separated format:
```
timestamp | frame_type | value1 | value2 | ...
```
Frame types: `LOG` (events), `BMP` (pressure/altitude/temperature), `SPEED` (vertical speed)

### `TOUCHDOWN` → End of flight
Detected when altitude drops back below 15m. Logging continues for a final buffer window, then stops.

---

## Key Technical Challenges

**Real-time multithreading on a microcontroller**
Separating data logging from flight logic using `_thread` to avoid blocking the apogee detection loop — a dedicated thread handles all flash writes via a deque buffer.

**Robust apogee detection**
Using a rolling speed average over 15 barometric samples rather than a static altitude threshold — more reliable against sensor noise and pressure glitches.

**Embedded radio communication**
Configuring the SX1262 LoRa module required understanding blocking vs. non-blocking receive modes, SPI timing constraints, and EU frequency regulations (868 MHz band).

**Hardware integration**
Custom PCB design, soldering, and full system wiring under strict size and mass constraints for rocket embedding.

---

## Repository Structure

```
├── main.py            # Primary flight software (ESP32 + LoRa launch)
├── main_1.py          # Alternative version (RPi Pico, button-triggered)
├── bmp280.py          # BMP280 I2C driver (David Stenwall)
├── testAllumeur.py    # Igniter circuit test
├── test_missile.py    # Full system integration test
└── Servo.py           # Parachute servo test
```

---

## Results

| Test | Result |
|------|--------|
| Remote launch trigger | ✅ Nominal |
| Apogee detection | ✅ ~300m reached |
| Parachute deployment | ✅ Autonomous |
| Data recovery | ✅ Full flight log retrieved |

*Launched from ground ramp — air-launch from UAV was the intended final configuration.*

---

## Credits

Developed within **SIERA ESTACA** as part of the **Dassault UAV Challenge**.  
BMP280 driver by [David Stenwall](https://github.com/dafvid/micropython-bmp280).
