# Input Protection for most of the board

## Device Overview
- **Part Number**: TPS4800-Q1
- **Package**: DGX (VSSOP, 19-pin)
- **Function**: 100V Automotive Low IQ High Side Driver with Protection
- **Operating Voltage Range**: 6-60V (application specific)
- **Typical Application Voltage**: 12V automotive systems
- **Key Features**: Reverse polarity protection to -65V, integrated charge pump, comprehensive fault detection

## Pin-by-Pin Configuration

### EN/UVLO (Pin 1) - Enable/Undervoltage Lockout
- **Threshold**: 1.24V rising, 1.14V falling (±4% tolerance)
- **Voltage Divider**: R1=470kΩ, R2+R3=105kΩ (total 575kΩ)
- **Turn-On Voltage**: 6.8V battery minimum
- **Turn-Off Voltage**: 6.25V battery minimum
- **At 12V Battery**: EN/UVLO pin = 2.19V (well above 1.24V threshold)
- **Current Draw**: 12V / 575kΩ = 20.9µA
- **Rationale**: Provides undervoltage lockout protection, keeps device off during deep battery discharge

### OV (Pin 2) - Overvoltage Protection
- **Threshold**: 1.225V rising ±4.37% (1.171V to 1.278V)
- **Design Requirement**: 58V minimum overvoltage protection (48V lithium system)
- **Voltage Divider**: R_top=499kΩ, R_bottom=10.2kΩ (total 509.2kΩ)
- **OVP Trigger Range**: 58.5V to 63.8V (accounting for threshold tolerance)
- **At 12V Battery**: OV pin = 0.240V (well below 1.171V minimum threshold)
- **Current Draw**: 12V / 509.2kΩ = 23.6µA
- **At 60V Battery**: Current = 117.8µA
- **Component Values**: Both 499kΩ and 10.2kΩ are standard 1% resistors

### INP (Pin 3) - Input Control Signal **[Modified for USB-C Power Option]**
- **Thresholds**: 2V high, 0.8V low
- **Configuration**: Jumper-controlled connection to VIN (6-60V)
- **Implementation**: 2-pin jumper header between VIN and INP pin
- **Battery Mode (Jumper Installed)**: INP = VIN, device enabled when battery voltage present
- **USB Mode (Jumper Removed)**: INP floating → internal 100nA pull-down → device disabled
- **At 12V Battery**: INP = 12V (well above 2V HIGH threshold)
- **Behavior**: Manual control - MOSFET enabled only when jumper installed
- **Absolute Maximum**: 70V (our 60V max is within limits)
- **Power Isolation**: Complete battery power path disconnect when jumper removed

### ISCP (Pin 8) - Short Circuit Protection Threshold
- **Design Requirement**: 15A overcurrent protection
- **Sense Resistor**: 2mΩ (HoLLR2010-1.5W-2mR-1%, 2W, AEC-Q200)
- **Calculation**: RISCP = (15A × 0.002Ω - 19mV) / 2µA = 5.5kΩ
- **Selected Value**: 5.6kΩ (1%, standard value)
- **Actual Protection Level**: 15.1A
- **Voltage Across Sense Resistor at Trip**: 30.2mV
- **At 12V Normal Operation**: No effect on ISCP pin (current dependent)
- **Note**: EVM demo board values appear inconsistent with datasheet formula

### TMR (Pin 9) - Fault Timer
- **Design Requirement**: ~5 second auto-retry interval
- **Selected Value**: 220nF ceramic capacitor to GND
- **Short-Circuit Response Time**: tSC = (220nF × 1.1V) / 80µA = **3.0µs**
- **Auto-Retry Interval**: tRETRY = 22.7 × 10⁶ × 220nF = **5.0 seconds**
- **Behavior**: 
  - Fault detected → 3µs delay → FET turns OFF
  - Auto-retry after 32 charge/discharge cycles (5 seconds)
  - Continues retrying until fault clears
- **At 12V Battery**: Timer behavior independent of supply voltage

## Power Source Integration

### USB-C Power Mode (INP Jumper Removed)
- **USB Power Selection**: J17 jumper on USB-C board enables 5V from VBUS
- **Battery Isolation**: Remove INP jumper → TPS4800-Q1 disabled → complete battery power path isolation
- **Power Flow**: USB 5V → existing 3.3V buck (TLV62569DBV) → ESP32-S3 + peripherals
- **Battery Current**: ~44µA (only voltage divider leakage, TPS4800 disabled)
- **USB Current**: ~400mA @ 5V for full system operation
- **5V Buck Safety**: Safe to apply USB 5V to LMR36510ADDA output when TPS4800 input isolated

### Battery Power Mode (INP Jumper Installed)  
- **Battery Power Selection**: Remove J17 jumper (isolate USB power from 5V rail)
- **Battery Enable**: Install INP jumper → TPS4800-Q1 enabled
- **Power Flow**: Battery → TPS4800-Q1 → LMR36510ADDA (5V buck) → TLV62569DBV (3.3V buck) → ESP32-S3
- **Battery Current**: 87µA quiescent + load current
- **USB Current**: 0mA (USB power isolated)

### Jumper Control Summary
| Jumper | Location | Function | Installed | Removed |
|--------|----------|----------|-----------|---------|
| **INP Jumper** | TPS4800-Q1 board | Battery enable | Battery power ON | Battery power OFF |
| **J17 Jumper** | USB-C board | USB power | USB power ON | USB power OFF |

**Power Source Selection Matrix:**
| INP Jumper | J17 Jumper | Result | Battery Current | USB Current |
|------------|------------|---------|-----------------|-------------|
| Installed | Removed | Battery only | 87µA + load | 0mA |
| Removed | Installed | USB only | 44µA | ~400mA |
| Removed | Removed | No power | 44µA | 0mA |
| **Installed** | **Installed** | **⚠️ CONFLICT** | **AVOID** | **AVOID** |

## Input Protection Design

### Transient Protection Strategy
The design implements a **hierarchical protection approach** to handle automotive transients while preserving proper overvoltage protection functionality:

1. **Primary Protection**: TPS4800-Q1 OV pin triggers controlled shutdown at 58.5V-63.8V
2. **Secondary Protection**: External TVS diodes clamp severe transients above 64V
3. **Energy Storage**: Input capacitors absorb transient energy during brief spikes

### TVS Diode Configuration (on TPS4800-Q1 input)
**Bidirectional TVS Clamp Bridge (Cathodes Tied Together):**

**D1 (Positive Rail Protection):**
- **SMBJ64A** (64V unidirectional TVS)
- **Connection**: Anode to GND, Cathode to shared node
- **Breakdown Range**: 60.8V-67.2V
- **Soft Conduction**: Begins ~65V (above OV protection range)
- **Function**: Clamps positive transients >64V to ~76-103V range

**D7 (Negative Rail Protection):**
- **SMBJ75A** (75V unidirectional TVS) 
- **Connection**: Anode to VIN, Cathode to shared node
- **Breakdown Range**: 71.3V-83.3V
- **Redundant Protection**: Backs up TPS4800-Q1's internal -65V protection
- **Function**: Clamps negative transients <-75V to ~-90 to -121V range

**Shared Cathode Node**: Creates floating reference point for bidirectional clamping

**Topology Benefits:**
- **Positive overvoltage**: D1 avalanche breakdown clamps high VIN
- **Negative overvoltage**: D7 avalanche breakdown clamps low VIN  
- **Reverse polarity**: Both diodes forward conduct (~1.4V drop), creating controlled short to trip upstream protection
- **No interference**: TVS breakdown voltages above TPS4800-Q1 OVP threshold

### Protection Hierarchy Operation
- **6V-58V Normal**: No protection active, normal operation
- **58.5V-63.8V**: OV protection triggers, controlled MOSFET shutdown
- **64V+**: TVS conducts, absorbs transient energy, prevents chip damage
- **Negative spikes**: TVS and internal protection both active

## Component Summary

| Component | Value | Tolerance | Type | Function |
|-----------|-------|-----------|------|----------|
| R1 (EN/UVLO top) | 470kΩ | 1% | Resistor | UVLO voltage divider |
| R2+R3 (EN/UVLO bottom) | 105kΩ | 1% | Resistor | UVLO voltage divider |
| R_OV_top | 499kΩ | 1% | Resistor | OVP voltage divider |
| R_OV_bottom | 10.2kΩ | 1% | Resistor | OVP voltage divider |
| RISCP | 5.6kΩ | 1% | Resistor | Short-circuit threshold |
| CTMR | 220nF | - | Ceramic Cap | Fault timer |
| CBST | 100nF | - | Ceramic Cap | Bootstrap capacitor |
| RSNS | 2mΩ | 1% | Power Resistor | Current sensing (2W, AEC-Q200) |
| D_POS (TVS) | SMBJ64A | - | TVS Diode | Positive transient protection |
| D_NEG (TVS) | SMBJ75A | - | TVS Diode | Negative transient protection |
| C_INPUT | 11µF total | - | Capacitors | Input energy storage |
| **INP Jumper** | **2-pin header** | - | **Jumper** | **Battery power isolation control** |

## Performance Analysis

### Battery Mode Operation (INP Jumper Installed)

| Parameter | Value | Notes |
|-----------|-------|-------|
| EN/UVLO Current | 20.9µA | Through voltage divider |
| OV Current | 23.6µA | Through voltage divider |
| Total Divider Current | 44.5µA | At 12V battery |
| Device Quiescent Current | 43µA typical | From datasheet |
| Total System Current | ~87µA | Excluding load current |
| Short-Circuit Protection | 15.1A | Fast 3µs response |
| Auto-Retry Interval | 5.0 seconds | If fault persists |

### USB Power Mode (INP Jumper Removed)

| Parameter | Value | Notes |
|-----------|-------|-------|
| EN/UVLO Current | 20.9µA | Still through voltage divider |
| OV Current | 23.6µA | Still through voltage divider |
| Device Quiescent Current | **0µA** | **Device disabled (INP floating)** |
| **Total Battery Current** | **44.5µA** | **Only voltage divider leakage** |
| TPS4800 Output | High impedance | MOSFETs disabled |
| Battery Power Path | Isolated | No power delivery to 5V buck |
| Current Savings | **43µA** | **TPS4800 quiescent eliminated** |

## Protection Summary

| Protection Type | Threshold | Response | Active When |
|----------------|-----------|----------|-------------|
| Undervoltage Lockout | 6.8V battery | Device shutdown | INP jumper installed |
| Overvoltage Protection | 58.5V - 63.8V battery | FET shutdown, FLT asserted | INP jumper installed |
| Overcurrent Protection | 15.1A load current | 3µs response, auto-retry | INP jumper installed |
| Reverse Polarity | -65V maximum | Built-in protection | INP jumper installed |
| **USB Power Isolation** | **Manual** | **Complete battery disconnect** | **INP jumper removed** |

## Power Consumption Analysis

### Battery Mode (INP Jumper Installed)
| Supply Voltage | EN/UVLO Current | OV Current | TPS4800 Quiescent | Total Current | Total Power | Equivalent @ 12V |
|----------------|-----------------|------------|-------------------|---------------|-------------|------------------|
| 13V | 22.6µA | 25.5µA | 43µA | 91.1µA | 1.18mW | 0.099mA |
| 26V | 45.2µA | 51.0µA | 43µA | 139.2µA | 3.62mW | 0.302mA |
| 52V | 90.4µA | 102.1µA | 43µA | 235.5µA | 12.25mW | 1.021mA |

### USB Mode (INP Jumper Removed)
| Supply Voltage | EN/UVLO Current | OV Current | TPS4800 Quiescent | Total Current | Total Power | Equivalent @ 12V |
|----------------|-----------------|------------|-------------------|---------------|-------------|------------------|
| 13V | 22.6µA | 25.5µA | **0µA** | **48.1µA** | **0.63mW** | **0.052mA** |
| 26V | 45.2µA | 51.0µA | **0µA** | **96.2µA** | **2.50mW** | **0.208mA** |
| 52V | 90.4µA | 102.1µA | **0µA** | **192.5µA** | **10.01mW** | **0.834mA** |

**Notes:**
- EN/UVLO divider: 575kΩ total (470kΩ + 105kΩ) - always active for monitoring
- OV divider: 509.2kΩ total (499kΩ + 10.2kΩ) - always active for monitoring
- TPS4800-Q1 quiescent current: 43µA typical when enabled, 0µA when INP disabled
- Power excludes load current through MOSFETs
- "Equivalent @ 12V" shows power consumption as current draw on a 12V system
- **USB mode power savings**: 43µA reduction from TPS4800 shutdown

## Design Notes
- **INP control**: Cleanest method for battery power path isolation without affecting protection monitoring
- **All resistor values**: Standard 1% E96 series for easy sourcing
- **Low current draw**: Design suitable for always-on automotive applications
- **Fast overcurrent protection**: 3µs response with reasonable 5-second retry intervals
- **Conservative margins**: All protection thresholds have adequate safety margins with 0.5V margin above 58V requirement
- **Power source safety**: No possibility of conflicts with proper jumper management
- **LMR36510ADDA compatibility**: Safe to apply USB 5V to buck output when TPS4800 input isolated
- **EVM discrepancy**: Demo board documentation appears to contain calculation errors vs datasheet
- **Reverse polarity behavior**: Still not fully understood without back-to-back output MOSFETs. Will populate both TVS but DNP for initial testing. TI datasheet inadequate for this specific configuration.