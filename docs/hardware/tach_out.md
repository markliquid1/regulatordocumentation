# ESP32 to 60V Tachometer Signal Circuit Documentation

## Circuit Connections (One by One)

**1. ESP32 GPIO Input:**
- ESP32 GPIO pin connects to one end of R78 (2.2kΩ)
- This is the 3.3V logic input signal

**2. Base Drive Circuit:**
- Other end of R78 (2.2kΩ) connects to base of Q1 (MMBT5551)
- R55 (100kΩ) connects between base of Q1 and GND
- R55 ensures base is pulled to ground when ESP32 GPIO is low

**3. Transistor Q1 (MMBT5551):**
- Base: Connected to R78/R55 junction
- Emitter: Connected directly to GND
- Collector: Connected to one end of R80 (100Ω)

**4. Output Circuit:**
- Other end of R80 (100Ω) connects to TACH_OUT pin
- R99 (22kΩ) connects between VIN_2-60-RAW (+Battery) and TACH_OUT
- R99 acts as pullup resistor

**5. Power Supply:**
- VIN_2-60-RAW: +Battery voltage (5V to 60V)
- GND/GND1: System ground, connected together

**6. Output:**
- TACH_OUT: Goes to dashboard tachometer input

## Circuit Operation with ESP32 Square Wave

**When ESP32 GPIO = HIGH (3.3V):**
- Base current flows: 3.3V through R78 (2.2kΩ) to base
- Base current = (3.3V - 0.7V) / 2.2kΩ ≈ 1.18mA
- Q1 turns ON and saturates (VCE_sat ≈ 0.2V)
- Current flows: Battery → R99 (22kΩ) → R80 (100Ω) → Q1 → GND
- **TACH_OUT = 0.2V** (essentially 0V)

**When ESP32 GPIO = LOW (0V):**
- R55 (100kΩ) pulls base to ground
- No base current flows
- Q1 turns OFF (open circuit)
- **TACH_OUT = Battery Voltage** (pulled up through R99)
- Only leakage current flows

## Frequency Range Analysis (8Hz to 2.64kHz)

**MMBT5551 Specifications:**
- fT (transition frequency): ~100MHz
- Storage time: ~200ns
- Rise/fall times: ~35ns

**RC Time Constants:**
- Base charging: R78 × Cbe ≈ 2.2kΩ × 8pF ≈ 18ns
- Output loading: R99 || (Rtach + stray C) 

**At 2.64kHz (worst case):**
- Period = 379µs
- Rise/fall times (35ns) are 0.009% of period
- Switching delays are negligible

**Conclusion:** The 8Hz to 2.64kHz range is **easily achievable** with plenty of margin.

## Power Consumption Analysis

### Case 1: 60V Supply, GPIO HIGH (Q1 ON)
- Total resistance: R99 + R80 = 22kΩ + 100Ω = 22.1kΩ
- Current: 60V / 22.1kΩ = 2.71mA
- **12V equivalent current: 13.6mA @ 12V**
- Power in R99: (2.71mA)² × 22kΩ = 162mW
- Power in R80: (2.71mA)² × 100Ω = 0.7mW
- Power in Q1: 0.2V × 2.71mA = 0.5mW

### Case 2: 60V Supply, GPIO LOW (Q1 OFF)
- Only leakage current through R99
- Leakage current: ~1µA (MMBT5551 spec)
- **12V equivalent current: 5µA @ 12V** (negligible)

### Case 3: 5V Supply, GPIO HIGH (Q1 ON)
- Current: 5V / 22.1kΩ = 0.226mA
- **12V equivalent current: 0.094mA @ 12V**

### Case 4: 5V Supply, GPIO LOW (Q1 OFF)
- **12V equivalent current: ~2µA @ 12V** (negligible leakage)

### Case 5: 50% Duty Cycle Square Wave @ 60V
- Average current = (2.71mA × 0.5) + (0.001mA × 0.5) = 1.36mA average
- **12V equivalent current: 6.8mA @ 12V average**

## Power Summary Table

| Condition | Supply | GPIO State | Current | 12V Equivalent |
|-----------|--------|------------|---------|----------------|
| Extreme High | 60V | HIGH | 2.71mA | **13.6mA @ 12V** |
| Extreme Low | 5V | LOW | ~1µA | **5µA @ 12V** |
| Typical @ 12V | 12V | HIGH | 0.54mA | **0.54mA @ 12V** |
| 50% duty @ 60V | 60V | Square wave | 1.36mA avg | **6.8mA @ 12V avg** |

The highest power consumption occurs at 60V with GPIO HIGH, equivalent to 13.6mA at 12V primarily in the 22kΩ pullup resistor.