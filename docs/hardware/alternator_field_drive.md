# Alternator Field Control

## Overview

This circuit provides PWM control of automotive alternator field current up to 15A. The design consists of a 5V to 12V boost converter driving a high-side MOSFET driver with parallel N-channel MOSFETs for field current switching.

**Key Specifications:**
- Input Voltage: 5V
- Field Supply: VIN_2-60 (2-60V automotive rail)
- Maximum Field Current: 15A
- PWM Frequency Range: 100Hz - 20kHz+ (thermally limited)
- Output: Controls alternator field winding via high-side switching

## Circuit Architecture

### Block 1: Boost Converter (5V → 12V)

**Purpose:** Generate clean 12V supply for gate driver logic from 5V input.

**IC:** U21 - MT3608 Step-Up Converter
- Input: 5V via C63 (4.7µF, 20V input filter)
- Output: ~12V (adjustable via feedback network)
- Enable: C42 (1µF) provides soft-start/enable functionality

**Key Components:**
- **L4:** SRP6540-220M (220µH inductor)
  - Saturation current: ~1-3A (check datasheet)
  - Critical for energy storage during switching
- **D2:** BAT54J Schottky diode
  - Forward voltage: ~0.25V
  - **Current limit: ~200mA continuous - likely system bottleneck**
  - Consider upgrading to 1N5819 or similar for higher current applications
- **Output filtering:** C43 (1µF) || C64 (4.7µF, 20V)

**Feedback Network:**
- **R94:** 10kΩ (upper divider resistor, SW pin to FB pin)
- **R95:** 576Ω (lower divider resistor, FB pin to GND)
- **Current configuration:** Vout ≈ 10.8V
- **For 12V output:**  511Ω

**Feedback Calculation:**
```
V_FB = 0.6V (MT3608 internal reference)
V_SW = V_FB × (R94 + R95) / R95
V_out = V_SW - V_f(D2) = V_SW - 0.25V

Current: V_out = 0.6 × (10000 + 576) / 576 - 0.25 = 10.8V
Target:  V_out = 0.6 × (10000 + 511) / 511 - 0.25 = 12.0V
```

**Current Limitations:**
1. BAT54J diode: 200mA continuous
2. MT3608 internal MOSFET: ~4A peak
3. Inductor saturation: 1-3A typical
4. Thermal: PCB layout dependent

### Block 2: High-Side MOSFET Driver

**Purpose:** Provide isolated high-side gate drive for field current switching.

**IC:** U22 - LM5109AMA High-Side/Low-Side Driver
- VDD: 12V from boost converter  
- PWM Input: OUT_PWM via R54 (100kΩ pull-down)
- Bootstrap supply for high-side drive

**Bootstrap Circuit:**
- **D7:** Diode (part number: TOSHIBA CRH01(TE85L,Q))
- **C4:** 1µF, 100V bootstrap capacitor
- **R96:** 2.2Ω gate drive resistor

**Bootstrap Operation:**
1. When low-side conducts, D7 forward biases
2. C4 charges to VDD (~12V) minus diode drop
3. When high-side switches, C4 provides floating gate drive
4. Gate drive voltage = C4 voltage + switching node voltage

**Frequency Limitations:**
- Bootstrap refresh requires low-side conduction time
- Higher frequencies reduce refresh time
- 1µF C4 adequate for >20kHz operation
- Consider reducing R96 to 1Ω for >50kHz operation

### Block 3: Power MOSFETs

**MOSFETs:** Q3, Q7 - BSC072N08NS5ATMA1 (Parallel Configuration)
- **Voltage Rating:** 80V
- **Rds(on):** 7.2mΩ @ 25°C, Vgs=10V
- **Gate Charge:** ~50nC total
- **Current Rating:** >15A each (thermally limited)

**Single MOSFET Analysis:**
- **Conduction losses:** I²R = 15² × 0.0072 = 1.62W
- **Thermal resistance:** Check datasheet for θjc, θja
- **Parallel MOSFETs likely unnecessary** - single device should handle 15A with proper thermal management

**Protection:**
- **D5:** FSV20100V flyback diode
  - **Critical for inductive load protection**
  - Provides path for field current when MOSFETs switch off
  - Prevents inductive voltage spikes that would destroy MOSFETs
  - 100V rating, fast recovery for high-frequency PWM

## System Operation

### Normal Operation
1. 5V input powers boost converter
2. MT3608 generates 12V for gate driver
3. PWM signal controls LM5109A driver
4. High-side MOSFETs switch field current on/off
5. Flyback diode handles inductive energy during switch-off

### Field Current Path
- **ON state:** VIN_2-60 → MOSFETs → ALTERNATORFIELD → GND
- **OFF state:** ALTERNATORFIELD → D5 → VIN_2-60 (freewheeling)

### PWM Control
- **Frequency range:** 100Hz - 20kHz+ (limited by thermal dissipation)
- **Duty cycle:** 0-100% (limited by bootstrap refresh at high frequency)
- **Current control:** Average field current = Duty Cycle × (VIN_2-60 / R_field)

## Design Considerations & Limitations

### Current Design Bottlenecks
1. **BAT54J diode (D2):** 200mA limit in boost converter
2. **Thermal management:** MOSFET heating at high frequency/current
3. **Bootstrap refresh:** High duty cycle + high frequency combinations

### Recommended Modifications

**For Higher Current Boost:**
- Replace D2 (BAT54J) with 1N5819 or similar 1A+ Schottky
- Verify inductor saturation current rating

**For Higher Frequency Operation (>20kHz):**
- Reduce R96 from 2.2Ω to 1Ω
- Add gate drive bypass capacitance
- Improve thermal management for MOSFETs

**Single MOSFET Operation:**
- DNP either Q3 or Q7 after thermal verification
- Simplifies gate drive, reduces parasitic inductance


## Component Selection Rationale

### Critical Components
- **MT3608:** Proven boost controller, adequate current handling
- **LM5109A:** Purpose-built high-side driver with bootstrap
- **BSC072N08NS5:** Low Rds(on), fast switching, automotive qualified
- **FSV20100V:** Fast recovery, high current flyback diode

### Standard Components
- **Capacitor values:** Standard practice for respective functions
- **Resistor values:** Calculated for specific voltage/current requirements

# Alternator Field Controller - BOM and Net Connections

## Bill of Materials (BOM)

### Active Components

| Designator | Part Number | Description | Package | Value/Rating | Notes |
|------------|-------------|-------------|---------|--------------|-------|
| U21 | MT3608 | Step-Up Boost Converter | SOT-23-6 | 2A, 24V | Main boost controller IC |
| U22 | LM5109AMA | High-Side/Low-Side MOSFET Driver | SOIC-8 | 100V, 2A | Gate driver with bootstrap |
| Q3 | BSC072N08NS5ATMA1 | N-Channel MOSFET | SuperSO8 | 80V, 7.2mΩ | Power switching MOSFET |
| Q7 | BSC072N08NS5ATMA1 | N-Channel MOSFET | SuperSO8 | 80V, 7.2mΩ | Parallel power switching MOSFET |

### Passive Components

| Designator | Value | Rating | Package | Description | Function |
|------------|-------|---------|---------|-------------|----------|
| L4 | 220µH | - | SRP6540 | Inductor (SRP6540-220M) | Boost converter energy storage |
| C63 | 4.7µF | 20V | 0805 | Ceramic capacitor | Boost input filter |
| C42 | 1µF | - | 0603 | Ceramic capacitor | Enable/soft-start |
| C43 | 1µF | - | 0603 | Ceramic capacitor | Boost output filter |
| C64 | 4.7µF | 20V | 0805 | Ceramic capacitor | Boost output filter |
| C4 | 1µF | 100V | 0805 | Ceramic capacitor | Bootstrap capacitor |
| R94 | 10kΩ | 1% | 0603 | Precision resistor | Feedback divider upper |
| R95 | 576Ω | - | 0603 | Resistor | Feedback divider lower |
| R96 | 2.2Ω | - | 0603 | Resistor | Gate drive resistor |
| R54 | 100kΩ | - | 0603 | Resistor | PWM input pull-down |

### Diodes

| Designator | Part Number | Package | Rating | Description | Function |
|------------|-------------|---------|---------|-------------|----------|
| D2 | BAT54J | SOD-323 | 30V, 200mA | Schottky diode | Boost converter output rectifier |
| D7 | CRH01(TE85L,Q) | - | - | Diode | Bootstrap charging diode |
| D5 | FSV20100V | - | 100V, 20A | Fast recovery diode | Flyback/freewheeling diode |

## Net Connections

### Power Rails

**5V Input Rail:**
- Connected to: U21 pin 1 (VIN), C63 positive terminal, C42 positive terminal, U21 pin 4 (EN via C42)

**11V/12V Boost Output Rail:**
- Connected to: D2 cathode, C43 positive terminal, C64 positive terminal, U22 pin 1 (VDD), U22 pin 8 (HB via bootstrap circuit)

**VIN_2-60 Rail (High Voltage):**
- Connected to: Q3 drain, Q7 drain, D5 cathode

**GND (Ground):**
- Connected to: U21 pin 3 (GND), U21 pin 2 (FB via R95), C63 negative terminal, C42 negative terminal, C43 negative terminal, C64 negative terminal, U22 pin 5 (VSS), R54 terminal, D5 anode

### Signal Connections

**OUT_PWM (PWM Input):**
- Connected to: R54 terminal, U22 pin 2 (HI)

**ALTERNATORFIELD (Load Output):**
- Connected to: Q3 source, Q7 source, D5 anode

### Internal IC Connections

**MT3608 (U21) Pin Connections:**
- Pin 1 (SW): Connected to L4, D2 anode, R94
- Pin 2 (GND): Connected to ground rail
- Pin 3 (FB): Connected to R94-R95 junction
- Pin 4 (EN): Connected to C42, 5V rail
- Pin 5 (VIN): Connected to 5V input rail
- Pin 6 (NC): No connection

**LM5109AMA (U22) Pin Connections:**
- Pin 1 (VDD): Connected to 12V boost rail
- Pin 2 (HI): Connected to OUT_PWM via R54
- Pin 3 (LI): Connected to ground (not used in this application)
- Pin 4 (VSS): Connected to ground
- Pin 5 (LO): No connection (not used)
- Pin 6 (HS): Connected to Q3 gate, Q7 gate via R96
- Pin 7 (HO): Connected to bootstrap high side
- Pin 8 (HB): Connected to bootstrap circuit (C4, D7)

### Bootstrap Circuit Connections

**Bootstrap Network:**
- D7 anode: Connected to 12V rail (U22 VDD)
- D7 cathode: Connected to C4 positive terminal, U22 pin 8 (HB)
- C4 negative terminal: Connected to switching node (Q3/Q7 sources, ALTERNATORFIELD)
- U22 pin 7 (HO): High-side gate drive output
- U22 pin 6 (HS): High-side return (connected to switching node)

### Feedback Network Connections

**Voltage Divider:**
- R94 top terminal: Connected to U21 pin 1 (SW switching node)
- R94 bottom terminal: Connected to R95 top terminal, U21 pin 3 (FB)
- R95 bottom terminal: Connected to ground

### Power MOSFET Connections

**Q3 (Primary MOSFET):**
- Drain: Connected to VIN_2-60 rail
- Gate: Connected to U22 pin 6 (HS) via R96
- Source: Connected to ALTERNATORFIELD, Q7 source, D5 anode

**Q7 (Parallel MOSFET):**
- Drain: Connected to VIN_2-60 rail, Q3 drain
- Gate: Connected to U22 pin 6 (HS) via R96, Q3 gate
- Source: Connected to ALTERNATORFIELD, Q3 source, D5 anode

### Inductor and Diode Connections

**L4 (Boost Inductor):**
- Terminal 1: Connected to 5V rail, U21 pin 5 (VIN)
- Terminal 2: Connected to U21 pin 1 (SW), D2 anode, R94

**D2 (Boost Rectifier):**
- Anode: Connected to L4, U21 pin 1 (SW), R94
- Cathode: Connected to 12V rail, C43, C64, U22 VDD

**D5 (Flyback Diode):**
- Anode: Connected to ALTERNATORFIELD, Q3 source, Q7 source
- Cathode: Connected to VIN_2-60 rail, Q3 drain, Q7 drain

## Physical Layout Considerations

### High Current Paths
- VIN_2-60 to MOSFET drains: Wide traces, heavy copper
- MOSFET sources to ALTERNATORFIELD: Wide traces, heavy copper
- Ground connections: Solid ground plane preferred

### Switching Node (High dV/dt)
- U21 pin 1 to L4 to D2: Minimize loop area
- Keep switching node traces short and wide
- Minimize parasitic inductance and capacitance

### Gate Drive Connections
- U22 to MOSFET gates: Controlled impedance, minimize length
- Bootstrap circuit: Keep C4 and D7 close to U22
- Ground returns: Low impedance path to common ground

### Thermal Considerations
- MOSFET thermal pads: Connect to ground plane via thermal vias
- Power dissipating components: Adequate copper pour for heat spreading
- Component spacing: Allow for airflow around power components

---

