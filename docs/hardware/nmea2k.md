# NMEA2000 CAN Transceiver Interface - Engineering Design Summary

## Circuit Overview
This circuit provides a galvanically isolated NMEA2000 CAN interface for an ESP32-S3 microcontroller. The design converts 12V boat power to isolated 5V, then uses an isolated CAN transceiver to communicate with the NMEA2000 network while protecting the microcontroller from electrical noise and ground potential differences common in marine environments.

## Power Supply Section

### Input Protection
- **BOAT_12V Input**: Raw 12V from vessel electrical system
- **D16 (1N5819WS)**: Schottky diode for reverse polarity protection
- **D17 (SMBJ16CA)**: Bidirectional TVS diode for transient/surge protection
- **C7 (10µF)**: Input filter capacitor

### Isolated DC-DC Converter (U11 - MCP3610AGQV-P)
- **Function**: Isolated buck converter generating 5V_ISOLATED from protected 12V input
- **Isolation**: Galvanic isolation between input (boat 12V system) and output (vessel CAN ground)
- **Switching Elements**: Internal power MOSFETs on pins 6, 5, 4, 19 (SW nodes)
- **Bootstrap**: Pin 11 (BST) provides gate drive for high-side MOSFET
- **Power Good**: Pin 10 (PG) indicates output regulation status

### Voltage Control Circuit
- **FB_MPM**: Feedback voltage divider output that sets the target voltage for U11 buck converter
- **R40 (100k, 1%)**: Upper feedback resistor
- **R17 (19.1k, 1%)**: Lower feedback resistor to ISOLGND_CAN
- **Target Voltage**: Vout = Vref × (1 + R40/R17) where Vref is internal reference of MCP3610AGQV

### Output Stage
- **5V_ISOLATED**: Regulated isolated 5V output
- **C36 (22µF, 16V)**: Primary output filter capacitor
- **C35 (1µF)**: Secondary filter capacitor
- **ISOLGND_CAN**: Isolated ground referenced to vessel CAN ground

## CAN Transceiver Section

### Isolated CAN Transceiver (U7 - ISO1050DUB)
- **Function**: Provides 2.5kV galvanic isolation between ESP32-S3 and CAN bus
- **Package**: 8-pin SOIC with isolation barrier between pins 1-4 and 5-8

### Power Connections
- **Pin 1 (VCC1)**: 3V3 supply from ESP32-S3 side (non-isolated)
- **Pin 4 (GND1)**: Connected to system ground (GND)
- **Pin 8 (VCC2)**: Connected to 5V_ISOLATED
- **Pin 5 (GND2)**: Connected to ISOLGND_CAN (vessel ground)

### Signal Connections
- **Pin 2 (RXD)**: CAN receive data output to ESP32-S3 (RXDCAN signal)
- **Pin 3 (TXD)**: CAN transmit data input from ESP32-S3 (TXDCAN signal)
- **Pin 7 (CANH)**: CAN High differential signal
- **Pin 6 (CANL)**: CAN Low differential signal

## CAN Bus Physical Interface

### Standard CAN Connector (U18)
- **Function**: Standard NMEA2000 CAN cable connector
- **CANP**: CAN Positive signal
- **CANN**: CAN Negative signal
- **Connection**: Differential 120Ω impedance CAN bus

### Bus Termination
- **R37 (120Ω)**: CAN bus termination resistor
- **J15**: Jumper connection (2.54-1*2P)
  - Pin 1: Connected to CANP
  - Pin 2: Connected to CANN (through 120Ω termination)
- **Usage**: Short J15 if this device is the last node on the CAN bus

## ESP32-S3 Interface

### Digital Signals
- **TXDCAN**: CAN transmit data output from ESP32-S3 to U7 pin 3
- **RXDCAN**: CAN receive data input to ESP32-S3 from U7 pin 2
- **3V3**: Power supply for MCU side of ISO1050DUB (from ESP32-S3 board)

### Control/Monitoring
- **FB_MPM**: Voltage divider feedback signal to U11 for output voltage control
- **GND**: System ground connection to ESP32-S3 board

## Power Consumption Analysis

### Standby Conditions (CAN bus idle, no traffic)
- **MCP3610AGQV (U11)**: ~2-5mA quiescent current
- **ISO1050DUB (U7)**: ~15-20mA total (both sides)
- **Total Standby**: ~20-25mA from 12V input

### Active Conditions (Normal CAN traffic)
- **MCP3610AGQV (U11)**: ~2-5mA + load current
- **ISO1050DUB (U7)**: ~20-25mA during transmission
- **Load Current**: Depends on 5V_ISOLATED rail loading
- **Total Active**: ~25-30mA from 12V input

### Sleep Mode (if ESP32-S3 enables low power)
- **MCP3610AGQV (U11)**: Can be shut down via enable control
- **ISO1050DUB (U7)**: Has standby mode capability
- **Potential Sleep Current**: <1mA if properly controlled

## Circuit Operation

### Power Conversion Flow
1. BOAT_12V → D16 (reverse protection) → D17 (surge protection) → U11 input
2. U11 buck converter creates 5V_ISOLATED referenced to vessel CAN ground
3. FB_MPM voltage divider sets target output voltage for regulation loop
4. 5V_ISOLATED powers vessel-side of U7 transceiver
5. 3V3 from ESP32-S3 board powers MCU-side of U7 transceiver

### CAN Communication Flow
1. **Transmit**: ESP32-S3 → TXDCAN → U7 isolation → CANH/CANL → U18 connector
2. **Receive**: U18 connector → CANH/CANL → U7 isolation → RXDCAN → ESP32-S3

### Ground Isolation
- **ESP32-S3 Ground**: Referenced to microcontroller board ground
- **ISOLGND_CAN**: Referenced to vessel CAN network ground
- **Isolation**: Complete galvanic isolation prevents ground loops

## Key Specifications
- **Input Voltage**: 12V nominal (9-16V operating range typical)
- **Isolation Voltage**: 2.5kV (ISO1050DUB specification)
- **CAN Standard**: NMEA2000 compliant differential signaling
- **Connector**: Standard NMEA2000 CAN connector (U18)
- **Termination**: Selectable 120Ω via J15 jumper
- **Power Efficiency**: >80% typical for MCP3610AGQV buck converter

## Component Summary
- **U11 (MCP3610AGQV-P)**: Isolated buck converter (12V → 5V)
- **U7 (ISO1050DUB)**: Isolated CAN transceiver
- **U18**: Standard NMEA2000 CAN connector
- **D16**: Reverse polarity protection diode
- **D17**: Surge protection TVS diode
- **R37**: 120Ω CAN termination resistor
- **J15**: Termination enable jumper