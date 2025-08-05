# Analog Inputs

The regulator has four analog input channels via ADS1115 16-bit ADC with I2C interface, and a precision voltage divider architecture with op-amp buffering for high-impedance, accurate voltage measurements with minimal power draw.

---

# Design Architecture

## System Overview
All analog input channels use the same fundamental architecture:

1. **High-impedance voltage divider**: Reduces input voltage to ADC-compatible range
2. **Low-pass filter**: 5nF capacitor provides noise filtering  
3. **Unity-gain buffer**: MCP6004 op-amp provides impedance conversion
4. **16-bit ADC**: ADS1115 performs precision analog-to-digital conversion

## Single-Supply Design Benefits
- **Inherent overvoltage protection**: Op-amp cannot exceed 3.3V supply rails
- **No protection diodes needed**: Eliminates leakage current errors
- **Perfect accuracy**: No temperature coefficient issues that come from diode type protections (leakage currents)
- **Low power consumption**: Single 3.3V supply shared with ADS1115

---

# Circuit Architecture

```
Input Signal ──[R1]──┬──[R2]──GND
                     │
                  [5nF]──GND (Low-pass filter)
                     │
              [MCP6004 Input+]
                     │
              [MCP6004 Output]──ADS1115 Channel
                     │
              [Feedback to Input-] (Unity gain)
```

## Component Functions

### Voltage Divider (R1/R2)
- **Function**: Scales high input voltages to ADS1115-compatible range (0-3.3V)
- **Power consumption**: Continuous, proportional to input voltage
- **Ratio selection**: Determines maximum measurable voltage

### Low-Pass Filter (5nF Capacitor)
- **Cutoff frequency**: f = 1 / (2π × R_thevenin × 5nF)
- **Function**: Removes high-frequency noise and EMI
- **Settling time**: ~5 × time constant for 99% accuracy
- **Placement**: Before op-amp buffer (optimal for noise rejection)

### Unity-Gain Buffer (MCP6004)
- **Function**: High input impedance, low output impedance conversion
- **Gain**: Exactly 1.0 (Vout = Vin)
- **Supply**: Single 3.3V rail
- **Configuration**: Non-inverting input from voltage divider, output connected to inverting input

### ADC (ADS1115)
- **Resolution**: 16-bit (±4.096V range recommended)
- **Interface**: I2C
- **Sample rate**: Configurable (128 SPS recommended for low noise)
- **Effective Input range**: 0-3.6V (GAIN = 1 setting but limited by Vdd)

---

# Channel Specifications

## Channel 0: Battery Voltage Monitor

### Design Parameters
- **R1**: 1MΩ ±0.1%, 0805 SMD
- **R2**: 49.9kΩ ±0.1%, 0805 SMD  
- **Divider ratio**: 0.0475
- **Filter capacitor**: 5nF ±10%, 0603 SMD

### Performance Analysis
| Input Voltage | Divided Voltage | Measurable | Resolution |
|---------------|-----------------|------------|------------|
| 2.1V | 0.100V | ✅ Minimum | 2.6mV |
| 5V | 0.238V | ✅ YES | 2.6mV (0.053%) |
| 12V | 0.570V | ✅ YES | 2.6mV (0.022%) |
| 24V | 1.141V | ✅ YES | 2.6mV (0.011%) |
| 48V | 2.281V | ✅ YES | 2.6mV (0.0054%) |
| 60V | 2.852V | ✅ YES | 2.6mV (0.0043%) |
| 65.2V | 3.100V | ✅ Maximum | 2.6mV |

### Filter Characteristics
- **Thevenin resistance**: 47.5kΩ
- **Cutoff frequency**: 670Hz
- **Time constant**: 0.24ms
- **99% settling time**: 1.2ms

### Power Consumption
| Input Voltage | Divider Current | Divider Power | Op-amp Power | Total Power |
|---------------|-----------------|---------------|--------------|-------------|
| 12V | 11.4µA | 0.14mW | 1µA | 0.15mW |
| 24V | 22.9µA | 0.55mW | 1µA | 0.56mW |
| 48V | 45.7µA | 2.19mW | 1µA | 2.20mW |
| 60V | 57.1µA | 3.43mW | 1µA | 3.44mW |

**Measurement range**: 2.1V to 65.2V

---

## Channel 1: Alternator Current Monitor

### Design Parameters
- **R1**: 768kΩ ±0.1%, 0805 SMD
- **R2**: 768kΩ ±0.1%, 0805 SMD
- **Divider ratio**: 0.5000
- **Filter capacitor**: 5nF ±10%, 0603 SMD

### Performance Analysis
| Input Voltage | Divided Voltage | Measurable | Resolution |
|---------------|-----------------|------------|------------|
| 0.2V | 0.100V | ✅ Minimum | 2.6mV |
| 1V | 0.500V | ✅ YES | 2.6mV (0.26%) |
| 3V | 1.500V | ✅ YES | 2.6mV (0.087%) |
| 5V | 2.500V | ✅ YES | 2.6mV (0.052%) |
| 6.2V | 3.100V | ✅ Maximum | 2.6mV |

### Filter Characteristics
- **Thevenin resistance**: 384kΩ
- **Cutoff frequency**: 83Hz
- **Time constant**: 1.9ms
- **99% settling time**: 9.6ms

### Power Consumption
| Input Voltage | Divider Current | Divider Power | Op-amp Power | Total Power |
|---------------|-----------------|---------------|--------------|-------------|
| 1V | 0.65µA | 0.65µW | 1µA | 3.95µW |
| 3V | 1.95µA | 5.86µW | 1µA | 9.16µW |
| 5V | 3.26µA | 16.3µW | 1µA | 19.6µW |
| 6.2V | 4.04µA | 25.0µW | 1µA | 28.3µW |

**Measurement range**: 0.2V to 6.2V

---

## Channel 2: Engine Speed Monitor (LM2907 Output)

### Design Parameters
- **R1**: 768kΩ ±0.1%, 0805 SMD
- **R2**: 768kΩ ±0.1%, 0805 SMD
- **Divider ratio**: 0.5000
- **Filter capacitor**: 5nF ±10%, 0603 SMD

### Performance Analysis
| Input Voltage | Divided Voltage | Measurable | Resolution |
|---------------|-----------------|------------|------------|
| 0.2V | 0.100V | ✅ Minimum | 2.6mV |
| 1V | 0.500V | ✅ YES | 2.6mV (0.26%) |
| 3V | 1.500V | ✅ YES | 2.6mV (0.087%) |
| 5V | 2.500V | ✅ YES | 2.6mV (0.052%) |
| 6.2V | 3.100V | ✅ Maximum | 2.6mV |

### Filter Characteristics
- **Thevenin resistance**: 384kΩ
- **Cutoff frequency**: 83Hz
- **Time constant**: 1.9ms
- **99% settling time**: 9.6ms

### Power Consumption
| Input Voltage | Divider Current | Divider Power | Op-amp Power | Total Power |
|---------------|-----------------|---------------|--------------|-------------|
| 1V | 0.65µA | 0.65µW | 1µA | 3.95µW |
| 3V | 1.95µA | 5.86µW | 1µA | 9.16µW |
| 5V | 3.26µA | 16.3µW | 1µA | 19.6µW |
| 6.2V | 4.04µA | 25.0µW | 1µA | 28.3µW |

**Measurement range**: 0.2V to 6.2V

---

## Channel 3: User Configurable

### Design Parameters
- **R1**: ___ kΩ ±0.1%, 0805 SMD (User specified)
- **R2**: ___ kΩ ±0.1%, 0805 SMD (User specified)
- **Divider ratio**: R2 / (R1 + R2)
- **Filter capacitor**: 5nF ±10%, 0603 SMD

### Design Guidelines

**To calculate resistor values for desired voltage range:**

1. **Determine maximum input voltage** (V_max)
2. **Set maximum ADC voltage** = 3.0V (safety margin)
3. **Calculate divider ratio**: Ratio = 3.0V / V_max
4. **Select R1**: Choose 470kΩ to 2.2MΩ for low power
5. **Calculate R2**: R2 = R1 × Ratio / (1 - Ratio)

**Example calculations:**

| Max Input | Divider Ratio | R1 | R2 | Power @ Max |
|-----------|---------------|----|----|-------------|
| 10V | 0.300 | 1MΩ | 429kΩ | 70µW |
| 30V | 0.100 | 1MΩ | 111kΩ | 812µW |
| 100V | 0.030 | 2MΩ | 62kΩ | 4.85mW |

### Filter Characteristics (varies with resistor selection)
- **Thevenin resistance**: R_th = (R1 × R2) / (R1 + R2)
- **Cutoff frequency**: f = 1 / (2π × R_th × 5nF)
- **99% settling time**: 5 × R_th × 5nF

**Measurement range**: Determined by user resistor selection

---

# MCP6004 Quad Op-Amp Configuration

## Pinout and Connections

| Pin | Symbol | Description | Connection |
|-----|--------|-------------|------------|
| 1 | VOUT, VOUTA | Analog Output (Op Amp A) | **ADS1115 Channel 0 (A0)** |
| 2 | VIN-, VINA- | Inverting Input (Op Amp A) | **Connect to Pin 1 (VOUTA)** |
| 3 | VIN+, VINA+ | Noninverting Input (Op Amp A) | **Channel 0 signal from voltage divider** |
| 4 | VSS | Negative Power Supply | **GND (0V)** |
| 5 | VINB+ | Noninverting Input (Op Amp B) | **Channel 1 signal from voltage divider** |
| 6 | VINB- | Inverting Input (Op Amp B) | **Connect to Pin 7 (VOUTB)** |
| 7 | VOUTB | Analog Output (Op Amp B) | **ADS1115 Channel 1 (A1)** |
| 8 | VOUTC | Analog Output (Op Amp C) | **ADS1115 Channel 2 (A2)** |
| 9 | VINC- | Inverting Input (Op Amp C) | **Connect to Pin 8 (VOUTC)** |
| 10 | VINC+ | Noninverting Input (Op Amp C) | **Channel 2 signal from voltage divider** |
| 11 | VDD | Positive Power Supply | **+3.3V** |
| 12 | VIND+ | Noninverting Input (Op Amp D) | **Channel 3 signal from voltage divider** |
| 13 | VIND- | Inverting Input (Op Amp D) | **Connect to Pin 14 (VOUTD)** |
| 14 | VOUTD | Analog Output (Op Amp D) | **ADS1115 Channel 3 (A3)** |

## Unity Gain Configuration
Each op-amp is configured as a unity-gain buffer by connecting the output to the inverting input:
- Op-Amp A: Pin 2 connects to Pin 1
- Op-Amp B: Pin 6 connects to Pin 7
- Op-Amp C: Pin 9 connects to Pin 8
- Op-Amp D: Pin 13 connects to Pin 14

---

# ADS1115 Configuration

## Recommended Settings
- **PGA (Programmable Gain Amplifier)**: ±4.096V (GAIN = 1)


## Resolution Analysis
- **ADC range**: ±4.096V
- **ADC resolution**: 0.125mV per LSB (16-bit)
- **Input resolution**: 0.125mV / divider_ratio per LSB

---

# System Power Consumption

## Total 4-Channel Power Consumption

### At Typical Operating Voltages
| Condition | Channel 0 (12V) | Channel 1 (3V) | Channel 2 (3V) | Channel 3 (User) | ADS1115 | MCP6004 | Total |
|-----------|-----------------|----------------|----------------|-------------------|---------|---------|-------|
| **Divider Power** | 0.14mW | 5.9µW | 5.9µW | User defined | - | - | Variable |
| **IC Power** | - | - | - | - | 0.50mW | 0.013mW | 0.51mW |
| **Total Current** | 11.4µA | 1.95µA | 1.95µA | User defined | 150µA | 4µA | Variable |

### Battery Life Impact (100Ah 12V Battery)
- **Continuous operation**: >10 years (assuming reasonable Channel 3 configuration)
- **Daily consumption**: <0.01% of battery capacity
- **Excellent for battery-powered marine applications**

---

# Design Validation

## Overvoltage Protection
- **Inherent protection**: Single-supply op-amp cannot exceed 3.3V output
- **No protection diodes needed**: Eliminates leakage current errors
- **Safety margin**: Op-amp max output (3.1V) < ADS1115 max input (3.6V)

## Accuracy Verification
- **No leakage current errors**: Op-amp buffer isolates high-impedance dividers
- **Temperature stable**: No Schottky diode temperature coefficients
- **Precision components**: 0.1% resistors, precision voltage reference

## Noise Performance
- **Low-pass filtering**: 5nF capacitors remove high-frequency noise
- **EMI immunity**: Capacitive filtering before amplification
- **Ground isolation**: Single-point grounding to ADS1115

---

# Bill of Materials

## Per Channel (Channels 0-2 Specified)
- **R1**: High-side resistor, ±0.1%, 1/8W, 0805 SMD
- **R2**: Low-side resistor, ±0.1%, 1/8W, 0805 SMD  
- **C1**: 5nF ±10%, X7R, 0603 SMD ceramic capacitor

## Shared Components (All Channels)
- **U1**: MCP6004-I/SL quad op-amp, SOIC-14 SMD
- **U2**: ADS1115 16-bit ADC, MSOP-10 SMD
- **C2**: 100nF ±10%, X7R, 0603 SMD decoupling capacitor (VDD to VSS)

## Channel 3 User Selection
- **R1**: ___ kΩ ±0.1%, 1/8W, 0805 SMD (User specified)
- **R2**: ___ kΩ ±0.1%, 1/8W, 0805 SMD (User specified)

---

## Marine Environment Suitability
- **Temperature range**: -40°C to +85°C (Industrial grade components)
- **Low power consumption**: Suitable for battery-powered installations
- **High accuracy**: Precision monitoring for critical systems
- **EMI immunity**: 5nF filtering handles marine electrical noise

# Minimum Measurable Input Limits (Engineering Units)

The table below summarizes the lowest input value that can be reliably measured on each channel, based on:
- MCP6004 op-amp VOL (20–70 mV typical)
- Voltage divider ratios
- ADC scaling equations from firmware

| Channel | Engineering Units | Min Input (typ–max)        |
| ------- | ----------------- | -------------------------- |
| 0       | Battery Voltage   | **0.29–1.00 V**            |
| 1       | Measured Amps     | **−226 to −166 A**         |
| 2       | RPM               | **0.25–0.84 RPM × factor** |
| 3       | Thermistor V      | **>204V → always invalid** |

### Calculation Notes

- **Channel 0**:  
  Equation: `BatteryV = ADC_V * 6.144 / 0.0697674419`  
  ADC_V min: 20–70 mV → BatteryV min = ~0.29–1.00 V

- **Channel 1**:  
  Equation: `MeasuredAmps = ((ADC_V * 2) – 2.5) * 100`  
  ADC_V min: 20–70 mV → MeasuredAmps = −226 A to −166 A

- **Channel 2**:  
  Equation: `RPM = ADC_V * 2 * 6.144 * RPMScalingFactor`  
  ADC_V min: 20–70 mV → RPM = ~0.25–0.84 × scaling factor

- **Channel 3**:  
  Equation: `Channel3V = ADC_V * 10232.064`  
  ADC_V min: 20–70 mV → Channel3V = ~204–716V → exceeds sanity checks and always rejected in firmware

Conclusion: Only Channel 3 fails to operate near 0V due to excessive gain; other channels behave correctly within design limits.

# Validation Results Summary (Post-Review)

The following conclusions were drawn from detailed review and simulation:

- **Channel 0**: Fully operational. Battery voltages never drop low enough to hit the op-amp output floor. No action needed.

- **Channel 1 (Current Sensor)**:
  - Uses QNHC1K-21 200 A hall sensor (2.5V at 0 A, ±2V swing).
  - Voltage divider and op-amp comfortably support full ±200 A range.
  - ADC input is 0.25V to 2.25V after division. No clipping. Resolution is optimal.
  - Measuring near 0 A is inherently accurate (2.5V center point). Negative current is also measurable down to −200 A.

- **Channel 2 (RPM)**:
  - Functionally valid but final resolution is pending confirmation of `RPMScalingFactor`.
  - ADC and divider design will support wide RPM range once scaling is determined.
  # LM2907 Circuit Configuration Summary

The following summarizes the full electrical connections for the LM2907N-8 frequency-to-voltage converter circuit

---

## Power & Bypass

- **Pin 6 (V+)** → 5 V  
- **Pin 5 (COL)** → 5 V  
- **Pin 8 (GND)** → GND  
- **Bypass Capacitor:** 1 µF from Pin 6 (V+) to GND

---

## Timing & Filter Network

- **Pin 2 (CP1)** → 10 nF capacitor to GND  
- **Pin 3 (CP2 / IN+)** connected to:  
  - Three **1 µF** capacitors to GND (total capacitance = 3 µF)  
  - **25 kΩ** resistor to GND

---

## Output & Feedback Configuration

- **Pin 4 (EMIT)**:  
  - Connected to **Pin 7 (IN−)**  
  - Pulled to GND through a **10 kΩ** resistor  
  - Connected to the **ADS1115 analog input**  
- **Pin 7 (IN−)** → Connected to **Pin 4 (EMIT)**

---

## Input Signal Path (Pin 1 – TACH+)

The TACH input is derived from a single stator tap of an automotive alternator and processed as follows:

### Series path:
- Two **10 µF / 100 V** capacitors in parallel (AC coupling)  
- **4.7 kΩ** series resistor

### Shunt components at the node to Pin 1:
- **6.8 nF** capacitor to GND  
- **100 kΩ** resistor to GND  
- **SMBJ12CA bidirectional TVS diode**:  
  - **Cathode (pointy end)** → TACH+ net  
  - **Anode (flat end)** → GND

---

## Notes

- This configuration accepts a wide range of input amplitudes thanks to AC coupling and TVS clamping.
- The output from EMIT is read by ADC (ADS1115 with op amp etc.) and reflects frequency through a filtered voltage.


- **Channel 3 (Thermistor)**:
  - Confirmed support for 10kΩ NTC with 10kΩ fixed resistor and 3.3V supply.
  - Entire −40 °C to +125 °C range is measurable with ~0.1 V to ~3.25 V swing.
  - A 5V supply is not recommended unless re-scaling the divider.

