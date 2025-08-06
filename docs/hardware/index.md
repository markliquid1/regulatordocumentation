# Hardware Overview

## System Architecture

The Xengineering alternator regulator is a marine-grade open-source battery management and charging control system built around an ESP32-S3 microcontroller. The design emphasizes ultra-low power consumption, wide input voltage range, and robust protection for harsh marine environments.

### Core Specifications

| Parameter | Specification | Notes |
|-----------|---------------|-------|
| Input Voltage | 4.2V - 65V | Via TPS4800-Q1 protection |
| Typical Operating Voltage | 12V/24V/48V | Marine/automotive systems |
| Operating Temperature | -40°C to +85°C | Industrial grade components |
| Field Current Control | Up to 15A | PWM controlled |
| Current Measurement | ±500A capability | Via external shunt |

## Power System

### Multi-Stage Power Architecture
```
Battery (4.2-65V) → TPS4800-Q1 → LMR36510 (5V) → TLV62569 (3.3V) → ESP32-S3
```

**Key Features:**
- **Primary Protection:** TPS4800-Q1 high-side driver with overcurrent, overvoltage, and reverse polarity protection
- **Dual Buck Converters:** High-efficiency switching supplies (87-94% overall efficiency)
- **USB-C Alternative:** Programming and development power via USB-C with jumper selection
- **Power Isolation:** Complete battery disconnect via jumper control for USB operation

## Processing Core

### ESP32-S3 Controller
- **Architecture:** Dual-core Xtensa LX7 @ 240MHz
- **Memory:** 16MB Flash, 8MB PSRAM
- **Connectivity:** WiFi 802.11b/g/n, native USB OTG
- **I/O Capabilities:** Sufficient GPIO for all analog inputs, digital I/O, communications
- **Programming:** USB-C native programming (no USB-to-serial converter required)

**Selection Rationale:**
- Native USB support eliminates external converters
- Ample GPIO and processing power for real-time control
- Integrated WiFi for remote monitoring and configuration
- Low-power sleep modes for battery applications

## Input/Output Systems

### Analog Inputs (ADS1115 + TLV9154)
4-channel precision measurement system with op-amp buffering:

| Channel | Function | Range | Resolution | Power Consumption |
|---------|----------|-------|------------|------------------|
| 0 | Battery Voltage | 0.21V - 65.2V | 2.1mV | 22.5µA @ 12V input |
| 1 | Alternator Current | ±200A (via hall sensor) | 0.02A theoretical | 10.8µA |
| 2 | Engine RPM | 8Hz - 2640Hz | Variable by setup | 10.8µA |
| 3 | Temperature | 15°C - 125°C* | 0.8-5.8°C | 115µA @ 25°C |

*Limited by 5V supply configuration - see documentation for details

**Design Features:**
- High-impedance voltage dividers minimize power consumption
- Unity-gain op-amp buffers prevent loading errors
- Single 3.3V supply with inherent overvoltage protection
- 16-bit ADC resolution with 0.1mV LSB sensitivity

### Digital Inputs
Two architectures for different applications:

**Optocoupler Style (6 channels):**
- Activation input range: 5V - 54V
- Power consumption: 0.16mA - 33.7mA @ 12V equivalent (when active)
- Applications: Any on/off voltage signals

**Ground Style (4 channels):**
- Activation: Short to ground
- Continuous power: 0.09mA @ 12V equivalent
- RC debounce filtering with ESD protection
- Applications: Switch inputs, mode selections

### Digital Outputs

**Buzzer/Relay Control:**
- High-side P-channel MOSFET switching
- 5V supply, up to 500mA recommended
- Flyback protection for inductive loads
- 0.35mA idle current @ 12V equivalent

### Communication Interfaces

**UART Isolation (2 channels):**
- HCPL2531 optocoupler isolation (2500V)
- NMEA0183 and VE.Direct compatible
- 1.86mA @ 12V equivalent power consumption
- Up to 1MBaud data rates

**NMEA2000 CAN:**
- Isolated CAN transceiver (ISO1050DUB)
- 12V boat power to isolated 5V conversion
- Selectable 120Ω termination
- 20-25mA standby current from 12V boat supply

**Pressure/Temperature Sensor:**
- BMP390 barometric p/t sensor
- I2C interface, 3.3V supply

## Field Control System

### Alternator Field Driver
High-current PWM control system:

**Architecture:**
```
5V → MT3608 Boost (12V) → LM5109A Driver → MOSFET (~15A max)
```

**Specifications:**
- Field current: Up to 15A continuous
- PWM frequency: 100Hz - 20kHz+
- Flyback protection for inductive field loads
- Bootstrap driver for high-side switching

**Current Limitations:**
- MT3608 boost limited by BAT54J diode (200mA continuous)
- Thermal management required for high-frequency operation
- Parallel MOSFETs may be unnecessary , planning to remove 1 in test

## Battery Monitoring

### INA228 Current/Voltage Monitor
High-precision battery state monitoring:

**Configuration:**
- Bus voltage: Direct battery measurement (up to 85V rating)
- Current: Low-side shunt measurement (±163.84mV differential range)
- Interface: I2C at address 0x40
- Resolution: 16-bit with programmable gain

**Protection:**
- P6SMB68A TVS protection (58.1V standoff for 48V systems)
- RC filtering on all inputs (1.59kHz cutoff)
- ESD protection diodes on differential inputs

**Power Consumption:**
- Normal operation: 3.63mW (1.1mA @ 3.3V)
- Shutdown mode: 9.24µW (2.8µA)

### Tachometer Output
Signal conditioning for dashboard tachometer:

- Input: ESP32 frequency signal (8Hz - 2.64kHz)
- Output: Battery voltage level square wave
- Drive capability: Standard automotive tachometers
- Power: 0.54mA - 13.6mA @ 12V (depending on supply voltage and duty cycle)

## Protection Systems

### Input Protection (TPS4800-Q1)
Comprehensive automotive-grade protection:

**Protection Features:**
- Reverse polarity: -65V maximum
- Overvoltage: 58.8V - 61.5V shutdown threshold
- Overcurrent: 15.1A with 3µs response time
- Undervoltage lockout: 6.8V turn-on threshold
- Auto-retry: 5-second intervals after fault

**Power Management:**
- 43µA quiescent current when enabled
- Complete power path isolation via jumper control
- Hierarchical TVS protection for severe transients

### ESD and Transient Protection
Multi-layer protection throughout system:

- TVS diodes on power inputs and high-voltage signals
- ESD protection on all exposed I/O
- RC filtering before sensitive analog inputs
- Bidirectional TVS arrays on communication lines

## PCB Considerations

### Design Requirements
- 4-layer PCB recommended for thermal management and noise reduction
- Wide traces for high-current paths (field driver, power supplies)
- Proper ground plane separation for isolated circuits
- Thermal vias for power-dissipating components
- Marine-grade conformal coating compatibility

### Component Selection
- Industrial temperature range (-40°C to +85°C)
- High-reliability automotive-grade components where applicable
- Standard values (1% resistors, common capacitor values) for easy sourcing
- AEC-Q200 qualified components for critical protection circuits

## Development and Programming

### USB-C Interface
Native programming and debugging:

- ESP32-S3 native USB OTG (no USB-to-serial converter)
- Power delivery capability with jumper control
- ESD protection on data lines
- Manual boot control (GPIO0 + reset) for programming

### Power Source Management
Jumper-controlled power source selection:

| Battery Jumper | USB Jumper | Result | Use Case |
|----------------|------------|---------|-----------|
| Installed | Removed | Battery only | Production operation |
| Removed | Installed | USB only | Development/programming |
| Removed | Removed | No power | Safe configuration |

## System Integration

### Typical Application Connections
- Battery positive/negative for monitoring and power
- External shunt resistor for current measurement (75-150µΩ, 500A rating)
- Alternator field winding connection
- Alternator current sensor (included hall effect clamp)
- Engine RPM signal (alternator stator or other frequency source)
- Temperature sensor (10kΩ NTC thermistor or Digital sensor (included))
- Digital inputs for engine/system status
- NMEA2000 connection for marine network integration
- VeDirect and/or NMEA0183 as needed

### Performance Summary
- Ultra-low quiescent current suitable for continuous battery monitoring
- High-efficiency power conversion minimizes heat generation
- Robust protection systems handle marine electrical environment
- Comprehensive I/O capabilities for complete engine/charging system monitoring

This hardware platform provides a complete solution for intelligent alternator regulation and battery management in marine and automotive applications, with emphasis on reliability, efficiency, and comprehensive monitoring capabilities.