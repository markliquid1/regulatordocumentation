# Analog Inputs - ADS1115

## Overview

The 4 channel ADS1115 DAC with op-amp buffering provides high-impedance, accurate measurements of Battery Voltage, Alternator Current, Engine Speed, and Thermistor Temp with negligible power consumption.  The design is hardened for noisy marine environments.

##  Design Considerations

### Op-Amp Output Voltage Constraints

**Why Op-Amp Buffering is Required**: The high-impedance voltage dividers (typically 384kΩ to 1MΩ+) used to minimize current draw cannot directly drive the ADS1115 input without significant loading errors, especially if protection diodes are used. Op-amp buffers provide the necessary impedance transformation from high-impedance sources to low-impedance ADC inputs while at the same time constaining inputs to the safe range.

**However, this introduces an imporant limitation**: Even with the excellent rail-to-rail TLV9154IDR op-amp, the **minimum output voltage is approximately 10-20mV** (typical VOL specification). This directly limits the minimum measurable input values on all 4 ADS channels:

| Channel | Minimum Op-Amp Output | Minimum Measurable Input | Impact |
|---------|----------------------|-------------------------|---------|
| 0 (Battery) | ~10mV | 0.21V | No practical impact (batteries never this low) |
| 1 (Current) | ~10mV | -246A | No impact (within ±200A sensor range) |
| 2 (RPM) | ~10mV | 8Hz → 27-479 RPM | **min idle detection depends on signal freq** |
| 3 (Temperature) | ~10mV | Depends on configuration | **Limits cold temperature range** |

### ADS1115 Input Range Limitations

**Specification Clarity**: The ADS1115 specifications are easy to misunderstand:

- **Advertised range**: ±4.096V (with GAIN = 1 setting)
- **Reality**: Practical range for this use case is **0V to +3.3V maximum** due to single 3.3V supply rail
- **Negative voltages**: **Cannot be measured** - ADS1115 inputs must remain positive

These constraints must be considered for all channels.

---

## System Architecture

### Signal Chain Architecture
All analog input channels share a common architecture*:

```
Input Signal ──[R1]──┬──[R2]──GND
                     │
                  [5nF]──GND (Low-pass filter)
                     │
              [TLV9154IDR Input+]
                     │
              [TLV9154IDR Output]──ADS1115 Channel (0-3.3V only)
                     │
              [Feedback to Input-] (Unity gain)
```
* in the case of Channel 3 (Engine Speed), there is effectively no voltage divider, as R1=0 and R2 = anything.

### Component Functions

#### Voltage Divider (R1/R2)
- **Function**: Scales high input voltages to ADS1115-compatible range (0-3.3V maximum)
- **Power consumption**: Continuous, proportional to input voltage
- **Ratio selection**: Determines maximum measurable voltage and resolution
- **Precision**: ±0.1% tolerance for accuracy

#### Low-Pass Filter (5nF Capacitor)
- **Cutoff frequency**: f = 1 / (2π × R_thevenin × 5nF)
- **Function**: Removes high-frequency noise and EMI before amplification
- **Settling time**: ~5 × time constant for 99% accuracy
- **Placement**: Before op-amp buffer for optimal noise rejection

#### Unity-Gain Buffer (TLV9154IDR)
- **Function**: High input impedance to low output impedance conversion
- **Gain**: Exactly 1.0 (Vout = Vin)
- **Supply**: Single 3.3V rail
- **Configuration**: Non-inverting input from voltage divider, output feedback to inverting input
- **Input impedance**: >1MΩ (isolates high-impedance voltage dividers)
- **Critical limitation**: Minimum output ~10-20mV limits low-signal detection

#### ADC (ADS1115)
- **Resolution**: 16-bit with 0 to +3.3V input range (not ±4.096V as commonly stated)
- **Interface**: I2C communication
- **LSB resolution**: 0.1mV per count (3.3V ÷ 32768 counts)

### Single-Supply Design Benefits 

**Benefits**:
- **Inherent overvoltage protection**: Op-amp output cannot exceed 3.3V supply rails
- **No protection diodes needed**: Eliminates leakage current and temperature coefficient errors
- **Perfect accuracy**: No diode-related drift or offset issues
- **Low power consumption**: Single 3.3V supply shared with ADS1115

---

## Channel Specifications

### Channel 0: Battery Voltage Monitor

**Application**: battery voltage monitoring for 12V, 24V, and 48V 

#### Design Parameters
- **R1**: 1MΩ ±0.1%, 1/8W, 0805 SMD
- **R2**: 49.9kΩ ±0.1%, 1/8W, 0805 SMD  
- **Divider ratio**: 0.0475 (1:21.04 scaling)
- **Filter capacitor**: 5nF ±10%, X7R, 0603 SMD
- **Measurement range**: 0.21V to 65.2V
- **Minimum measurable**: 0.21V (limited by op-amp minimum output)

#### Performance Analysis
| Input Voltage | Divided Voltage | ADC Reading | ADC Resolution (LSB) | Engineering Resolution | Practical Accuracy |
|---------------|-----------------|-------------|---------------------|----------------------|-------------------|
| 0.21V | 0.010V | **Op-amp minimum** | 0.1mV | 0.0021V | Limited by noise/drift |
| 2.1V | 0.100V | Valid | 0.1mV | 0.0021V | 2.6% |
| 5.0V | 0.238V | Valid | 0.1mV | 0.0021V | 1.1% |
| 12.0V | 0.570V | Valid | 0.1mV | 0.0021V | 0.46% |
| 24.0V | 1.141V | Valid | 0.1mV | 0.0021V | 0.23% |
| 48.0V | 2.281V | Valid | 0.1mV | 0.0021V | 0.11% |
| 60.0V | 2.852V | Valid | 0.1mV | 0.0021V | 0.092% |
| 65.2V | 3.100V | Valid | 0.1mV | 0.0021V | 0.084% |

**Resolution**: 
- **ADC LSB resolution**: 0.1mV (3.3V ÷ 32768 counts)
- **Engineering resolution**: 0.0021V input (0.1mV ÷ 0.0475 divider ratio)  
- **Practical accuracy**: Limited by component tolerances (±0.1% resistors), temperature drift, and noise floor
- **Effective resolution**: ~55mV due to noise floor and component limitations

#### Filter Characteristics
- **Thevenin resistance**: 47.5kΩ
- **Cutoff frequency**: 670Hz
- **Time constant**: 0.24ms
- **99% settling time**: 1.2ms

#### Power Consumption Analysis
| Input Voltage | Divider Current | Divider Power | Total Power | Equivalent at 12V |
|---------------|-----------------|---------------|-------------|-------------------|
| 12.0V | 11.4µA | 0.14mW | 0.27mW | **22.5µA** |
| 24.0V | 22.9µA | 0.55mW | 0.68mW | **56.7µA** |
| 48.0V | 45.7µA | 2.19mW | 2.32mW | **193µA** |
| 60.0V | 57.1µA | 3.43mW | 3.56mW | **297µA** |

---

### Channel 1: Alternator Current Monitor

**Application**: QNHC1K-21 200A Hall Effect Current Sensor monitoring (2.5V @ 0A, ±2V swing for ±200A)

#### Design Parameters
- **R1**: 768kΩ ±0.1%, 1/8W, 0805 SMD
- **R2**: 768kΩ ±0.1%, 1/8W, 0805 SMD
- **Divider ratio**: 0.5000 (1:2 scaling)
- **Filter capacitor**: 5nF ±10%, X7R, 0603 SMD
- **Sensor voltage range**: 0.5V to 4.5V (±200A)
- **ADC voltage range**: 0.25V to 2.25V
- **Minimum measurable**: 0.02V sensor voltage (limited by op-amp, equivalent to -248A but real limit is -200 (sensor))

#### Performance Analysis
| Hall Sensor Voltage | Current (A) | Divided Voltage | ADC Reading | ADC Resolution (LSB) | True Current Resolution | Practical Accuracy | Measurable |
|--------------------|-----------|------------------|-------------|---------------------|----------------------|-------------------|------------|
| 0.5V | -200A | 0.250V | Valid | 0.1mV | 0.05A | ±1.04A | ✅ Valid |
| 2.5V | 0A | 1.250V | Valid | 0.1mV | 0.05A | ±1.04A | ✅ Valid |
| 4.5V | +200A | 2.250V | Valid | 0.1mV | 0.05A | ±1.04A | ✅ Valid |

**Resolution**:
- **ADC LSB resolution**: 0.1mV (3.3V ÷ 32768 counts)
- **Sensor scaling**: 100A/V (200A range ÷ 2V swing)
- **True current resolution**: 0.1mV ÷ 0.5 divider × 100A/V = **0.02A per LSB**
- **Practical accuracy**: ±1.04A represents the noise floor and worst-case measurement uncertainty, not the fundamental resolution
- **Noise sources**: Op-amp noise, ADC noise, sensor drift, temperature effects, EMI
- **Effective resolution**: While theoretical resolution is 0.02A, practical measurements are limited to ~1A accuracy due to system noise and sensor specifications

**Note**: Op-amp minimum output (10mV) corresponds to -246A, well outside the ±200A physical sensor limits, so this has no practical impact.

#### Filter Characteristics
- **Thevenin resistance**: 384kΩ
- **Cutoff frequency**: 83Hz
- **Time constant**: 1.9ms
- **99% settling time**: 9.6ms

#### Power Consumption Analysis
| Input Voltage | Divider Current | Divider Power | Total Power | Equivalent at 12V |
|---------------|-----------------|---------------|-------------|-------------------|
| 2.5V | 1.63µA | 4.07µW | 0.13mW | **10.8µA** |
| 4.0V | 2.60µA | 10.4µW | 0.14mW | **11.7µA** |

---

### Channel 2: Engine Speed Monitor

**Application**: LM2907 frequency-to-voltage converter output from alternator stator tap

####  Design Parameters
- **Direct connection**: LM2907 output directly to op-amp input (no voltage divider)
- **Filter capacitor**: 5nF ±10%, X7R, 0603 SMD
- **LM2907 output range**: 0V to 5V
- **Op-amp output range**: 0V to 3.3V (clipped by supply rail)
- **ADC voltage range**: 0V to 3.3V
- **Minimum measurable**: 0.01V LM2907 output (8Hz minimum frequency)
- **Maximum measurable**: 3.3V LM2907 output (2640Hz maximum frequency)

#### LM2907 Circuit Configuration

**Power & Bypass**
- **Pin 6 (V+)**: 5V supply
- **Pin 5 (COL)**: 5V supply  
- **Pin 8 (GND)**: System ground
- **Bypass capacitor**: 1µF from Pin 6 to ground

**Timing Components (Frequency-to-Voltage Conversion)**
- **Pin 2 (CP1)**: 10nF capacitor to ground
- **Pin 3 (CP2/IN+)**: 
  - Three 1µF capacitors to ground (total 3µF)
  - 25kΩ resistor to ground
- **Transfer function**: VO = VCC × fIN × C1 × R1 = 5V × fIN × 10nF × 25kΩ
- **Scaling factor**: **VO = fIN × 0.00125 V/Hz**

**Output Configuration**
- **Pin 4 (EMIT)**: Connected to Pin 7 and 10kΩ pull-down, feeds voltage divider
- **Pin 7 (IN−)**: Connected to Pin 4 (feedback)

**Input Signal Conditioning (Pin 1 – TACH+)**
- **AC coupling**: Two 10µF/100V capacitors in parallel (20µF total)
- **Series resistance**: **4.7kΩ, 2W (HP122WJ0472T4E)**
- **Input filtering**: 6.8nF capacitor to ground
- **Input termination**: 100kΩ resistor to ground
- **Overvoltage protection**: SMBJ12CA bidirectional TVS diode (12V clamp)

#### Input Signal Conditioning - Detailed Analysis

The LM2907 input circuit is an AC-coupled, attenuated input with complex frequency-dependent behavior. Proper analysis requires treating this as an AC circuit with both capacitive reactances and resistive elements.

**Circuit Topology:**
```
Stator Signal → 4.7kΩ series → 20µF coupling → LM2907 Pin 1
                                                    ↓
                                            (100kΩ || 6.8nF) to GND
                                                    ↓
                                              SMBJ12CA TVS
```

**High-Pass Filter Characteristics:**
- **Time constant**: τ = R_bias × C_coupling = 100kΩ × 20µF = **2.0 seconds**
- **Corner frequency**: f_c = 1/(2πτ) = 1/(2π × 100kΩ × 20µF) = **0.08 Hz**
- **Purpose**: Blocks DC offset, allows AC signal to pass

**Complex Impedance Analysis at Operating Frequencies:**

At typical alternator frequency (60Hz):
- **Coupling capacitor reactance**: X_c(20µF) = 1/(2π × 60Hz × 20µF) ≈ **133Ω**
- **Shunt capacitor reactance**: X_c(6.8nF) = 1/(2π × 60Hz × 6.8nF) ≈ **389kΩ**

**Parallel Load Impedance (100kΩ || 6.8nF at 60Hz):**
```
Z_parallel = (R_bias × Z_shunt) / (R_bias + Z_shunt)
Z_parallel = (100kΩ × (-j389kΩ)) / (100kΩ - j389kΩ)
|Z_parallel| ≈ 96.9kΩ at ∠-14.4°
```

**Total Circuit Impedance:**
```
Z_total = R_series + Z_coupling + Z_parallel
Z_total = 4.7kΩ + (-j133Ω) + (93.8kΩ - j24.1kΩ)
|Z_total| ≈ 101.5kΩ
```

**Frequency-Dependent Transfer Function:**

The combined transfer function includes both high-pass coupling and voltage division:
```
H(f) = H_coupling(f) × H_divider(f)

where:
H_coupling(f) = [j2πf × τ] / [1 + j2πf × τ]  (high-pass from AC coupling)
H_divider(f) = Z_load(f) / [Z_series + Z_coupling(f) + Z_load(f)]
```

**Signal Attenuation vs. Frequency:**

| Frequency | X_c(20µF) | X_c(6.8nF) | Coupling Gain | Divider Gain | Total Gain | Attenuation | Required Input* |
|-----------|-----------|------------|---------------|--------------|------------|-------------|-----------------|
| 5 Hz | 1592Ω | 4.68MΩ | 0.628 | 0.955 | 0.600 | 40% | 41.7mVpp |
| 10 Hz | 796Ω | 2.34MΩ | 0.783 | 0.955 | 0.748 | 25% | 33.4mVpp |
| 50 Hz | 159Ω | 468kΩ | 0.950 | 0.955 | 0.907 | 9.3% | 27.6mVpp |
| 60 Hz | 133Ω | 389kΩ | 0.958 | 0.955 | 0.915 | 8.5% | 27.3mVpp |
| 100 Hz | 80Ω | 234kΩ | 0.975 | 0.955 | 0.931 | 6.9% | 26.8mVpp |
| 1000 Hz | 8Ω | 23.4kΩ | 0.998 | 0.955 | 0.953 | 4.7% | 26.2mVpp |
| 2000 Hz | 4Ω | 11.7kΩ | 0.999 | 0.955 | 0.954 | 4.6% | 26.2mVpp |

\* Required stator amplitude to produce 25mVpp at LM2907 Pin 1 (minimum detection threshold)

**Key Observations:**
1. **At high frequencies** (>100Hz): Coupling capacitor has minimal impedance, attenuation approaches resistive divider limit (~4.7%)
2. **At mid frequencies** (50-100Hz): Combined attenuation ~7-9% due to coupling capacitor reactance
3. **At low frequencies** (<10Hz): Significant high-pass attenuation, requires much stronger input signals
4. **Shunt capacitor** (6.8nF): Negligible effect below 1kHz, provides high-frequency noise filtering

**TVS Clamping Analysis:**

The SMBJ12CA TVS begins conducting when node voltage exceeds approximately 13V. Node peak voltage depends on input amplitude and frequency:

| Input Amplitude | Node Peak @ 60Hz | TVS Status | Impact |
|-----------------|------------------|------------|--------|
| 12V pp | 5.73V | No clamp | Normal operation |
| 24V pp | 11.46V | No clamp | Normal operation |
| 48V pp | 22.91V | **Clamps** | Limits peaks, reduces power |
| 60V pp | 28.64V | **Clamps** | Heavy clamping, significant power |

At 48V pp and above, TVS clamping occurs during signal peaks, providing overvoltage protection and reducing actual power dissipation below calculated linear values.

#### LM2907 Recovery Time After Field Dropout

**Problem Description:**
When the alternator field is cut off (load dump protection or shutdown), the RPM reading drops to zero and takes 5-10 seconds to recover even though actual zero-crossing loss is only ~300ms. This is caused by the AC coupling capacitor charge retention.

**Coupling Capacitor Discharge Mechanism:**

When a large signal is present, the 20µF coupling capacitor charges to a DC level corresponding to the signal amplitude. When the signal suddenly drops (field cutoff), this stored charge must discharge through the 100kΩ bias resistor before the LM2907 can detect the restored signal.

**Exponential Decay:**
```
V_offset(t) = V_initial × e^(-t/τ)

where τ = R_bias × C_coupling = 100kΩ × 20µF = 2.0 seconds
```

**Recovery Time to Detection Threshold:**

The LM2907 requires approximately 25mVpp at Pin 1 for reliable frequency detection. Recovery is defined as the time until the residual DC offset has decayed enough for the restored signal to exceed this threshold.

```
Recovery fraction α = 1 - (V_threshold / V_initial)
t_recovery = -τ × ln(1 - α)
```

**Recovery Timing:**

| Recovery Level | Time Factor | Recovery Time | Practical Meaning |
|----------------|-------------|---------------|-------------------|
| 63% recovered | 1.0τ | 2.0 seconds | One time constant |
| 90% recovered | 2.3τ | **4.6 seconds** | **Practical recovery** |
| 95% recovered | 3.0τ | 6.0 seconds | Near-complete |
| 99% recovered | 4.6τ | 9.2 seconds | Essentially complete |

**Observed Behavior:**
- Field dropout creates 5-10 second RPM reading gap
- Actual zero-crossing signal loss is only ~300ms
- Remaining 4.6-9.2 seconds is coupling capacitor discharge time
- This matches the exponential recovery prediction of 2.3τ to 4.6τ

**Why Component Values Were Not Changed:**

An alternative bias resistor value of 10kΩ would reduce the time constant to 0.2 seconds (recovery time ~0.46 seconds), eliminating the dropout problem. However, testing revealed that fixing one problem caused another, and a better balance could not be found. 

Speculation: The lower bias resistance (10kΩ) likely created issues with:
- Reduced signal sensitivity (40% gain reduction requiring stronger input signals)
- Potential high-frequency instability or noise pickup with lower impedance node
- Changed signal-to-noise ratio affecting detection reliability at idle speeds
- TVS clamping occurring at lower input voltages (16V peak at 48V pp input vs 23V with 100kΩ)

**Solution - Software Field Management:**

Rather than compromise the hardware circuit balance, the recovery time issue is addressed in software by maintaining a small alternator field current at all times. This field current is sufficient to generate a detectable stator signal for RPM measurement but small enough that no charging occurs (effectively "off" from a charging perspective). This ensures continuous RPM readings without the 5-10 second dropout after field transitions.

#### Power Dissipation Analysis

**AC Operation (Normal Conditions):**

Under normal AC operation, power dissipation must be calculated using complex impedance analysis with RMS voltage and current values.

**At 12V Peak-to-Peak (4.24V RMS, 60Hz):**
```
|Z_total| = 101.5kΩ
I_rms = 4.24V / 101.5kΩ = 41.8µA
P_series = (41.8µA)² × 4.7kΩ = 0.008mW
P_bias = |V_node_rms|² / R_bias = 0.16mW
P_total = 0.17mW
```

**At 24V Peak-to-Peak (8.49V RMS, 60Hz):**
```
I_rms ≈ 0.106mA
P_series = (0.106mA)² × 4.7kΩ = 0.053mW
P_bias = 0.71mW
P_total = 0.76mW
```

**At 48V Peak-to-Peak (16.97V RMS, 60Hz) - TVS Clamping Occurs:**
```
Without TVS: I_rms ≈ 0.21mA, P_bias would be ~2.9mW
With TVS clamping: Node voltage limited to ~13V peaks
Actual P_bias ≈ 2-3mW (reduced by clamping)
P_series ≈ 0.2mW
```

**Summary - AC Power Dissipation:**

| Input Amplitude | Frequency | Bias R Power | Series R Power | Total | TVS Status |
|-----------------|-----------|--------------|----------------|-------|------------|
| 12V pp (4.24V RMS) | 60Hz | 0.16mW | 0.008mW | 0.17mW | No clamp |
| 24V pp (8.49V RMS) | 60Hz | 0.71mW | 0.053mW | 0.76mW | No clamp |
| 48V pp (16.97V RMS) | 60Hz | ~2-3mW* | 0.2mW | ~2.2-3.2mW | **Clamps** |
| 60V pp (21.21V RMS) | 60Hz | ~3-4mW* | 0.3mW | ~3.3-4.3mW | **Clamps** |

\* TVS clamping reduces power dissipation below linear calculation

**DC Fault Conditions (Engine-Off with Rectifier Leakage):**

If DC voltage appears on the stator tap (fault condition), the coupling capacitor will charge to the input voltage and steady-state DC current flows through the bias resistor:

**Steady-State DC Analysis (After Capacitor Charged):**
```
I_dc = V_input / R_bias
P_bias = V_input² / R_bias
P_series ≈ 0 (no current through series resistor after capacitor charges)
```

| DC Input | Steady-State Current | Bias Resistor Power | Status |
|----------|---------------------|---------------------|---------|
| 12V | 120µA | 1.44mW | Negligible |
| 14V | 140µA | 1.96mW | Safe |
| 24V | 240µA | 5.76mW | Safe |
| 48V | 480µA | 23.0mW | Safe |

**Transient DC Fault (Before Capacitor Charges):**

During the initial moment of a DC fault before the coupling capacitor charges, current flows through both resistors in series. The TVS provides protection:

```
I_transient = V_input / (R_series + R_bias) 
            = V_input / 104.7kΩ (if TVS doesn't clamp)
```

| DC Input | Maximum Transient I | Series R Power | TVS Protection | Status |
|----------|---------------------|----------------|----------------|---------|
| 14V | 134µA | 0.08mW | No clamp needed | ✅ Safe |
| 28V | 267µA | 0.33mW | No clamp needed | ✅ Safe |
| 56V | 535µA | 1.34mW | TVS clamps | ✅ Safe |

**Component Ratings:**
- **4.7kΩ series resistor**: HP122WJ0472T4E rated at 2W - massive safety margin
- **100kΩ bias resistor**: 0.1W rated - adequate for all operating conditions
- Maximum power in bias resistor: ~23mW at 48V DC fault (4.3× safety margin)
- Normal AC operation: <1mW in bias resistor under typical conditions

#### Normal AC Operation (3V to 58V inputs)
- **No damage risk**: AC signals don't cause sustained power dissipation above safe limits
- **TVS clipping**: Signals above ~48V pp get peak-clamped to ~13V by SMBJ12CA, maintaining proper LM2907 operation while protecting the circuit
- **Circuit robustness**: Safe operation with input signals from 3V to 58V+ amplitude

#### Filter Characteristics (Optimized - No Voltage Divider)
- **Thevenin resistance**: ~0Ω (direct connection from LM2907)
- **Cutoff frequency**: 6.4MHz (effectively no filtering of alternator signals)
- **Purpose**: Provides minimal high-frequency noise protection without signal attenuation

#### Frequency and RPM Analysis

**Engine-to-Stator Scaling Factors:**
- **Conservative case**: 6-pulse alternator, 1.5:1 belt ratio → **Engine RPM × 0.15 = Stator Hz**
- **Aggressive case**: 7-pulse alternator, 2.5:1 belt ratio → **Engine RPM × 0.292 = Stator Hz**  
- **Worst case**: 1-pulse per revolution, 1:1 direct → **Engine RPM × 0.0167 = Stator Hz**

#### Performance Analysis (Conservative Case Example)

**Minimum detectable frequency**: 8Hz 
**Maximum detectable frequency**: 2640Hz (limited by 3.3V op-amp supply rail)

| Engine RPM | Stator Frequency | LM2907 Output | ADC Voltage | Status |
|------------|------------------|---------------|-------------|---------|
| 53 RPM | 8Hz | 0.010V | 0.010V | **Minimum detectable** |
| 133 RPM | 20Hz | 0.025V | 0.025V | Valid |
| 300 RPM | 45Hz | 0.056V | 0.056V | Valid |
| 600 RPM | 90Hz | 0.113V | 0.113V | Valid |
| 1500 RPM | 225Hz | 0.281V | 0.281V | Valid |
| 3000 RPM | 450Hz | 0.563V | 0.563V | Valid |
| 4000 RPM | 600Hz | 0.750V | 0.750V | Valid |
| 6000 RPM | 900Hz | 1.125V | 1.125V | Valid |
| 8000 RPM | 1200Hz | 1.500V | 1.500V | Valid |
| 10667 RPM | 1600Hz | 2.000V | 2.000V | Valid |
| 14667 RPM | 2200Hz | 2.750V | 2.750V | Valid |
| 17600 RPM | 2640Hz | 3.300V | 3.300V | **Maximum detectable** |

**RPM Range Summary (Significantly Improved)**:
- **Conservative (6-pulse, 1.5:1)**: **53 RPM** to **17,600 RPM** 
- **Aggressive (7-pulse, 2.5:1)**: **27 RPM** to **9,041 RPM**   
- **1-pulse direct**: **479 RPM** to **158,400 RPM** 

#### Input Signal Requirements

**Minimum detectable signal**: ~25mVpp at Pin 1 after attenuation (LM2907 worst-case threshold per datasheet interpretation)

**Required Stator Signal vs. Frequency:**

Due to high-pass filter attenuation at low frequencies, the required stator amplitude varies with frequency:

| Frequency | Required Stator | Signal at Pin 1 | High-Pass Loss | Detection Status | Engine RPM (Conserv.) |
|-----------|-----------------|-----------------|----------------|------------------|---------------------|
| 5Hz | 41.7mVpp | 25mVpp | 40% | Marginal | 33 RPM |
| 10Hz | 33.4mVpp | 25mVpp | 25% | ✅ Good | 67 RPM |
| 20Hz | 28.0mVpp | 25mVpp | 11% | ✅ Good | 133 RPM |
| 50Hz | 27.6mVpp | 25mVpp | 9.3% | ✅ Good | 333 RPM |
| 100Hz | 26.8mVpp | 25mVpp | 6.9% | ✅ Excellent | 667 RPM |
| 500Hz | 26.3mVpp | 25mVpp | 5.0% | ✅ Excellent | 3333 RPM |
| 1000Hz | 26.2mVpp | 25mVpp | 4.7% | ✅ Excellent | 6667 RPM |

**Frequency Response Impact:**
- **Above 100Hz**: Minimal attenuation (~5-7%), excellent sensitivity
- **50-100Hz**: Moderate attenuation (~7-9%), good sensitivity
- **10-50Hz**: Increasing attenuation (10-25%), requires stronger signals
- **Below 10Hz**: Significant attenuation (>25%), marginal detection

This explains why very low RPM detection depends on stator signal strength - the high-pass filter causes 2× signal requirement at 5Hz compared to 100Hz.

**Input signal analysis for 100mVpp stator input:**

| Frequency | Signal at Pin 1 | Detection | Engine RPM (Conservative) | Engine RPM (1-pulse) |
|-----------|-----------------|-----------|---------------------------|---------------------|
| 50Hz | 90.7mVpp | ✅ Excellent | 333 RPM | 3000 RPM |
| 100Hz | 93.1mVpp | ✅ Excellent | 667 RPM | 6000 RPM |
| 500Hz | 95.3mVpp | ✅ Excellent | 3333 RPM | 30000 RPM |
| 800Hz | 95.4mVpp | ✅ Excellent | 5333 RPM | 48000 RPM |
| 1000Hz | 95.3mVpp | ✅ Excellent | 6667 RPM | 60000 RPM |

**Very Low Frequency Analysis (1-pulse systems):**

**Minimum detectable**: 8Hz = 479 RPM for 1-pulse systems 

| Engine RPM | Frequency | Required Stator | Signal at Pin 1 | Detection Status |
|------------|-----------|-----------------|-----------------|------------------|
| 479 RPM | 8Hz | ~50mVpp | 25mVpp | **Minimum detectable** |
| 1500 RPM | 25Hz | 28.8mVpp | 25mVpp | ✅ Good if signal present |
| 3000 RPM | 50Hz | 27.6mVpp | 25mVpp | ✅ Good |
| 6000 RPM | 100Hz | 26.8mVpp | 25mVpp | ✅ Excellent |
| 18000 RPM | 300Hz | 26.3mVpp | 25mVpp | ✅ Excellent |
| 30000 RPM | 500Hz | 26.3mVpp | 25mVpp | ✅ Excellent |
| 60000 RPM | 1000Hz | 26.2mVpp | 25mVpp | ✅ Excellent |
| 95040 RPM | 1584Hz | 26.2mVpp | 25mVpp | ✅ Excellent |
| 158400 RPM | 2640Hz | 26.2mVpp | 25mVpp | **Maximum detectable** |

**Sine vs. Square Wave Performance**: Both waveform types perform identically for frequency detection. The LM2907 detects zero crossings regardless of waveform shape, and AC coupling affects both equally at very low frequencies.

---

### Channel 3: Temperature Monitor

**Application**: 10kΩ NTC thermistor (Murata NXFT15XH103FA2B050) for engine or ambient temperature

#### Design Parameters (Current Configuration)
- **Supply voltage**: 5V
- **Pull-up resistor**: 10kΩ ±1%, 0805 SMD
- **Thermistor**: 10kΩ @ 25°C, B=3380K, ±0.5%
- **Filter capacitor**: 5nF ±10%, X7R, 0603 SMD
- **No voltage divider**: Direct connection to op-amp buffer input
- **Measurement range**: 15°C to 125°C (limited by 3.3V ADC maximum)

#### Circuit Topology
```
5V ──[10kΩ pullup]──┬──[5nF filter]──GND
                    │
              [NTC Thermistor to GND]
                    │
            [Op-amp buffer input]
```

**Voltage division**: V_sense = 5V × R_NTC / (10kΩ + R_NTC)

#### Performance Analysis
| Temperature | R_NTC | V_sense | ADC Input | ADC Reading | Status |
|-------------|-------|---------|-----------|-------------|--------|
| -40°C | 119.4kΩ | 4.61V | **3.3V max** | Clipped | ❌ Cannot measure |
| -20°C | 52.7kΩ | 4.20V | **3.3V max** | Clipped | ❌ Cannot measure |
| 0°C | 27.3kΩ | 3.66V | **3.3V max** | Clipped | ❌ Cannot measure |
| 15°C | 18.6kΩ | 3.25V | 3.25V | Valid | ✅ Minimum measurable |
| 25°C | 10.0kΩ | 2.50V | 2.50V | Valid | ✅ Valid |
| 50°C | 3.60kΩ | 1.32V | 1.32V | Valid | ✅ Valid |
| 75°C | 1.39kΩ | 0.61V | 0.61V | Valid | ✅ Valid |
| 100°C | 0.58kΩ | 0.27V | 0.27V | Valid | ✅ Valid |
| 125°C | 0.26kΩ | 0.13V | 0.13V | Valid | ✅ Valid |

**Resolution**:
- **ADC LSB resolution**: 0.1mV (3.3V ÷ 32768 counts)
- **Temperature resolution**: Variable, best at mid-range (~0.03°C per LSB at 25°C)
- **Practical accuracy**: ±1°C including thermistor tolerance, ADC noise, and self-heating

#### Filter Characteristics
- **Thevenin resistance**: 5kΩ (10kΩ || 10kΩ at 25°C, varies with temperature)
- **Cutoff frequency**: ~6.4kHz at 25°C
- **Time constant**: ~25µs
- **99% settling time**: ~125µs

#### Power Consumption Analysis
| Temperature | R_NTC | Circuit Current | Power | Equivalent at 12V |
|-------------|-------|-----------------|-------|-------------------|
| -40°C | 119.4kΩ | 38.6µA | 193µW | **16.1µA** |
| 25°C | 10.0kΩ | 250µA | 1.25mW | **104µA** |
| 50°C | 3.60kΩ | 367µA | 1.84mW | **153µA** |
| 75°C | 1.39kΩ | 440µA | 2.20mW | **183µA** |
| 100°C | 0.58kΩ | 473µA | 2.36mW | **197µA** |
| 125°C | 0.26kΩ | 488µA | 2.44mW | **203µA** |

**Power Characteristics**:
- **Temperature coefficient**: Power consumption increases with temperature
- **Self-heating**: Negligible (<0.1°C error) due to low power dissipation
- **Power impact**: 115µA at 25°C, up to 209µA at 125°C
- **Supply requirement**: Currently 5V (creates ADC range limitations)
- **Design limitation**: Cold temperatures below ~15°C cannot be measured due to voltage exceeding 3.3V ADC maximum

### System Integration Notes
- **Total power consumption**: 193µA equivalent at 12V (with 5V Channel 3)
- **Battery life impact**: Minimal (<0.01% daily consumption on 100Ah battery)
- **Environmental rating**: Suitable for marine applications (-40°C to +85°C)

### Recommended Design Improvements

**Channel 3 Limitation** (current 5V design):
- Current 5V/10kΩ configuration cannot measure temperatures below ~15°C
- Consider 3.3V/15kΩ configuration for full -40°C to +125°C range
- Would reduce power consumption by ~40%
- Would enable full cold temperature measurement capability


# TLV9154 Op-Amp Failure Analysis and Protection Fix

## Failure Description

**Failure Pattern:**
- Two TLV9154 op-amps failed with similar symptoms
- One showed visible burn damage centered around ground pin (Pin 11)
- Second showed no visible damage but produced garbage ADC readings
- One failure occurred with engine off (no input signals active)
- Neither circuit used the Thermistor channel.


## Root Cause Analysis

### Theory 1: Power Sequencing Issue

**Problem:** Mixed voltage design with inadequate input protection
- **3.3V supply** powers TLV9154 op-amp (Pin 4 = V+, Pin 11 = GND)
- **5V signals** directly connected to op-amp inputs:
  - Channel 2 (Pin 10): LM2907 output (0-5V range)
  - Channel 3 (Pin 12): Thermistor voltage divider with 5V pullup

**Failure Mechanism:**
1. During power-up, 5V rail establishes before 3.3V rail
2. Op-amp inputs receive 5V while supply pins are unpowered (0V)
3. Internal ESD protection diodes become forward-biased
4. Current path: 5V source → Input pin → ESD diode → unpowered supply rail
5. ESD diodes exceed safe current limits, causing thermal damage
6. Ground pin (Pin 11) becomes current exit point, showing burn damage

### Current Calculations

**Without Protection (Original Design):**
- Channel 2: 5V directly to Pin 10, limited only by LM2907 output impedance
- Channel 3: 5V through 10kΩ thermistor pullup = 0.5mA potential current
- Internal ESD diodes not rated for continuous current → thermal failure

**Systematic Failure:** Two similar failures indicate design issue, not random component failure

## Secondary Issue: Grounding Architecture

### Theory #2: Current Grounding Problem

**Problematic Design:**
- **ADS1115 ground** connected to battery negative through INA228 Shunt- terminal separate Cat6 wire strand (thin wire)
- **TLV9154 ground** connected to main PCB ground plane
- **INA228** grounded normally to ground plane, but Shunt- goes to battery via Cat6
Ground Plane connected by battery bank - by a thick wire, say 12 gauge.

**Why This Is Poor Design:**

*Kelvin Sensing Misuse:*
- INA228 Shunt+/Shunt- are designed for **Kelvin sensing** (voltage measurement only)
- Should carry **no return current** from other circuits
- Using Shunt- as ground reference for ADS1115 violates this principle

*Ground Impedance Mismatch:*
- **Main ground plane:** Low impedance, wide copper area
- **Cat6 wire:** Higher resistance, inductance, and voltage drop
- Creates **different ground potentials** between TLV9154 and ADS1115

### Potential Contribution to Failure

**Ground Loop Analysis:**

*Steady-State Voltage Drop:*
- Cat6 wire resistance: ~0.1Ω (typical)
- ADS1115 current: 150µA
- Voltage drop: 150µA × 0.1Ω = **15µV**

*Transient Current Problem:*
- The steady-state drop calculation is **irrelevant** to the failure
- The problem is **transient return**: when the ADS input clamps conduct (during spikes/over-range/power-sequence)
- **Amps** of instantaneous current can try to return through that skinny "Kelvin" path
- Forces a ground delta and dumps heat at the TLV9154 V− bond
- Note from Mark, i don't believe this BS, but doesn't change the result.

*Impact Assessment:*
- **15µV steady-state offset** not sufficient to cause op-amp damage directly
- **Transient ground currents** during fault conditions could contribute to thermal damage
- **Bad practice** that violates current sensing design principles

### Recommended Grounding Fix

**Proper Architecture:**
1. **Connect ADS1115 AGND** to main PCB ground plane (same as TLV9154)
2. **Use INA228 Shunt± only for Kelvin sensing** (no current return path)
3. **Route main power return** through dedicated thick wire/plane
4. **Keep sensing and power grounds separate** until single-point connection


## Protection Solution for Theory #1

### Input Protection Resistors

**Value:** 4.7kΩ in series with each TLV9154 non-inverting input
**Tolerance:** 1% or 5% (tolerance not critical for protection function)

**Pins to Protect:**
- Pin 3 (Channel 0 - Battery voltage)
- Pin 5 (Channel 1 - Current sensor)
- Pin 10 (Channel 2 - LM2907 RPM) ← **Critical for this failure due to 5V supply**
- Pin 12 (Channel 3 - Temperature) ← **Critical for this failure due to 5V supply**

**Protection Calculations:**

*Fault Current with 4.7kΩ Protection:*
- **Op-amp powered (3.3V):** (5V - 3.3V - 0.3V) / 4.7kΩ = **0.3mA** → Safe
- **Op-amp unpowered (0V):** (5V - 0.3V) / 4.7kΩ = **1.0mA** → Safe (survives)

*Accuracy Impact:*
- Op-amp input bias current: ~10pA typ
- Voltage drop: 10pA × 4.7kΩ = **0.047µV** → Negligible
- Existing voltage dividers have much higher source impedance anyway

### Output Protection Resistors

**Value:** 330Ω in series with each TLV9154 output to ADS1115
**Tolerance:** 1% or 5%

**Pins to Protect:**
- Pin 1 → ADS1115 AIN0
- Pin 7 → ADS1115 AIN1  
- Pin 8 → ADS1115 AIN2
- Pin 14 → ADS1115 AIN3

**Protection Calculations:**

*Fault Current (ADS1115 unpowered):*
- Maximum op-amp output: 3.3V
- Current into ADS1115 input clamp: (3.3V - 0.3V) / 330Ω = **9.1mA**
- ADS1115 specification: ±10mA continuous maximum → **Within spec**

*Accuracy Impact:*
- ADS1115 input impedance: Very high during sampling
- Voltage drop during normal operation: **Effectively zero**


## Implementation Summary

### Required Changes
1. **Add 4.7kΩ resistors** in series with TLV9154 inputs (Pins 3, 5, 10, 12)
2. **Add 330Ω resistors** in series with TLV9154 outputs (Pins 1, 7, 8, 14)
3. **Fix ADS1115 grounding** to use main ground plane (separate issue)
4. **Extra cap** 0.1 µF X7R from V+↔V− at the pins (also keep the 10 µF bulk). This is standard power integrity for op-amps.


### Expected Results
- **Prevents power sequencing failures** (primary cause of current failures)
- **Limits fault currents to safe levels** in all operating conditions
- **Maintains measurement accuracy** (protection resistors have negligible impact)
- **Provides ESD protection** during assembly and field service

## Design Validation

**Power Sequencing Test:**
- Verify 5V and 3.3V startup timing with oscilloscope
- Confirm protection resistors limit current during sequencing faults
- Test system behavior with various power-up sequences

**Fault Testing:**
- Apply 5V to individual inputs with op-amp unpowered
- Verify current limitation and no damage
- Confirm normal operation after fault conditions

**Critical Insight:** Reference circuit designs often omit protection resistors, assuming ideal operating conditions. Real-world applications require protection against predictable fault modes like power sequencing issues.
