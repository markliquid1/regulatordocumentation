# Alternator Field Control

## Overview

This circuit provides PWM control of automotive alternator field current up to 15A. The design consists of a 5V to 12V boost converter driving a high-side MOSFET driver with parallel N-channel MOSFETs for field current switching.

**Key Specifications:**
- Input Voltage: 5V
- Field Supply: VIN_2-60 (2-60V automotive rail)
- Maximum Field Current: 15A
- PWM Frequency Range: 100Hz - 20kHz+ (thermally limited)
- Output: Controls alternator field winding via high-side switching
- Control Input: 3.3V PWM from ESP32 microcontroller

**Test Results- Parallel MOSFETs removed!:**
- Bank Nominal Voltage: 12V
- Date: 9/10/25
- Enclosure: None
- Ambient Temp: 63F
- PWM Frequency: 1200hz 
- PWM Duty Cycle: 100%
- Battery Voltage: 12.93V
- Field Voltage: 12.83V
- Field Current (engine off): 4.2A
- Maximum PCB Temp: ~30C and most of that heat is left over from soldering
- Conclusion: Not expecting any thermal issues in this design, V3 issues were related to high MOSFET Rds(on) due to low Vgs


## Circuit Architecture

### Block 1: Level Shifter and System Enable Logic (3.3V → 5V Logic)

**Purpose:** Translate ESP32 3.3V FIELD_ENABLE signal to 5V logic for boost converter control while providing multiple system disable conditions.

**IC:** U15 - SN74LVC1T45DBV Single-Bit Dual-Supply Bus Transceiver
- **VCCA (Pin 1):** 5V supply for A-side
- **VCCB (Pin 6):** 3V3 supply for B-side  
- **B (Pin 4):** Combined input from three control signals
- **A (Pin 3):** Output to MT3608 Enable pin (5V logic)
- **DIR (Pin 5):** Direction control - tied to B pin (Pin 4)
- **GND (Pin 2):** Ground

**System Enable/Disable Logic:**

The system uses a **wired-AND** configuration at U15 Pin 4 that combines three control signals:

1. **FIELD_ENABLE (ESP32 GPIO):** Active-HIGH enable signal
   - 3.3V output from ESP32 through 10kΩ series resistor
   - Provides pull-up voltage when system should be enabled
   - When LOW or ESP32 powered down: disables entire system

2. **ALERT! (INA228 Alert Pin):** Active-LOW disable signal  
   - Open-drain output from current monitor IC
   - When INA228 detects overcurrent/fault: pulls pin to ground
   - Overrides FIELD_ENABLE pull-up, disabling system immediately

3. **ON/OFF (Control Panel Toggle Switch):** Active-LOW disable signal
   - Manual toggle switch that can short pin to ground
   - Operator control for manual system disable
   - When closed (OFF position): overrides FIELD_ENABLE, disables system

**Logic Function at U15 Pin 4:**
- **System ENABLED when:** FIELD_ENABLE = HIGH **AND** ALERT! = open (no fault) **AND** ON/OFF = open (ON position)
- **System DISABLED when:** FIELD_ENABLE = LOW **OR** ALERT! = active (pulls low) **OR** ON/OFF = closed (OFF position)

**Operation:**
- U15 level-shifts the combined 3.3V logic signal to 5V for MT3608 enable
- Any single disable condition (FIELD_ENABLE low, ALERT active, or ON/OFF switch) will disable the boost converter
- This immediately removes 12V power from the LM5109A gate driver, turning off field current
- Provides multiple layers of protection: software control, hardware fault detection, and manual override

**Enable Signal Path:**
FIELD_ENABLE/ALERT!/ON/OFF → U15 Pin 4 → U15 Pin 3 (5V) → U21 Pin 4 (EN) → MT3608 boost converter → 12V supply for LM5109A

### Block 2: Boost Converter (5V → 12V)

**Purpose:** Generate clean 12V supply for gate driver logic from 5V input.

**IC:** U21 - MT3608 (SOT-23-6, fixed-freq current-mode boost)
- **VIN (Pin 5):** 5V via C63 (4.7µF ceramic, 20V) - *Datasheet recommends ≥22µF for optimal performance*
- **VOUT (target):** 12.07V (set by feedback divider from VOUT → FB → GND). FB regulates to 0.6V
- **EN (Pin 4):** From U15 (5V logic). EN high ≥1.5V
- **fSW:** ~1.2MHz internal switching frequency

**Key Components:**
- **L4 (Inductor):** SRP6540-220M = 22µH (within recommended 4.7-22µH range at 1.2MHz)
  - Use low DCR; choose Isat high enough to avoid saturation near switch current limit
- **D2 (Boost Schottky):** BAT54J - Anode = SW (Pin 1), Cathode = VOUT
  - Reverse rating > VOUT; current rating sized for application
  - For present load (LM5109A only, few mA), BAT54J is adequate
  - For higher current applications, upgrade to SS14/SS16-class
- **Output caps (C43, C64):** Total 5.7µF ceramic - *Datasheet recommends ≥22µF total for optimal ripple performance*
  - C43: 1µF, C64: 4.7µF, ≥25V rating, close to D2 cathode-GND

**Feedback Network (VOUT → FB → GND):**
- **R94:** 10kΩ (upper divider resistor, VOUT to FB)
- **R95:** 523Ω (lower divider resistor, FB to GND)
- **Setpoint formula:** V_OUT = 0.6V × (1 + R94/R95)
- **Current configuration:** V_OUT = 0.6V × (1 + 10000/523) = 12.07V

**Feedback Network Calculation:**
```
V_FB = 0.6V (MT3608 internal reference)
V_OUT = 0.6V × (1 + 10000/523) = 12.07V
```

**Note:** Feedback divider senses VOUT (after diode), not the SW switching node. This ensures regulation of the actual output voltage.

**Current Limitations:**
1. BAT54J diode: 200mA continuous current limit
2. MT3608 internal MOSFET: ~4A peak
3. Inductor saturation: Check L4 specifications
4. Input/output capacitors: Smaller than datasheet recommendations may affect ripple performance

### Block 3: High-Side MOSFET Driver

**Purpose:** Provide isolated high-side gate drive for field current switching.

**IC:** U22 - LM5109AMA High-Side/Low-Side Driver
- VDD: 12.07V from boost converter  
- PWM Input: OUT_PWM via voltage divider network
- Bootstrap supply for high-side drive

**PWM Input Network:**
- **R89:** 100Ω (series resistor)
- **R54:** 10kΩ (pull-down resistor)
- **Input voltage:** 3.3V ESP32 PWM → 3.27V at HI pin
- **Signal path:** ESP32 → R89 → Pin 2 (HI)
- **Pull-down:** Pin 2 → R54 → GND

**Pin Configuration:**
- **Pin 1 (VDD):** 12.07V from boost converter
- **Pin 2 (HI):** PWM input (3.27V from ESP32)
- **Pin 3 (LI):** Tied to ground (VSS) - unused input per datasheet requirement
- **Pin 4 (VSS):** Ground
- **Pin 5 (LO):** Not connected (unused output)
- **Pin 6 (HS):** High-side return (connected to switching node)
- **Pin 7 (HO):** High-side gate drive output
- **Pin 8 (HB):** Bootstrap supply input

**Bootstrap Circuit:**
- **D7:** CRH01(TE85L,Q) bootstrap charging diode
- **C4:** 1µF, 100V bootstrap capacitor
- **R96:** 2.2Ω gate drive resistor

**Bootstrap Operation:**
1. When MOSFETs are off, HS node at ground potential
2. D7 forward biases, C4 charges to VDD (~12.07V) minus diode drop
3. When HI goes high, C4 provides floating gate drive
4. Gate drive voltage = C4 voltage + switching node voltage
5. Adequate for frequencies up to 20kHz with proper duty cycle

**Frequency Limitations:**
- Bootstrap refresh requires off-time when HS ≈ 0V
- Higher frequencies reduce refresh time
- 1µF C4 adequate for >20kHz operation
- Consider reducing R96 to 1Ω for >50kHz operation

### Block 4: Power MOSFETs

**MOSFETs:** Q3, Q7 - BSC072N08NS5ATMA1 (Parallel Configuration)
- **Voltage Rating:** 80V
- **Rds(on):** 7.2mΩ @ 25°C, Vgs=10V
- **Gate Charge:** ~50nC total
- **Current Rating:** >15A each (thermally limited)

**Parallel Configuration:**
- **Drains:** Both connected to VIN_2-60 rail
- **Sources:** Both connected to ALTERNATORFIELD output
- **Gates:** Both driven from HO pin through R96

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
1. ESP32 FIELD_ENABLE signal enables boost converter via U15 (requires ALERT! inactive and ON/OFF switch in ON position)
2. 5V input powers boost converter
3. MT3608 generates 12.07V for gate driver
4. ESP32 PWM signal controls LM5109A driver via voltage divider
5. High-side MOSFETs switch field current on/off
6. Flyback diode handles inductive energy during switch-off

### System Enable/Disable Conditions

**System ENABLED when ALL conditions met:**
- FIELD_ENABLE = HIGH (ESP32 GPIO active)
- ALERT! = inactive/open (no current monitor fault)  
- ON/OFF = open (manual switch in ON position)

**System DISABLED when ANY condition occurs:**
- FIELD_ENABLE = LOW (ESP32 disables or powers down)
- ALERT! = active (INA228 detects overcurrent/fault)
- ON/OFF = closed (manual switch in OFF position)

**Disable Response:**
1. Any disable condition pulls U15 input low
2. U15 output goes low, disabling MT3608
3. 12V supply to LM5109A is removed
4. Gate driver stops switching MOSFETs
5. Field current ceases immediately

### Field Current Path
- **ON state:** VIN_2-60 → MOSFETs → ALTERNATORFIELD → Load → GND
- **OFF state:** ALTERNATORFIELD → D5 → VIN_2-60 (freewheeling)

### PWM Control
- **Input:** 3.3V PWM from ESP32
- **Processing:** Voltage divider reduces to 3.27V at LM5109A
- **Frequency range:** 100Hz - 20kHz+ (limited by thermal dissipation and bootstrap refresh)
- **Duty cycle:** 0-100% (limited by bootstrap refresh at high frequency)
- **Current control:** Average field current = Duty Cycle × (VIN_2-60 / R_field)

## Design Considerations & Limitations

### Current Design Characteristics
1. **BAT54J diode (D2):** 200mA continuous current limit - adequate for gate driver load
2. **Capacitor values:** Smaller than datasheet recommendations but functional for light loads
3. **Thermal management:** MOSFET heating at high frequency/current
4. **Bootstrap refresh:** High duty cycle + high frequency combinations

### Recommended Modifications

**For Higher Current Boost Applications:**
- Replace D2 (BAT54J) with 1N5819 or similar 1A+ Schottky
- Upgrade input capacitor C63 to ≥22µF as per datasheet
- Upgrade output capacitors C43+C64 to ≥22µF total as per datasheet

**For Exact 12V Output:**
- Change R95 from 523Ω to 526Ω for 12.01V output

**For Higher Frequency Operation (>20kHz):**
- Reduce R96 from 2.2Ω to 1Ω
- Add gate drive bypass capacitance
- Improve thermal management for MOSFETs

**Single MOSFET Operation:**
- DNP either Q3 or Q7 after thermal verification
- Simplifies gate drive, reduces parasitic inductance

## Component Selection Rationale

### Critical Components
- **SN74LVC1T45DBV:** Reliable 3.3V to 5V level translation with bidirectional capability
- **MT3608:** Proven boost controller, adequate current handling
- **LM5109A:** Purpose-built high-side driver with bootstrap
- **BSC072N08NS5:** Low Rds(on), fast switching, automotive qualified
- **FSV20100V:** Fast recovery, high current flyback diode

### Standard Components
- **Capacitor values:** Functional for application but smaller than datasheet optimal values
- **Resistor values:** Calculated for specific voltage/current requirements

# Alternator Field Controller - BOM and Net Connections

## Bill of Materials (BOM)

### Active Components

| Designator | Part Number | Description | Package | Value/Rating | Notes |
|------------|-------------|-------------|---------|--------------|-------|
| U15 | SN74LVC1T45DBV | Single-Bit Dual-Supply Bus Transceiver | SOT-23-6 | 5.5V, 32mA | Level shifter 3.3V→5V with enable logic |
| U21 | MT3608 | Step-Up Boost Converter | SOT-23-6 | 2A, 24V | Main boost controller IC |
| U22 | LM5109AMA | High-Side/Low-Side MOSFET Driver | SOIC-8 | 100V, 2A | Gate driver with bootstrap |
| Q3 | BSC072N08NS5ATMA1 | N-Channel MOSFET | SuperSO8 | 80V, 7.2mΩ | Power switching MOSFET |
| Q7 | BSC072N08NS5ATMA1 | N-Channel MOSFET | SuperSO8 | 80V, 7.2mΩ | Parallel power switching MOSFET |

### Passive Components

| Designator | Value | Rating | Package | Description | Function |
|------------|-------|---------|---------|-------------|----------|
| L4 | 22µH | - | SRP6540 | Inductor (SRP6540-220M) | Boost converter energy storage |
| C63 | 4.7µF | 20V | 0805 | Ceramic capacitor | Boost input filter (datasheet recommends ≥22µF) |
| C42 | 1µF | - | 0603 | Ceramic capacitor | Enable/soft-start |
| C43 | 1µF | - | 0603 | Ceramic capacitor | Boost output filter |
| C64 | 4.7µF | 20V | 0805 | Ceramic capacitor | Boost output filter (total 5.7µF, datasheet recommends ≥22µF) |
| C4 | 1µF | 100V | 0805 | Ceramic capacitor | Bootstrap capacitor |
| C11 | 1µF | - | 0603 | Ceramic capacitor | Level shifter supply filter |
| R94 | 10kΩ | 1% | 0603 | Precision resistor | Feedback divider upper |
| R95 | 523Ω | - | 0603 | Resistor | Feedback divider lower |
| R96 | 2.2Ω | - | 0603 | Resistor | Gate drive resistor |
| R89 | 100Ω | - | 0603 | Resistor | PWM input series resistor |
| R54 | 10kΩ | - | 0603 | Resistor | PWM input pull-down |

### Diodes

| Designator | Part Number | Package | Rating | Description | Function |
|------------|-------------|---------|---------|-------------|----------|
| D2 | BAT54J | SOD-323 | 30V, 200mA | Schottky diode | Boost converter output rectifier |
| D7 | CRH01(TE85L,Q) | - | - | Diode | Bootstrap charging diode |
| D5 | FSV20100V | - | 100V, 20A | Fast recovery diode | Flyback/freewheeling diode |

## Net Connections

### Power Rails

**3V3 Rail:**
- Connected to: U15 pin 6 (VCCB), C11 positive terminal

**5V Input Rail:**
- Connected to: U15 pin 1 (VCCA), U21 pin 5 (VIN), C63 positive terminal, C42 positive terminal

**12.07V Boost Output Rail:**
- Connected to: D2 cathode, C43 positive terminal, C64 positive terminal, U22 pin 1 (VDD), D7 anode, R94 (feedback upper resistor)

**VIN_2-60 Rail (High Voltage):**
- Connected to: Q3 drain, Q7 drain, D5 cathode

**GND (Ground):**
- Connected to: U15 pin 2, U21 pin 2, U22 pin 4 (VSS), U22 pin 3 (LI), C63 negative, C42 negative, C43 negative, C64 negative, C11 negative, R95, R54, D5 anode

### Signal Connections

**FIELD_ENABLE (System Enable):**
- Connected to: U15 pin 4 (B input) through 10kΩ series resistor from ESP32 GPIO

**ALERT! (INA228 Alert Pin):**
- Connected to: U15 pin 4 (B input) - open drain output, pulls low on fault

**ON/OFF (Manual Toggle Switch):**
- Connected to: U15 pin 4 (B input) - shorts to ground when in OFF position

**OUT_PWM (PWM Input from ESP32):**
- Connected to: R89 → U22 pin 2 (HI), R54 pull-down to GND

**ALTERNATORFIELD (Load Output):**
- Connected to: Q3 source, Q7 source, D5 anode, C4 negative terminal, U22 pin 6 (HS)

### Internal IC Connections

**SN74LVC1T45DBV (U15) Pin Connections:**
- Pin 1 (VCCA): 5V rail
- Pin 2 (GND): Ground
- Pin 3 (A): Output to U21 pin 4 (EN)
- Pin 4 (B): Combined input from FIELD_ENABLE (via 10kΩ resistor), ALERT! (open drain), and ON/OFF (toggle switch)
- Pin 5 (DIR): Direction control - tied to Pin 4 (B input)
- Pin 6 (VCCB): 3V3 rail

**MT3608 (U21) Pin Connections:**
- Pin 1 (SW): Connected to L4 terminal 2, D2 anode
- Pin 2 (GND): Connected to ground rail
- Pin 3 (FB): Connected to R94-R95 junction (feedback divider midpoint)
- Pin 4 (EN): Connected to U15 pin 3 output
- Pin 5 (VIN): Connected to 5V input rail, L4 terminal 1
- Pin 6 (NC): No connection

**LM5109AMA (U22) Pin Connections:**
- Pin 1 (VDD): Connected to 12.07V boost rail
- Pin 2 (HI): Connected to OUT_PWM via R89, pulled down by R54
- Pin 3 (LI): Connected to ground (VSS) per datasheet requirement
- Pin 4 (VSS): Connected to ground
- Pin 5 (LO): No connection (unused output)
- Pin 6 (HS): Connected to switching node (Q3/Q7 sources, ALTERNATORFIELD)
- Pin 7 (HO): Connected to Q3 gate, Q7 gate via R96
- Pin 8 (HB): Connected to bootstrap circuit (C4 positive, D7 cathode)

### Bootstrap Circuit Connections

**Bootstrap Network:**
- D7 anode: Connected to 12.07V rail (U22 VDD)
- D7 cathode: Connected to C4 positive terminal, U22 pin 8 (HB)
- C4 negative terminal: Connected to switching node (Q3/Q7 sources, ALTERNATORFIELD, U22 pin 6)
- U22 pin 7 (HO): High-side gate drive output to MOSFET gates
- U22 pin 6 (HS): High-side return (connected to switching node)

### Feedback Network Connections

**Voltage Divider (senses VOUT after D2):**
- R94 top terminal: Connected to boost output rail (D2 cathode, 12.07V)
- R94 bottom terminal: Connected to R95 top terminal, U21 pin 3 (FB)
- R95 bottom terminal: Connected to ground

### Power MOSFET Connections

**Q3 (Primary MOSFET):**
- Drain: Connected to VIN_2-60 rail
- Gate: Connected to U22 pin 7 (HO) via R96
- Source: Connected to ALTERNATORFIELD, Q7 source, D5 anode, U22 pin 6 (HS), C4 negative

**Q7 (Parallel MOSFET):**
- Drain: Connected to VIN_2-60 rail, Q3 drain
- Gate: Connected to U22 pin 7 (HO) via R96, Q3 gate
- Source: Connected to ALTERNATORFIELD, Q3 source, D5 anode, U22 pin 6 (HS), C4 negative

### Inductor and Diode Connections

**L4 (Boost Inductor):**
- Terminal 1: Connected to 5V rail, U21 pin 5 (VIN)
- Terminal 2: Connected to U21 pin 1 (SW), D2 anode

**D2 (Boost Rectifier):**
- Anode: Connected to L4 terminal 2, U21 pin 1 (SW)
- Cathode: Connected to 12.07V rail, C43, C64, U22 VDD, R94

**D5 (Flyback Diode):**
- Anode: Connected to ALTERNATORFIELD, Q3 source, Q7 source, U22 pin 6 (HS), C4 negative
- Cathode: Connected to VIN_2-60 rail, Q3 drain, Q7 drain

### Enable Logic Connections (Wired-AND Configuration)

**U15 Pin 4 (Combined Enable Input):**
- **FIELD_ENABLE:** ESP32 GPIO → 10kΩ resistor → U15 Pin 4 (provides pull-up when active)
- **ALERT!:** INA228 alert pin → U15 Pin 4 (open-drain, pulls low on fault)
- **ON/OFF:** Manual toggle switch → U15 Pin 4 (shorts to ground when in OFF position)
- **Pull-down:** R_PD (100kΩ) → U15 Pin 4 to GND (ensures defined LOW state when all signals inactive)

**Logic Function:**
- Pin 4 voltage HIGH (3.0V) when: FIELD_ENABLE high AND ALERT! inactive (open) AND ON/OFF open (ON position)
- Pin 4 voltage LOW (0V) when: FIELD_ENABLE low OR ALERT! active (low) OR ON/OFF closed (OFF position)
- ESP32 current draw: 30µA through 10kΩ + 100kΩ voltage divider

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

### Signal Integrity
- PWM input traces: Keep away from switching nodes
- Bootstrap circuit: Minimize loop area for HB-HS
- Level shifter: Adequate supply filtering near IC
- Feedback traces: Route away from SW node to minimize noise pickup
- Enable logic traces: Keep FIELD_ENABLE, ALERT!, and ON/OFF traces separated from high-frequency switching nodes

### MT3608 Specific Layout Requirements
- Input capacitor C63: Place close to VIN-GND pins for tight current loop
- Output capacitors C43/C64: Place close to D2 cathode-GND for tight current loop
- SW node: Minimize copper area and trace length to reduce EMI
- Feedback routing: Keep FB traces away from SW node switching area

### Enable Logic Layout
- U15 placement: Position close to enable signal sources to minimize trace lengths
- Control signal routing: Keep ALERT! and ON/OFF traces away from switching nodes
- Pull-up resistor: Place 10kΩ FIELD_ENABLE series resistor close to ESP32 output
- Ground connections: Ensure solid ground reference for control logic

---