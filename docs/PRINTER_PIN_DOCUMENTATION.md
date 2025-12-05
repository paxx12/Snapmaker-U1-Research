# Comprehensive Pin and MCU Documentation
## Multi-Extruder 3D Printer System

This document describes all pins, their functions, behaviors, and MCU architectures for a multi-extruder CoreXY 3D printer system with 4 independent extruders, NFC/RFID filament detection, and advanced features.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [MCU Overview](#mcu-overview)
3. [Main MCU (mcu)](#main-mcu-mcu)
4. [Host MCU (Linux SBC)](#host-mcu-linux-sbc)
5. [Extruder MCUs (e0-e3)](#extruder-mcus-e0-e3)
6. [Communication and Protocols](#communication-and-protocols)
7. [Behavioral Notes](#behavioral-notes)
8. [Firmware Configuration Notes](#firmware-configuration-notes)

---

## System Architecture

The printer consists of **6 microcontrollers**:
- **1 Main MCU** (`mcu`): Controls motion system (XYZ), heated bed, chamber management
- **1 Host MCU** (`host`): Linux SBC GPIO for system control, NFC readers
- **4 Extruder MCUs** (`e0`, `e1`, `e2`, `e3`): Independent hotend control units

**Kinematics**: CoreXY
**Build Volume**: 271mm (X) × 335mm (Y) × 275mm (Z)
**Communication**: Serial UART/USB at 460800 baud (main), virtual serial for host

---

## MCU Overview

| MCU Name | Type | Serial Connection | Primary Function |
|----------|------|-------------------|------------------|
| `mcu` | STM32 (main controller) | `/dev/ttyS6` @ 460800 baud | Motion control, bed heating, chamber |
| `host` | Linux GPIO | `/tmp/klipper_host_mcu` | System control, NFC/RFID readers |
| `e0` | STM32 (toolhead) | USB: `platform-xhci-hcd.0.auto-usb-0:1.3:1.0` | Extruder 0 (left front) |
| `e1` | STM32 (toolhead) | USB: `platform-xhci-hcd.0.auto-usb-0:1.4:1.0` | Extruder 1 (left rear) |
| `e2` | STM32 (toolhead) | USB: `platform-xhci-hcd.0.auto-usb-0:1.2:1.0` | Extruder 2 (right front) |
| `e3` | STM32 (toolhead) | USB: `platform-xhci-hcd.0.auto-usb-0:1.1:1.0` | Extruder 3 (right rear) |

---

## Main MCU (mcu)

**Connection**: `/dev/ttyS6` @ 460800 baud
**Restart Method**: Command-based

### Motion System Pins

#### X-Axis (CoreXY)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PC11 | X Stepper STEP | Digital Out | Step pulses for X motor |
| PC12 | X Stepper DIR | Digital Out | Direction (inverted: `!`) |
| PB2 | X/Y/Z Enable | Digital Out | Shared enable (inverted: `!`) for all axes |
| PC10 | X Diag/Endstop | Digital In | TMC2240 DIAG0, pullup inverted (`^!`) |
| PB12 | X TMC2240 CS | SPI CS | Chip select for TMC2240 driver |

**Driver**: TMC2240 (SPI2 bus)
**Run Current**: 1.2A
**Sensorless Homing**: SGT=1, uses virtual_endstop via TMC2240 stallGuard
**Microsteps**: 64
**Rotation Distance**: 40mm

#### Y-Axis (CoreXY)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PD2 | Y Stepper STEP | Digital Out | Step pulses for Y motor |
| PB3 | Y Stepper DIR | Digital Out | Direction control |
| PB2 | Y Enable (shared) | Digital Out | Shared with X/Z (inverted: `!`) |
| PC14 | Y Diag/Endstop | Digital In | TMC2240 DIAG0, pullup inverted (`^!`) |
| PE12 | Y TMC2240 CS | SPI CS | Chip select for TMC2240 driver |

**Driver**: TMC2240 (SPI4 bus)
**Run Current**: 1.2A
**Sensorless Homing**: SGT=1, uses virtual_endstop
**Microsteps**: 64
**Rotation Distance**: 40mm

#### Z-Axis

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PE7 | Z Stepper STEP | Digital Out | Step pulses for Z motor |
| PE8 | Z Stepper DIR | Digital Out | Direction control |
| PB2 | Z Enable (shared) | Digital Out | Shared with X/Y (inverted: `!`) |
| PC15 | Z Diag/Endstop | Digital In | TMC2209 DIAG, pullup (`^`) |
| PB4 | Z TMC2209 UART | UART | Single-wire UART |

**Driver**: TMC2209 (UART, address 0)
**Run Current**: 0.7A
**Sensorless Homing**: SGTHRS=110, uses virtual_endstop
**Microsteps**: 16
**Rotation Distance**: 8mm with 3:1 gear ratio
**Travel**: -6mm to 275mm (allows below-bed probing)

### Heated Bed

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PE9 | Bed Heater | PWM | Heater control output |
| PC2 | Bed Thermistor | Analog In | NTC 100K 3950 temperature sensor |

**Control**: Dual PID system
- Primary PID: Kp=16.154, Ki=0.075, Kd=105.0 (for high temps)
- Secondary PID: Kp=51.172, Ki=0.625, Kd=95.0 (for low temps)
**PWM Cycle**: 1 second
**Temperature Range**: 0-100°C

### Chamber Environment

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PC0 | Cavity Thermistor | Analog In | Chamber temperature sensor (NTC 100K 3950) |
| PC8 | Cavity Fan Control | PWM | Chamber cooling fan (inverted: `!`) |
| PC7 | Cavity Fan Tachometer | Digital In | Fan speed monitoring |
| PA10 | Cavity LED | PWM | Chamber lighting (white LED) |

**Cavity Fan**: Variable speed with tachometer feedback
**Temperature Range**: -273.15°C to 999°C (sensor capable)

### Air Purifier System

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PA8 | Purifier Control | PWM | Main purifier fan control (inverted: `!`) |
| PA9 | Purifier Tachometer | Digital In | Primary fan speed (2 PPR) |
| PE15 | Purifier Enable | Digital Out | Purifier power enable |
| PA6 | Extra Fan Tachometer | Digital In | Secondary fan monitoring |
| PA7 | Power Detection | Analog In | Purifier power status (threshold: 0.88V) |

**Purifier**: Dual-fan system with power detection
**Tachometer Poll Interval**: 0.5ms

### Power System Monitoring

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB7 | Power Loss Detection | Digital In | AC power loss detection |
| PC1 | Current Sensor | Analog In | ADC current measurement (0.01Ω sense, 33.333× scale) |

**Power Loss Detection**:
- Trigger Time: 22ms
- Debounce Threshold: 20 events
- Duty Threshold: 0.5
- Type Confirmation: 3 samples

### Main Board Cooling

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PC9 | Power Fan | PWM | Driver cooling fan (inverted: `!`) |

**Power Fan**: Temperature-controlled based on extruder temperatures
**Speed Table**:
- Temp > 260°C: 100% speed (3× boost)
- Temp > 45°C: 60% speed
- Shutdown speed: 0%

### Park Detector Pins (Main Board)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB0 | E0 Park Detector | Analog In | Extruder 0 park position (0-470 range) |
| PC4 | E1 Park Detector | Analog In | Extruder 1 park position (0-470 range) |
| PC5 | E2 Park Detector | Analog In | Extruder 2 park position (0-470 range) |
| PB1 | E3 Park Detector | Analog In | Extruder 3 park position (0-470 range) |

**Function**: Detects when extruder toolheads are properly parked

### Filament Feed System (Main Board)

#### Left Feed Module

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PE4 | Ch1 White LED | Digital Out | Port 1 white indicator |
| PE0 | Ch1 Red LED | Digital Out | Port 1 error indicator |
| PD10 | Ch2 White LED | Digital Out | Port 2 white indicator |
| PE1 | Ch2 Red LED | Digital Out | Port 2 error indicator |
| PA3 | Ch1 Port Detect | Analog In | Filament presence (threshold: 0.18V) |
| PA2 | Ch2 Port Detect | Analog In | Filament presence (threshold: 0.18V) |
| PE5 | Ch1 Wheel Tach 1 | Digital In | Feed wheel speed sensor |
| PE6 | Ch1 Wheel Tach 2 | Digital In | Feed wheel speed sensor |
| PD1 | Ch2 Wheel Tach 1 | Digital In | Feed wheel speed sensor |
| PD0 | Ch2 Wheel Tach 2 | Digital In | Feed wheel speed sensor |
| PD7 | Ch1 Motor PWM | PWM | Feed motor control (10ms cycle) |
| PD6 | Ch2 Motor PWM | PWM | Feed motor control (10ms cycle) |
| PD15 | Motor Tachometer | Digital In | Motor speed monitoring (1 PPR) |

**Left Module**: Feeds extruders 1 (e1) and 0 (e0)
**Preload Length**: 950mm
**Load Position**: X=150, Y=5

#### Right Feed Module

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PE3 | Ch1 White LED | Digital Out | Port 1 white indicator |
| PE2 | Ch1 Red LED | Digital Out | Port 1 error indicator |
| PD11 | Ch2 White LED | Digital Out | Port 2 white indicator |
| PE10 | Ch2 Red LED | Digital Out | Port 2 error indicator |
| PA1 | Ch1 Port Detect | Analog In | Filament presence (threshold: 0.18V) |
| PA0 | Ch2 Port Detect | Analog In | Filament presence (threshold: 0.18V) |
| PD12 | Ch1 Wheel Tach 1 | Digital In | Feed wheel speed sensor |
| PD13 | Ch1 Wheel Tach 2 | Digital In | Feed wheel speed sensor |
| PD9 | Ch2 Wheel Tach 1 | Digital In | Feed wheel speed sensor |
| PD8 | Ch2 Wheel Tach 2 | Digital In | Feed wheel speed sensor |
| PD5 | Ch1 Motor PWM | PWM | Feed motor control (10ms cycle) |
| PD4 | Ch2 Motor PWM | PWM | Feed motor control (10ms cycle) |
| PD14 | Motor Tachometer | Digital In | Motor speed monitoring (1 PPR) |

**Right Module**: Feeds extruders 2 (e2) and 3 (e3)
**Preload Length**: 950mm
**Tachometer PPR**: 6 (wheel), 1 (motor)

---

## Host MCU (Linux SBC)

**Connection**: Virtual serial `/tmp/klipper_host_mcu`
**Platform**: RK3588-based SBC (Linux GPIO control)

### System Control

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| gpiochip0/gpio12 | Head Hub Reset | Digital Out | USB hub reset (GPIO0_B4), active low |

**Head Hub Reset**: Controls power/reset to USB hub for extruder MCUs
- Normal operation: HIGH (1)
- Reset: LOW (0)
- Shutdown value: HIGH (1)

### NFC/RFID Reader System

#### SoC-Connected Reader (FM175xx)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| SPI Bus 2, Dev 0 | SPI Communication | SPI | Mode 0, 500kHz max speed |
| gpiochip1/gpio25 | SoC Reader Reset | Digital Out | Reader reset control |
| gpiochip1/gpio27 | RF Channel 1 | Digital Out | Antenna channel select |
| gpiochip1/gpio24 | RF Channel 2 | Digital Out | Antenna channel select |

**SoC Reader Channels**:
- Channel 1: Extruder 3 (e3)
- Channel 2: Extruder 2 (e2)

#### Extra Reader (FM175xx)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| SPI Bus 2, Dev 1 | SPI Communication | SPI | Mode 0, 500kHz max speed |
| gpiochip1/gpio28 | Extra Reader Reset | Digital Out | Reader reset control |

**Extra Reader Channels**:
- Channel 1: Extruder 0 (e0)
- Channel 2: Extruder 1 (e1)

**NFC Protocol**: FM175xx chip (ISO14443A compatible)
**Function**: Reads filament spool RFID tags
**SPI Mode**: 0 (CPOL=0, CPHA=0)

---

## Extruder MCUs (e0-e3)

All four extruder MCUs share identical pin configurations. Each is an independent STM32-based board controlling one hotend.

### Extruder 0 (e0)

**Location**: X=35.0, Y=332.2 (left front position)
**Serial**: `platform-xhci-hcd.0.auto-usb-0:1.3:1.0`
**Grab Direction**: True (towards front)

### Extruder 1 (e1)

**Location**: X=102.7, Y=332.2 (left rear position)
**Serial**: `platform-xhci-hcd.0.auto-usb-0:1.4:1.0`
**Grab Direction**: False (towards rear)

### Extruder 2 (e2)

**Location**: X=170.2, Y=332.2 (right front position)
**Serial**: `platform-xhci-hcd.0.auto-usb-0:1.2:1.0`
**Grab Direction**: False (towards rear)

### Extruder 3 (e3)

**Location**: X=237.7, Y=332.2 (right rear position)
**Serial**: `platform-xhci-hcd.0.auto-usb-0:1.1:1.0`
**Grab Direction**: False (towards rear)

### Common Extruder MCU Pin Configuration

#### Stepper Motor Control

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PA8 | Stepper STEP | Digital Out | Step pulses to extruder motor |
| PA9 | Stepper DIR | Digital Out | Extrusion direction |
| PB2 | Stepper Enable | Digital Out | Driver enable (inverted: `!`) |
| PA15 | TMC2209 UART | UART | Driver communication (address 3) |

**Driver**: TMC2209 (UART)
**Run Current**: 0.8A
**Hold Current**: 0.2A
**Sense Resistor**: 0.150Ω
**Microsteps**: 16
**Rotation Distance**: 4.95mm (Bondtech-style gears)
**Hold Delay**: 0.32768s
**Power Down Time**: ~3s

#### Hotend Heating System

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB5 | Heater Cartridge | PWM | Hotend heater output |
| PA2 | Thermistor | Analog In | Temperature sensor (NTC 100K 3950) |
| PB7 | Heat Switch | Digital Out | Heater power enable (inverted: `!`) |

**Temperature Control**:
- PID: Kp=33.540, Ki=9.317, Kd=30.186
- Range: 0-300°C
- Min Extrude Temp: 170°C
- Smooth Time: 2s
- Overshoot Tolerance: ±10°C

**Heat Switch**: Safety interlock for heater power

#### Part Cooling Fan

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB3 | Fan PWM | PWM | Print cooling fan control |
| PB4 | Fan Tachometer | Digital In | Fan speed monitoring |

**Fan Features** (e0 only):
- Auxiliary cavity fan control (ID: 2)
- Exhaust purifier control (ID: 3)
- Shutdown speed: 0%
- Tachometer poll: 1ms

#### Hotend Cooling Fan

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB0 | Nozzle Fan PWM | PWM | Hotend heatsink fan (inverted: `!`) |
| PA10 | Nozzle Fan Tach | Digital In | Fan speed monitoring |

**Hotend Fan**:
- Temperature-controlled: ON at 45°C
- Full speed: 100%
- Probe speed: 50% (for leveling)
- Tachometer poll: 0.5ms
- Shutdown speed: 0%

#### Inductance Coil Probe (Bed Leveling)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PA0 | Inductance Coil | Analog/PWM | Capacitive frequency sensor |

**Probe Characteristics**:
- Type: LC oscillator frequency measurement
- Z-Offset: -0.05mm (e0), 0mm (e1-e3)
- Frequency Range: 1.2-1.65 MHz
- Trigger Frequency: 300 Hz relative change
- Sample Count: 4 samples per measurement
- Tolerance: 0.020mm (10 retries)
- Probing Speed: 2mm/s
- Calibration Mode: 1 (frequency-based)
- Horizontal Move Z: 100mm (parking height)

**Function**: Non-contact bed distance sensing for mesh leveling and Z-offset calibration

#### Filament Sensor System

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PA1 | Filament Motion Sensor | Analog/Digital In | Detects filament movement |
| PB1 | Park Active Detect | Analog In | Tool park validation (0-47000 range) |
| PA3 | Grab Valid Detect | Analog In | Filament grab validation (0-390 range) |

**Filament Motion Sensor**:
- Analog Range: 0-1504
- Detection Length: 0.5mm
- Action: Pause on runout
- Integration: Links to entangle detection

**Park Detector Integration**:
- Monitors toolhead parking position
- Validates filament loading
- Analog sensing for precise positioning

#### Accelerometer (Input Shaper)

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PA4 | LIS2DW CS | SPI CS | Accelerometer chip select |
| SPI1 bus | LIS2DW SPI | SPI | SPI communication |

**Accelerometer** (e0 only actively used):
- Model: LIS2DW
- Axes Mapping: Y, X, Z (remapped from physical)
- Function: Resonance testing for input shaper calibration
- Probe Point: X=147, Y=154, Z=20
- Frequency Range: 5-100 Hz (step: 10 Hz)

**Input Shaper Settings**:
- X-axis: MZV @ 54 Hz (range: 47.5-62.5 Hz)
- Y-axis: MZV @ 47.5 Hz (range: 43-54 Hz)

#### Power Loss Detection

| Pin | Function | Type | Behavior |
|-----|----------|------|----------|
| PB8 | Power Loss Detect | Digital In | Per-extruder power monitoring |

**Power Loss Detection**:
- Trigger Time: 2ms
- Report Interval: 0s (immediate)
- Type Confirmation: 2 samples
- Function: Detects loss of power to extruder MCU

---

## Communication and Protocols

### Serial Communication

| MCU | Protocol | Baud Rate | Interface |
|-----|----------|-----------|-----------|
| mcu | UART | 460800 | `/dev/ttyS6` hardware serial |
| host | Virtual | N/A | Unix socket `/tmp/klipper_host_mcu` |
| e0-e3 | USB CDC | Variable | USB full-speed (USB 1.1/2.0) |

**Restart Method**: All MCUs use command-based restart (no hardware reset pin)

### SPI Buses

#### Main MCU SPI

| Bus | Devices | Speed | Pins |
|-----|---------|-------|------|
| SPI2 | TMC2240 (X-axis) | Driver speed | PB13 (SCK), PB14 (MISO), PB15 (MOSI) |
| SPI4 | TMC2240 (Y-axis) | Driver speed | PE11 (SCK), PE13 (MISO), PE14 (MOSI) |

#### Host MCU SPI

| Bus | Devices | Speed | Purpose |
|-----|---------|-------|---------|
| SPI2 Dev0 | FM175xx SoC Reader | 500 kHz | NFC reader for e2/e3 |
| SPI2 Dev1 | FM175xx Extra Reader | 500 kHz | NFC reader for e0/e1 |

#### Extruder MCU SPI

| Bus | Devices | Speed | Purpose |
|-----|---------|-------|---------|
| SPI1 | LIS2DW Accelerometer | Standard | Input shaper data (e0) |

### UART Communication

| Device | Type | Address | Purpose |
|--------|------|---------|---------|
| TMC2209 Z | Single-wire | 0 | Z-axis stepper driver |
| TMC2209 Ex | Single-wire | 3 | Extruder stepper drivers |

**UART Pins**: Shared TX/RX single-wire interface

---

## Behavioral Notes

### Homing Behavior

**X/Y Axes**:
- Sensorless homing using TMC2240 stallGuard
- Home current reduced to 0.650A during homing
- Normal run current: 1.2A
- Homing speed: 40mm/s
- Second homing speed: 40mm/s (for precision)
- Homing tolerance: 0.01mm
- Samples: 3 attempts
- Retract distance: 3mm between attempts

**Z Axis**:
- Sensorless homing using TMC2209 stallGuard
- SGTHRS: 110 (stallGuard threshold)
- Homing speed: 10mm/s
- Allows negative Z (-6mm) for below-bed probing

**Homing Sequence**:
1. Enable X stepper, wait 1s
2. Home Y-axis (wait 1.5s for stallGuard clear)
3. Home X-axis (wait 2s for stallGuard clear)
4. Execute diagonal precision homing (CoreXY calibration)
5. Restore run currents

### Bed Mesh Leveling

- **Grid**: 13×13 points (169 measurements)
- **Area**: 3,3 to 267,267 mm
- **Speed**: 200mm/s (5000mm/s² accel)
- **Samples**: 3 per point
- **Lift Speed**: 30mm/s
- **Horizontal Move Z**: 2mm (fast: 0.5mm)
- **Algorithm**: Bicubic interpolation
- **Split Delta**: 0.01mm (mesh segmentation)

### Temperature Control Behavior

**Heated Bed**:
- Dual PID control for different temperature ranges
- PWM cycle: 1 second (smooth heating)
- Report delta: 1s
- Settle delta: 1.5°C

**Hotends**:
- PID control with 2s smoothing
- Automatic shutdown if temp exceeds limits by >10°C
- Minimum extrude temperature: 170°C enforced

### Power Loss Recovery

**Main Board**:
- Detects AC power loss within 22ms
- Saves position every 1000 lines of G-code
- Z-hop to 140°C temperature on power loss
- Automatic retraction/unretraction on resume

**Extruder Boards**:
- Independent power monitoring per MCU
- 2ms detection time
- Immediate shutdown on power loss

### Filament Handling

**Motion Sensing**:
- Detects 0.5mm of missing movement
- Automatic pause on runout
- Analog range: 0-1504 units

**Feed System**:
- Preloads 950mm of filament
- Max 20 loading attempts
- Coil frequency thresholds:
  - Soft: 800 Hz (filament near)
  - Hard: 1500 Hz (filament touching nozzle)

### Multi-Extruder Coordination

**Park Positions**:
- E0: X=35.0, Y=332.2
- E1: X=102.7, Y=332.2
- E2: X=170.2, Y=332.2
- E3: X=237.7, Y=332.2

**Idle Position**: Y=250 (all extruders)

**Tool Change Behavior**:
- Fast move speed: 400mm/s
- Slow move speed: 50mm/s
- Grab speed: 10mm/s
- Horizontal move: 10mm
- Retract distance: 1.5mm
- Switch acceleration: 5000mm/s²

### Safety Features

1. **Thermal Runaway Protection**: Enabled on all heaters
2. **Overshoot Detection**: ±10°C limits
3. **Max Extrude Distance**: 200mm (prevents runaway extrusion)
4. **Stepper Timeout**: Motors disable after idle timeout
5. **Power Loss Detection**: Multi-level monitoring
6. **Heater Interlocks**: Heat switches control power to cartridges

---

## Firmware Configuration Notes

### Key Settings

- **Max Velocity**: 500mm/s
- **Max Acceleration**: 20,000mm/s²
- **Square Corner Velocity**: 8mm/s
- **Max Z Velocity**: 30mm/s
- **Max Z Accel**: 500mm/s²
- **Idle Timeout**: 300s (5 minutes)
- **Pause Timeout**: 9999999999s (effectively infinite)

### Special Features

- **Exclude Object**: Enabled (cancel individual print objects)
- **Arc Support**: G2/G3 with 0.5mm resolution
- **Firmware Retraction**: Configured per extruder
- **Input Shaping**: MZV algorithm for X/Y
- **Resonance Testing**: Via LIS2DW accelerometer
- **Time-lapse**: 24 FPS video capture support

---

*Document generated from printer.cfg and Klipper source analysis with the help of Claude*
*Last updated: 2025-12-05*
