### TVS Diode Configuration
**Bidirectional TVS Clamp Bridge (Cathodes Tied Together):**

**D1 (Positive Rail Protection):**
- **SMBJ64A** (64V unidirectional TVS)
- **Connection**: Anode to GND, Cathode to shared node
- **Breakdown Range**: 60.8V-67.2V
- **Function**: Clamps positive transients >64V to ~76-103V range

**D7 (Negative Rail Protection):**
- **SMBJ75A** (75V unidirectional TVS) 
- **Connection**: Anode to VIN, Cathode to shared node
- **Breakdown Range**: 71.3V-83.3V
- **Function**: Clamps negative transients <-75V to ~-90 to -121V range

**Shared Cathode Node**: Creates floating reference point for bidirectional clamping

**Topology Benefits:**
- **Positive overvoltage**: D1 avalanche breakdown clamps high VIN
- **Negative overvoltage**: D7 avalanche breakdown clamps low VIN  
- **Reverse polarity**: Both diodes forward conduct (~1.4V drop), creating controlled short to trip upstream protection
- **No interference**: TVS breakdown voltages above TPS4800-Q1 OVP threshold# TPS4800-Q1 High-Side Driver Design Documentation

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
- **Threshold**: 1.225V rising ±2% (1.20V to 1.25V)
- **Design Requirement**: 58.8V overvoltage protection (48V lithium system)
- **Voltage Divider**: R_top=487kΩ, R_bottom=10.2kΩ (total 497.2kΩ)
- **OVP Trigger Range**: 58.8V to 61.5V (accounting for threshold tolerance)
- **At 12V Battery**: OV pin = 0.246V (well below 1.225V threshold)
- **Current Draw**: 12V / 497.2kΩ = 24.1µA
- **At 60V Battery**: Current = 120.7µA
- **Component Values**: Both 487kΩ and 10.2kΩ are standard 1% resistors

### INP (Pin 3) - Input Control Signal
- **Thresholds**: 2V high, 0.8V low
- **Configuration**: Direct connection to VIN (6-60V)
- **At 12V Battery**: INP = 12V (well above 2V HIGH threshold)
- **Behavior**: Always-on operation when battery voltage present
- **Absolute Maximum**: 70V (our 60V max is within limits)
- **Internal**: 100nA pull-down when floating
- **Rationale**: Simplified control - MOSFET enabled whenever battery connected

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

## Input Protection Design

### Transient Protection Strategy
The design implements a **hierarchical protection approach** to handle automotive transients while preserving proper overvoltage protection functionality:

1. **Primary Protection**: TPS4800-Q1 OV pin triggers controlled shutdown at 61.8V-64.7V
2. **Secondary Protection**: External TVS diodes clamp severe transients above 65V
3. **Energy Storage**: Input capacitors absorb transient energy during brief spikes

### TVS Diode Configuration
**Positive Rail Protection:**
- **SMBJ64A** (64V TVS): Cathode to VIN, Anode to GND
- **Breakdown Range**: 60.8V-67.2V
- **Soft Conduction**: Begins ~65V (above OV protection range)
- **Function**: Clamps positive transients >65V without interfering with OV protection

**Negative Rail Protection:**
- **SMBJ75A** (75V TVS): Anode to VIN, Cathode to GND  
- **Function**: Clamps negative transients below -75V
- **Redundant Protection**: Backs up TPS4800-Q1's internal -65V protection

**Input Capacitance:**
- **Total**: 11µF (10µF bulk + 1µF ceramic)
- **Function**: Energy storage during transients, reduces TVS stress

### Protection Hierarchy Operation
- **6V-58V Normal**: No protection active, normal operation
- **58.8V-61.5V**: OV protection triggers, controlled MOSFET shutdown
- **64V+**: TVS conducts, absorbs transient energy, prevents chip damage
- **Negative spikes**: TVS and internal protection both active

## Component Summary

| Component | Value | Tolerance | Type | Function |
|-----------|-------|-----------|------|----------|
| R1 (EN/UVLO top) | 470kΩ | 1% | Resistor | UVLO voltage divider |
| R2+R3 (EN/UVLO bottom) | 105kΩ | 1% | Resistor | UVLO voltage divider |
| R_OV_top | 487kΩ | 1% | Resistor | OVP voltage divider |
| R_OV_bottom | 10.2kΩ | 1% | Resistor | OVP voltage divider |
| RISCP | 5.6kΩ | 1% | Resistor | Short-circuit threshold |
| CTMR | 220nF | - | Ceramic Cap | Fault timer |
| CBST | 100nF | - | Ceramic Cap | Bootstrap capacitor |
| RSNS | 2mΩ | 1% | Power Resistor | Current sensing (2W, AEC-Q200) |
| D_POS (TVS) | SMBJ64A | - | TVS Diode | Positive transient protection |
| D_NEG (TVS) | SMBJ75A | - | TVS Diode | Negative transient protection |
| C_INPUT | 11µF total | - | Capacitors | Input energy storage |

## Performance at 12V Battery (Typical Operation)

| Parameter | Value | Notes |
|-----------|-------|-------|
| EN/UVLO Current | 20.9µA | Through voltage divider |
| OV Current | 24.1µA | Through voltage divider |
| Total Divider Current | 45.0µA | At 12V battery |
| Device Quiescent Current | 43µA typical | From datasheet |
| Total System Current | ~88µA | Excluding load current |
| Short-Circuit Protection | 15.1A | Fast 3µs response |
| Auto-Retry Interval | 5.0 seconds | If fault persists |

## Protection Summary

| Protection Type | Threshold | Response |
|----------------|-----------|----------|
| Undervoltage Lockout | 6.8V battery | Device shutdown |
| Overvoltage Protection | 58.8V - 61.5V battery | FET shutdown, FLT asserted |
| Overcurrent Protection | 15.1A load current | 3µs response, auto-retry |
| Reverse Polarity | -65V maximum | Built-in protection |

## Design Notes
- All resistor values are standard 1% E96 series for easy sourcing
- Low current draw design suitable for always-on automotive applications
- Fast overcurrent protection with reasonable retry intervals
- Conservative margins on all protection thresholds
- EVM demo board documentation appears to contain calculation errors vs datasheet