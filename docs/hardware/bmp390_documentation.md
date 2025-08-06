# BMP390 Pressure Sensor Interface Circuit Documentation

## Overview
Interface circuit for BMP390 digital barometric pressure sensor with I2C communication protocol and power conditioning. The sensor is configured for I2C operation with address selection via SDO pin.

## Bill of Materials
| Reference | Component | Value | Package | Description |
|-----------|-----------|-------|---------|-------------|
| U14 | BMP390 | - | HLGA-10 | Barometric pressure sensor |
| C61 | Capacitor | 1µF | 0603 | VDD decoupling capacitor |
| C62 | Capacitor | 1µF | 0603 | VDDIO decoupling capacitor |
| R81 | Resistor | 10kΩ | 0603 | CSB pull-up resistor |

## Pinout and Connections

### BMP390 (U14) Pin Configuration
| Pin | Symbol | Function | Connection | Notes |
|-----|--------|----------|------------|-------|
| 1 | VDDIO | I/O Supply | 3.3V via C61 | I2C interface power |
| 2 | SCK | Serial Clock | I2C_SCL | I2C clock input |
| 3 | VSS | Ground | GND | Ground reference |
| 4 | SDI | Serial Data In | I2C_SDA | I2C data line |
| 5 | SDO | Serial Data Out | GND | I2C address select (0x76) |
| 6 | CSB | Chip Select | 3.3V via R81 | I2C mode enable (HIGH) |
| 7 | INT | Interrupt | DNC | Do Not Connect per datasheet |
| 8 | VSS | Ground | GND | Ground reference |
| 9 | VDD | Core Supply | 3.3V via C62 | Main power supply |
| 10 | GND | Ground | GND | Ground reference |

## Power Supply Configuration
- **Supply Voltage**: 3.3V common rail for both VDD and VDDIO
- **VDD Range**: 1.8V - 3.6V (core supply)
- **VDDIO Range**: 1.2V - 3.6V (I/O supply)
- **Decoupling**: 1µF ceramic capacitors on both supply pins

## Power Consumption
| Operating Mode | Typical Current | Supply Voltage | Power |
|----------------|-----------------|----------------|-------|
| Sleep Mode | 0.2 µA | 3.3V | 0.66 µW |
| Standby Mode | 1.8 µA | 3.3V | 5.94 µW |
| Normal Mode @ 1Hz | 3.4 µA | 3.3V | 11.2 µW |
| Forced Mode (single conversion) | ~500 µA | 3.3V | 1.65 mW |
| Peak Current (during conversion) | 714 µA | 3.3V | 2.36 mW |

**Additional Power Notes:**
- **Startup Current**: Brief spike during power-on (~1mA for ~2ms)
- **I2C Activity**: Minimal additional current during communication
- **Pull-up Resistor (R81)**: 3.3V/10kΩ = 330µA when CSB active

## I2C Interface Setup
- **Protocol**: I2C slave device
- **Slave Address**: 0x76 (SDO tied to GND)
- **Alternative Address**: 0x77 (if SDO tied to VDDIO)
- **Bus Speed**: Supports 100kHz (standard) and 400kHz (fast mode)
- **Pull-up Resistors**: Implemented elsewhere in system on SCL/SDA lines
- **Mode Selection**: CSB pulled high via R81 enables I2C communication

## Interface Signals
- **I2C_SCL**: Serial clock line to microcontroller
- **I2C_SDA**: Serial data line to microcontroller  
- **BMP390_INT**: Interrupt output (currently unused, DNC per datasheet)

## Design Notes
- **Thermal Placement**: Component positioned away from heat sources for accurate readings
- **Single Supply**: VDD and VDDIO connected to same 3.3V rail (datasheet compliant)
- **Address Configuration**: SDO grounded for 7-bit address 0x76
- **Unused Pins**: INT pin left disconnected as recommended by datasheet
- **Communication Mode**: CSB high selects I2C over SPI interface

## Operational Characteristics
- **Pressure Range**: 300 - 1250 hPa (equivalent to +9000m to -500m altitude)
- **Temperature Range**: -40°C to +85°C
- **Resolution**: 0.0016 hPa (0.013m altitude resolution)
- **Interface**: I2C up to 400kHz
- **Power Modes**: Sleep, forced, and normal measurement modes

## Register Access
- **Default State**: Sleep mode after power-on reset
- **Calibration**: Factory calibration coefficients stored in NVM
- **Configuration**: All sensor parameters configurable via I2C registers
- **Data Format**: 20-bit pressure, 20-bit temperature values
