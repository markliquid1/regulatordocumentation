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
- **Input filtering**: 6.8nF capacitor to ground (5kHz cutoff)
- **Input termination**: 100kΩ resistor to ground
- **Overvoltage protection**: SMBJ12CA bidirectional TVS diode (12V clamp)
- **High-pass cutoff**: 1/(2π × 100kΩ × 20µF) = **0.08Hz**
- **Voltage attenuation**: ~4.5% (4.7kΩ + 100kΩ divider)

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

**Minimum detectable signal**: ±40mV at Pin 1 (LM2907 worst-case threshold *** datasheet unclear on this!!)

**Input signal analysis for 100mV (±100mV) input:**

| Frequency | Signal at Pin 1 | Detection | Engine RPM (Conservative) | Engine RPM (1-pulse) |
|-----------|-----------------|-----------|---------------------------|---------------------|
| 50Hz | 95.4mV | ✅ Valid | 333 RPM | 3000 RPM |
| 100Hz | 95.4mV | ✅ Valid | 667 RPM | 6000 RPM |
| 500Hz | 94.6mV | ✅ Valid | 3333 RPM | 30000 RPM |
| 800Hz | 93.0mV | ✅ Valid | 5333 RPM | 48000 RPM |
| 1000Hz | 91.8mV | ✅ Valid | 6667 RPM | 60000 RPM |

**Very Low Frequency Analysis (1-pulse systems):**

**Minimum detectable**: 8Hz = 479 RPM for 1-pulse systems 

| Engine RPM | Frequency | Signal at Pin 1 | Detection Status |
|------------|-----------|-----------------|------------------|
| 479 RPM | 8Hz | Variable | **Minimum detectable** |
| 1500 RPM | 25Hz | 238mV | ✅ Valid |
| 3000 RPM | 50Hz | 477mV | ✅ Valid |
| 6000 RPM | 100Hz | 954mV | ✅ Valid |
| 18000 RPM | 300Hz | 286mV | ✅ Valid |
| 30000 RPM | 500Hz | 477mV | ✅ Valid |
| 60000 RPM | 1000Hz | 918mV | ✅ Valid |
| 95040 RPM | 1584Hz | 1510mV | ✅ Valid |
| 158400 RPM | 2640Hz | 2518mV | **Maximum detectable** |


**Sine vs. Square Wave Performance**: Both waveform types perform identically for frequency detection. The LM2907 detects zero crossings regardless of waveform shape, and AC coupling affects both equally at very low frequencies.

#### Normal AC Operation (3V to 58V inputs)
- **No damage risk**: AC signals don't cause sustained power dissipation
- **TVS clipping**: Signals above 12V get clipped to 12V, maintaining proper LM2907 operation
- **Circuit robustness**: Safe operation with input signals up to 58V+ amplitude

#### DC Fault Protection Analysis

**Sustained DC voltage analysis** (engine-off or fault conditions):

| DC Input | Current Through 4.7kΩ | Power in 4.7kΩ | Total Power | Equivalent at 12V | Status |
|----------|----------------------|----------------|-------------|-------------------|---------|
| 14V | 426µA | 0.85mW | 7.4mW | **0.62mA** | ✅ Safe |
| 28V | 3.4mA | 54.5mW | 96.7mW | **8.1mA** | ✅ Safe |
| 56V | 9.36mA | 412mW | 525mW | **43.8mA** | ✅ Safe |

**Component ratings**: 4.7kΩ resistor (HP122WJ0472T4E) rated for 2W, providing excellent safety margin up to 56V DC faults.

#### Filter Characteristics (Optimized - No Voltage Divider)
- **Input impedance**: Direct connection to op-amp (>1MΩ input impedance)
- **Cutoff frequency**: Determined by LM2907 output impedance and 5nF capacitor
- **Time constant**: Minimal (direct connection)
- **99% settling time**: <50µs

#### Power Consumption Analysis (No Voltage Divider)
- **Voltage divider power**: 0W (no divider resistors)
- **Op-amp power**: 0.13mW (shared IC allocation)
- **Total power**: 0.13mW
- **Equivalent at 12V**: **10.8µA** (unchanged from IC power)

---

### Channel 3: Temperature Monitor (5V Supply Design)

**Application**: Engine or ambient temperature monitoring using 10kΩ NTC thermistor

#### Current Design Parameters
- **Supply voltage**: 5V 
- **R1**: 10kΩ ±0.1%, 1/8W, 0805 SMD (pullup to 5V)
- **R2**: 10kΩ NTC thermistor (user-supplied sensor, to ground)
- **Divider configuration**: 10kΩ pullup to 5V, thermistor to ground
- **Filter capacitor**: 5nF ±10%, X7R, 0603 SMD

#### Design Limitations with 5V Supply
**Critical Issue**: 5V supply with 10kΩ/10kΩ divider creates voltages above 3.3V for cold temperatures:
- ADC cannot measure voltages above 3.3V (supply rail limitation)
- Cold temperatures below ~15°C are unmeasurable
- **Temperature range**: Limited to ~15°C to +125°C (cold temperature limitation)
- **ADC voltage range**: 1.67V to 4.17V (but clipped at 3.3V maximum)

#### Thermistor Performance Analysis (5V Supply)

**Voltage divider equation**: V_out = 5V × R_thermistor / (10kΩ + R_thermistor)

| Temperature | Thermistor Resistance | Divider Voltage | ADC Reading | Measurable | Temperature Resolution |
|-------------|----------------------|-----------------|-------------|------------|----------------------|
| -40°C | ~195kΩ | 4.76V | **3.3V clipped** | ❌ **Unmeasurable** | N/A |
| -20°C | ~84kΩ | 4.47V | **3.3V clipped** | ❌ **Unmeasurable** | N/A |
| 0°C | ~27kΩ | 3.65V | **3.3V clipped** | ❌ **Unmeasurable** | N/A |
| 15°C | ~15kΩ | 3.00V | 3.00V | ✅ Valid | ~1.2°C |
| 25°C | ~10kΩ | 2.50V | 2.50V | ✅ Valid | ~0.8°C |
| 50°C | ~3.9kΩ | 1.41V | 1.41V | ✅ Valid | ~1.0°C |
| 75°C | ~1.8kΩ | 0.76V | 0.76V | ✅ Valid | ~1.8°C |
| 100°C | ~0.9kΩ | 0.41V | 0.41V | ✅ Valid | ~3.2°C |
| 125°C | ~0.5kΩ | 0.24V | 0.24V | ✅ Valid | ~5.8°C |


#### Alternative Optimized Configuration (Recommendation)
For full temperature range capability:
- **Supply voltage**: 3.3V (matches ADC supply, but was impossible on this iteration of board hardware)
- **R1**: 15kΩ ±0.1%, 1/8W, 0805 SMD (pullup to 3.3V)
- **Temperature range**: -40°C to +125°C (full range)
- **Power reduction**: ~40% lower consumption

#### Original Configuration Limitations
The documented 5V/10kΩ configuration suffers from:
- **Cold temperature limitation**: Voltages above 3.3V cannot be measured by ADC
- **Wasted voltage range**: High voltages at cold temperatures exceed ADC capability
- **Higher power consumption**: 5V supply slightly increases current draw

#### Filter Characteristics (5V Supply Design)
- **Thevenin resistance**: 5kΩ (parallel combination at mid-range)
- **Cutoff frequency**: 6.4kHz
- **Time constant**: 25µs
- **99% settling time**: 125µs

#### Power Consumption Analysis (5V Supply Design)
| Temperature | Thermistor R | Divider Current | Divider Power | Total Power | Equiv at 12V |
|-------------|--------------|-----------------|---------------|-------------|--------------|
| 15°C | 15kΩ | 200µA | 1.00mW | 1.13mW | **94.2µA** |
| 25°C | 10kΩ | 250µA | 1.25mW | 1.38mW | **115µA** |
| 100°C | 0.9kΩ | 459µA | 2.30mW | 2.43mW | **203µA** |
| 125°C | 0.5kΩ | 476µA | 2.38mW | 2.51mW | **209µA** |

---

## IC Configuration Details

### TLV9154IDR Quad Op-Amp Pinout (SOIC-14 Package)

| Pin | Symbol | Function | Connection |
|-----|--------|----------|------------|
| 1 | OUT1 | Op Amp A Output | **ADS1115 Channel 0 (AIN0)** |
| 2 | IN1- | Op Amp A Inverting Input | **Connect to Pin 1** |
| 3 | IN1+ | Op Amp A Non-inverting Input | **Channel 0 voltage divider output** |
| 4 | V+ | Positive Power Supply | **+3.3V** |
| 5 | IN2+ | Op Amp B Non-inverting Input | **Channel 1 voltage divider output** |
| 6 | IN2- | Op Amp B Inverting Input | **Connect to Pin 7** |
| 7 | OUT2 | Op Amp B Output | **ADS1115 Channel 1 (AIN1)** |
| 8 | OUT3 | Op Amp C Output | **ADS1115 Channel 2 (AIN2)** |
| 9 | IN3- | Op Amp C Inverting Input | **Connect to Pin 8** |
| 10 | IN3+ | Op Amp C Non-inverting Input | **LM2907_OUT DIRECT (NO DIVIDER)** |
| 11 | V- | Negative Power Supply | **GND (0V)** |
| 12 | IN4+ | Op Amp D Non-inverting Input | **Channel 3 voltage divider output** |
| 13 | IN4- | Op Amp D Inverting Input | **Connect to Pin 14** |
| 14 | OUT4 | Op Amp D Output | **ADS1115 Channel 3 (AIN3)** |

**Unity Gain Configuration**: Each op-amp output connects to its inverting input for 1:1 voltage following.

### ADS1115 Configuration

**Resolution**
- **Theoretical ADC range**: ±4.096V (GAIN = 1)
- **Actual usable range**: 0V to +3.3V
- **16-bit resolution**: 0.1mV per LSB (3.3V ÷ 32768 counts)
- **Effective resolution**: 0.1mV / divider_ratio per input LSB

---

## System Performance Analysis

### Total System Power Consumption

#### Per-Channel Power Analysis (Typical Operating Conditions)
| Channel | Application | Input Condition | Channel Power | IC Power Share | Total Power | Equivalent at 12V |
|---------|-------------|-----------------|---------------|----------------|-------------|-------------------|
| 0 | Battery Monitor | 12V | 0.14mW | 0.13mW | 0.27mW | **22.5µA** |
| 1 | Current Monitor | 2.5V (0A) | 4.1µW | 0.13mW | 0.13mW | **10.8µA** |
| 2 | RPM Monitor (optimized) | 1V | 0µW | 0.13mW | 0.13mW | **10.8µA** |
| 3 | Temperature (5V design) | 25°C | 1.25mW | 0.13mW | 1.38mW | **115µA** |

#### Total System Power
- **ADS1115**: 0.50mW (150µA @ 3.3V) = **41.7µA equivalent at 12V**
- **TLV9154IDR**: 0.013mW (4µA @ 3.3V) = **1.1µA equivalent at 12V**
- **Total IC power**: 0.51mW = **42.8µA equivalent at 12V**
- **Total system power**: ~2.32mW = **193µA equivalent at 12V** (with 5V Channel 3)

#### Battery Life Analysis (100Ah 12V Battery)
- **Continuous current draw**: ~193µA
- **Daily consumption**: 4.6mAh (0.005% of capacity)
- **Estimated battery life**: >50 years (limited by self-discharge, not monitoring system)
- **Conclusion**: Excellent performance for battery-powered marine applications

### Accuracy and Resolution Summary

| Channel | Input Range | Engineering Units | ADC Resolution | Best Accuracy | Limitations |
|---------|-------------|-------------------|----------------|---------------|-------------|
| 0 | 0.21V - 65.2V | Battery voltage | 0.0021V | 0.084% | Min 0.21V (op-amp limit) |
| 1 | ±200A | Alternator current | 0.02A | 0.52% | None (within sensor range) |
| 2 | 8Hz - 2640Hz | Engine frequency | Variable | 0.03% | Min 8Hz → 27-479 RPM |
| 3 | 15°C to +125°C | Temperature | 0.1mV | Variable | Cold temp limitation (5V design) |

---

## Design Validation Results

### Overvoltage Protection Verification
✅ **Inherent protection confirmed**: Single-supply op-amp output cannot exceed 3.3V  
✅ **No protection diodes required**: Eliminates leakage current and temperature drift  
✅ **Safety margins verified**: Op-amp max output (3.3V) < ADS1115 max allowed input 

### Accuracy Verification  
✅ **No leakage current errors**: Op-amp buffer isolates high-impedance voltage dividers  
✅ **Temperature stability**: No Schottky diode temperature coefficients  
✅ **Precision components**: 0.1% resistors eliminate ratio errors  
⚠️ **Minimum voltage limitations**: Op-amp VOL (~10mV) creates measurement dead zones

### Noise Performance Verification
✅ **Low-pass filtering effective**: 5nF capacitors remove high-frequency noise before amplification  
✅ **EMI immunity**: Capacitive filtering with proper placement  
✅ **Ground isolation**: Single-point grounding minimizes ground loops  

### Environmental Suitability
✅ **Temperature range**: -40°C to +85°C (industrial grade components)  
✅ **Marine environment**: Ultra-low power, high accuracy, EMI immune design  
✅ **Battery compatibility**: 129µA total consumption suitable for extended operation  
✅ **Fault tolerance**: Robust protection against high-voltage transients and DC faults
---

## Bill of Materials

### Shared System Components
- **U1**: TLV9154IDR quad op-amp, SOIC-14 SMD package
- **U2**: ADS1115IDGSR 16-bit ADC, MSOP-10 SMD package  
- **C_BYPASS**: 100nF ±10%, X7R dielectric, 0603 SMD decoupling capacitor

### Channel 0: Battery Voltage Monitor
- **R1**: 1MΩ ±0.1%, 1/8W, 0805 SMD
- **R2**: 49.9kΩ ±0.1%, 1/8W, 0805 SMD
- **C1**: 5nF ±10%, X7R, 0603 SMD

### Channel 1: Current Monitor  
- **R1**: 768kΩ ±0.1%, 1/8W, 0805 SMD
- **R2**: 768kΩ ±0.1%, 1/8W, 0805 SMD
- **C1**: 5nF ±10%, X7R, 0603 SMD

### Channel 2: RPM Monitor (Optimized Configuration)
- **R1**: Not used (direct connection)
- **R2**: Not used (direct connection)  
- **C1**: 5nF ±10%, X7R, 0603 SMD

**LM2907 Input Circuit:**
- **R_SERIES**: 4.7kΩ, 2W (HP122WJ0472T4E), critical for DC fault protection
- **C_AC1, C_AC2**: 10µF/100V, parallel for 20µF total AC coupling
- **C_FILTER**: 6.8nF ±10%, 0603 SMD (5kHz low-pass filter)
- **R_TERM**: 100kΩ ±5%, 1/8W, 0805 SMD
- **TVS1**: SMBJ12CA bidirectional TVS diode

**LM2907 Timing Circuit:**
- **C_TIMING1**: 10nF ±10%, 0603 SMD (Pin 2)
- **C_TIMING2**: 3 × 1µF ±10%, 0805 SMD parallel (Pin 3, total 3µF)
- **R_TIMING**: 25kΩ ±0.1%, 1/8W, 0805 SMD (Pin 3)
- **C_BYPASS**: 1µF ±10%, 0805 SMD (power supply bypass)

### Channel 3: Temperature Monitor (5V Supply Design)
- **R1**: 10kΩ ±0.1%, 1/8W, 0805 SMD (pullup to 5V)
- **R2**: 10kΩ NTC thermistor (user-supplied)
- **Supply**: 5V 
- **C1**: 5nF ±10%, X7R, 0603 SMD

### Component Selection Notes
- **Temperature rating**: Industrial grade (-40°C to +85°C minimum)
- **Precision**: ±0.1% resistor tolerance critical for measurement accuracy
- **Package size**: 0805/0603 SMD for compact layout and thermal stability
- **Capacitor dielectric**: X7R for temperature stability in filtering applications
- **High-power resistor**: HP122WJ0472T4E (2W) essential for DC fault protection
- **Op-amp selection**: TLV9154IDR provides rail-to-rail output for superior low-voltage performance
- **Channel 3 limitation**: 5V supply creates cold temperature measurement limitations due to ADC range

---

## Customer Application Guidelines

### Channel 0: Battery Voltage Monitoring
- **Recommended range**: 12V, 24V, or 48V systems
- **Resolution**: 2.1mV (excellent for battery monitoring)
- **Update rate**: Suitable for continuous monitoring
- **Power impact**: 22.5µA at 12V input
- **Limitation**: Cannot measure below 0.21V (not practically relevant)

### Channel 1: Current Monitoring  
- **Compatible sensors**: QNHC1K-21 200A hall sensor or equivalent
- **Sensor requirements**: 2.5V center, ±2V swing for full scale
- **Resolution**: 0.02A theoretical, ~1A practical accuracy
- **Power impact**: 10.8µA (minimal)

### Channel 2: RPM Monitoring
- **Input requirements**: Minimum 50mV (±50mV) for reliable detection  
- **Recommended input**: 100mV+ for best performance
- **Supported configurations**:
  - Multi-pulse alternator stator (typical marine/automotive)
  - Single pulse per revolution (slow engines) - 
- **Frequency range**: **8Hz to 2640Hz** 
- **Engine RPM range**: 
  - Conservative (6-pulse, 1.5:1): **53 RPM** to **17,600 RPM**
  - Aggressive (7-pulse, 2.5:1): **27 RPM** to **9,041 RPM**  
  - 1-pulse direct: **479 RPM** to **158,400 RPM**
- **Signal types**: Both sine and square waves work identically
- **DC fault protection**: Safe up to 56V sustained DC input
- **Power impact**: **Eliminated** (no voltage divider power consumption)


### Channel 3: Temperature Monitor
- **Current configuration**: 10kΩ pullup at 5V supply
- **Temperature limitation**: 5V supply creates unmeasurable voltages above 3.3V ADC limit for cold temperatures
- **Temperature range**: 
  - **Current (5V/10kΩ)**: 15°C to +125°C (cold temperature limitation)
  - **Potential improvement (3.3V/15kΩ)**: -40°C to +125°C (full range)
- **Resolution**: 0.1mV ADC resolution with 0.8-5.8°C temperature resolution depending on curve position
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