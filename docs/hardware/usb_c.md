# USB-C ESP32-S3 Interface Board Documentation

## Overview
This USB-C interface board provides power delivery and programming connectivity for an ESP32-S3 microcontroller. The design includes comprehensive protection, proper USB-C device configuration, and power source selection for both development and production environments.

## Main Components

### USB-C Connector (J19)
- **Part Number**: TYPE_C_31_M_12
- **Configuration**: USB 2.0 device mode
- **Power Capability**: 5V delivery via USB-C VBUS
- **Data Support**: Full-speed USB communication and programming

### Power Management System

#### Power Source Selection (J17)
- **Component**: 2.54mm jumper header (2.54-1*2P)
- **Function**: Selects between USB power and external power isolation
- **Jumper Installed**: ESP32-S3 system powered by USB 5V
- **Jumper Removed**: Programming-only mode (no USB power delivery)

#### Battery Power Isolation
- **Method**: Manual jumper control of TPS4800-Q1 INP pin (Pin 3)
- **Implementation**: 2-pin header interrupts connection between VIN and INP
- **USB Mode**: Remove battery jumper → INP floating → TPS4800 disabled → battery path isolated
- **Battery Mode**: Install battery jumper → INP connected to VIN → normal battery operation

### USB-C Configuration

#### Configuration Channel (CC) Setup
- **CC1 Pin (A5)**: 5.1kΩ pull-down resistor to GND
- **CC2 Pin (B5)**: 5.1kΩ pull-down resistor to GND
- **Result**: USB device mode advertisement to host
- **Compliance**: USB-C specification for sink device identification

#### Data Line Implementation
- **USB D+ (A6/B6)**: Connected through 22Ω series resistor and ESD protection
- **USB D- (A7/B7)**: Connected through 22Ω series resistor and ESD protection
- **Destination**: ESP32-S3 GPIO19 (D-) and GPIO20 (D+)
- **Signal Integrity**: 22Ω resistors provide impedance matching and current limiting

#### ESD Protection
- **Component**: Dual low-capacitance TVS diode array (e.g., PESD5V0S2UT)
- **Placement**: Between series resistors and ESP32-S3 data pins
- **Specifications**: <1pF capacitance, ~6V clamping voltage
- **Protection**: Guards against electrostatic discharge on exposed USB connector

### ESP32-S3 Interface

#### USB Connection
- **ESP32-S3 D+**: GPIO20 (dedicated USB OTG pin)
- **ESP32-S3 D-**: GPIO19 (dedicated USB OTG pin)
- **USB Protocol**: Native USB OTG support in ESP32-S3
- **Programming**: CDC/ACM or vendor-specific protocol over USB

#### Power Connection
- **5V Rail**: Delivered from USB VBUS when J17 jumper installed
- **Regulation**: 5V → 3.3V via TLV62569DBV buck converter (existing system)
- **ESP32-S3 Power**: 3.3V rail powers ESP32-S3 and peripherals

## Pin Mapping Summary

| USB-C Pin | Signal | Series R | ESD Protection | ESP32-S3 Pin | Function |
|-----------|--------|----------|----------------|--------------|----------|
| A4/B4 | VBUS | - | - | - | 5V power via J17 |
| A5 | CC1 | - | - | - | 5.1kΩ to GND (device mode) |
| B5 | CC2 | - | - | - | 5.1kΩ to GND (device mode) |
| A6/B6 | D+ | 22Ω | TVS array | GPIO20 | USB data positive |
| A7/B7 | D- | 22Ω | TVS array | GPIO19 | USB data negative |
| A8/B8 | SBU1/SBU2 | - | - | - | Not connected |
| Shield | SHIELD | - | - | - | Chassis ground |

## Power System Integration

### Power Flow Architecture
1. **USB Mode**: USB VBUS → J17 → 5V rail → 3.3V buck → ESP32-S3
2. **Battery Mode**: Battery → TPS4800 → 5V buck → 3.3V buck → ESP32-S3
3. **Isolation**: Never both sources simultaneously (prevented by jumper design)

### Power Consumption (USB Mode)
| Component | Current | Rail | Power |
|-----------|---------|------|-------|
| ESP32-S3 (active) | 80mA | 3.3V | 264mW |
| ESP32-S3 (sleep) | 10µA | 3.3V | 33µW |
| Support circuits | ~26mA | Mixed | ~130mW |
| **Total USB current** | **~400mA** | **5V** | **~2W** |

### Power Source Selection Matrix
| Battery Jumper (INP) | USB Jumper (J17) | Result | Battery Current | USB Current |
|----------------------|-------------------|---------|-----------------|-------------|
| Installed | Removed | Battery only | 88µA + load | 0mA |
| Removed | Installed | USB only | 0µA | ~400mA |
| Removed | Removed | No power | 0µA | 0mA |
| **Installed** | **Installed** | **⚠️ CONFLICT** | **AVOID** | **AVOID** |

## Programming and Operation

### Programming Mode
- **Connection**: Standard USB-C cable to development host
- **Protocol**: Native USB programming via ESP32-S3 USB OTG
- **Boot Mode**: Manual - hold GPIO0 low during reset for programming
- **Auto-reset**: Not implemented (manual reset required)
- **Power**: Via USB (J17 installed) or external power (J17 removed)

### Production Operation
- **Power Source**: Typically battery power (USB jumpers removed)
- **USB Connector**: Can remain populated for field programming
- **ESD Protection**: Protects ESP32-S3 from connector exposure
- **Serviceability**: Manual jumper control allows power source selection

## Design Features

### Protection Systems
- **ESD Protection**: TVS diodes on USB data lines
- **Power Isolation**: Jumper-controlled source selection
- **Overcurrent**: Inherent USB host current limiting
- **Signal Integrity**: Series termination resistors

### Production Considerations
- **Component Selection**: Standard values for easy sourcing
- **Manual Programming**: Acceptable for production programming workflow  
- **Power Flexibility**: Supports both development and deployment scenarios
- **EMI Compliance**: ESD protection aids regulatory compliance

### User Interface
- **J17 Power Jumper**: Controls USB power delivery to system
- **INP Control Jumper**: Controls battery power path isolation  
- **Programming**: Standard USB-C connector with manual boot control
- **Status**: Power source clearly defined by jumper positions

## Operational Modes

### Development Mode
1. Install J17 jumper (USB power)
2. Remove battery jumper (isolate battery)  
3. Connect USB-C cable
4. Manual reset/GPIO0 control for programming

### Production Mode  
1. Remove J17 jumper (no USB power)
2. Install battery jumper (battery power)
3. USB connector available for field programming if needed

### Programming Mode (Either Power Source)
1. Connect USB-C cable for data
2. Set power jumpers as desired
3. Hold GPIO0 low, press reset
4. Release reset, then GPIO0
5. Program via USB interface

This design provides a robust, production-ready USB-C interface with comprehensive protection and flexible power management for ESP32-S3 based systems.