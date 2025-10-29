# LM2907 Bias Resistor Change - Corrected Analysis

## Problem Statement
LM2907 experiences 5-10 second RPM dropout after alternator field cutoff due to AC coupling time constant mismatch. Scope shows only ~300ms actual zero-crossing loss, but coupling capacitor takes much longer to recover proper DC bias level.

## Root Cause
- Time constant τ = R × C = 100kΩ × 20µF = 2.0 seconds
- Recovery time ≈ 2.3τ for 90% recovery = 4.6 seconds (not 5τ = 10+ seconds)
- AC coupling cap retains charge from large signal, creates DC offset on small returning signal

## Current Circuit Configuration
| Component | Value | Power Rating | Performance |
|-----------|-------|--------------|-------------|
| Series Resistor | 4.7kΩ | 2W | Adequate |
| Bias Resistor | 100kΩ | 0.1W | Adequate |
| Coupling Capacitor | 20µF | N/A | N/A |
| Shunt Capacitor | 6.8nF | N/A | N/A |
| **Time Constant** | **2.0 seconds** | N/A | **Problem: 4.6 second dropout** |
| **High-Pass Corner** | **0.08 Hz** | N/A | Preserves low frequencies |

## Proposed Circuit Configurations  
| Component | Current | Option 1 | Option 2 |
|-----------|---------|----------|----------|
| Series Resistor | 4.7kΩ | 4.7kΩ | 4.7kΩ |
| Bias Resistor | 100kΩ | **1kΩ** | **10kΩ** |
| Coupling Capacitor | 20µF | 20µF | 20µF |
| Shunt Capacitor | 6.8nF | 6.8nF | 6.8nF |
| **Time Constant** | **2.0 s** | **0.02 s** | **0.2 s** |
| **High-Pass Corner** | **0.08 Hz** | **8 Hz** | **0.8 Hz** |
| **90% Recovery Time** | **4.6 s** | **0.046 s** | **0.46 s** |

## Power Consumption Analysis (Corrected)

### Complex Impedance Calculation

The circuit impedance must account for both capacitive reactances using proper complex voltage division:

**Circuit Model:**
```
Stator → 4.7kΩ series → 20µF coupling → node → (R_bias || 6.8nF) to ground
```

**Load Impedance Calculation:**
The load impedance is the parallel combination of the bias resistor and shunt capacitor:
```
Z_load = R_bias || Z_shunt = (R_bias × Z_shunt) / (R_bias + Z_shunt)
where Z_shunt = 1/(j2πfC_shunt)
```

**Total Impedance:**
```
Z_total = R_series + Z_coupling + Z_load
where Z_coupling = 1/(j2πfC_coupling)
```

**Node Voltage (Complex Division):**
```
V_node = V_in × (Z_load / Z_total)
```

**Power Calculations:**
- Series resistor: P_series = |I_line|² × R_series
- Bias resistor: P_bias = |V_node|² / R_bias

### TVS Conduction Analysis

**SMBJ12CA TVS breakdown voltage: ~13V**

**Node Peak Voltage Calculations:**

| Input Voltage | 1kΩ Node Peak | 10kΩ Node Peak | 100kΩ Node Peak | TVS Conduction |
|---------------|---------------|----------------|-----------------|----------------|
| **12V pp** | 1.05V | 4.08V | 5.73V | None |
| **24V pp** | 2.10V | 8.16V | 11.46V | None |
| **48V pp** | 4.21V | **16.32V** | **22.91V** | 10kΩ, 100kΩ conduct |
| **60V pp** | 5.26V | **20.40V** | **28.64V** | 10kΩ, 100kΩ conduct |

### Power Analysis at Key Operating Points

#### At 12V Peak-to-Peak (4.24V RMS, 60Hz) - No TVS Conduction

**Complex Impedance Calculations (Corrected):**
- X_c(6.8nF, 60Hz) = 1/(2π × 60 × 6.8nF) ≈ 389kΩ
- X_c(20µF, 60Hz) = 1/(2π × 60 × 20µF) ≈ 133Ω

**Current Design (100kΩ bias):**
- Z_parallel = 93.8k - j24.1kΩ (magnitude = 96.9kΩ)
- |Z_total| = 101.5kΩ (complex: 98.5k - j24.2kΩ)
- I_rms = 4.24V / 101.5kΩ = 41.8µA
- P_series = (41.8µA)² × 4.7kΩ = 0.008mW
- P_bias = **0.16mW**

**Option 1 (1kΩ bias):**
- Z_parallel ≈ 1kΩ (shunt cap negligible)
- |Z_total| ≈ 4.83kΩ
- I_rms = 4.24V / 4.83kΩ = 0.88mA
- P_series = (0.88mA)² × 4.7kΩ = 3.6mW
- P_bias = 0.6mW

**Option 2 (10kΩ bias):**
- Z_parallel = 9.78k - j1.46kΩ (magnitude = 9.89kΩ)
- |Z_total| ≈ 10.8kΩ
- I_rms = 4.24V / 10.8kΩ = 0.39mA
- P_series = (0.39mA)² × 4.7kΩ = 0.7mW
- P_bias = 0.8mW

#### At 24V Peak-to-Peak (8.49V RMS, 60Hz) - No TVS Conduction

**Current Design (100kΩ bias):**
- I_rms = 8.49V / 79.7kΩ = 0.106mA
- P_series = (0.106mA)² × 4.7kΩ = 0.053mW
- P_bias = (0.106mA × 79.6kΩ)² / 100kΩ = 0.71mW

**Option 1 (1kΩ bias):**
- I_rms = 8.49V / 4.93kΩ = 1.72mA
- P_series = (1.72mA)² × 4.7kΩ = 13.9mW
- P_bias = (1.72mA × 997Ω)² / 1kΩ = 2.95mW

**Option 2 (10kΩ bias):**
- I_rms = 8.49V / 10.88kΩ = 0.78mA
- P_series = (0.78mA)² × 4.7kΩ = 2.86mW
- P_bias = (0.78mA × 9.74kΩ)² / 10kΩ = 5.78mW

#### At 48V Peak-to-Peak (16.97V RMS, 60Hz) - TVS Conducts for 10kΩ and 100kΩ

**Option 1 (1kΩ bias) - No TVS Conduction:**
- I_rms = 16.97V / 4.93kΩ = 3.44mA
- P_series = (3.44mA)² × 4.7kΩ = 55.7mW
- P_bias = (3.44mA × 997Ω)² / 1kΩ = 11.8mW

**Option 2 (10kΩ bias) - TVS Conducts:**
- Node peak exceeds 13V (16.32V peak)
- TVS clamps during peaks, reducing bias resistor power
- Conservative estimate: ~15-20mW (reduced from linear prediction of 23.1mW)

**Current Design (100kΩ bias) - TVS Conducts:**
- Node peak exceeds 13V (22.91V peak)
- TVS clamps during peaks, reducing bias resistor power
- Conservative estimate: ~2-3mW (reduced from linear prediction of 2.87mW)

#### At 60V Peak-to-Peak (21.21V RMS, 60Hz) - TVS Conducts for 10kΩ and 100kΩ

**Option 1 (1kΩ bias) - No TVS Conduction:**
- I_rms = 21.21V / 4.93kΩ = 4.30mA
- P_series = (4.30mA)² × 4.7kΩ = 87.0mW
- P_bias = (4.30mA × 997Ω)² / 1kΩ = 18.5mW

**Option 2 (10kΩ bias) - TVS Conducts:**
- Node peak exceeds 13V (20.40V peak)
- TVS clamps during peaks, reducing bias resistor power
- Conservative estimate: ~20-25mW (reduced from linear prediction)

**Current Design (100kΩ bias) - TVS Conducts:**
- Node peak exceeds 13V (28.64V peak)
- TVS clamps during peaks, reducing bias resistor power
- Conservative estimate: ~3-4mW (reduced from linear prediction)

## Power Rating Requirements and Safety Analysis

### Complete Power Analysis with TVS Effects

| Input Voltage | Current (100kΩ) | Option 1 (1kΩ) | Option 2 (10kΩ) | Analysis Method |
|---------------|------------------|----------------|----------------|-----------------|
| **12V pp** | 0.18mW | 0.74mW | 1.45mW | Small-signal (no TVS) |
| **24V pp** | 0.71mW | 2.95mW | 5.78mW | Small-signal (no TVS) |
| **48V pp** | ~2-3mW | 11.8mW | ~15-20mW | TVS clamped / Small-signal |
| **60V pp** | ~3-4mW | 18.5mW | ~20-25mW | TVS clamped / Small-signal |

### Safety Factor Analysis

**Safety Factor with 0.1W Rating:**
| Input Voltage | Current (100kΩ) | Option 1 (1kΩ) | Option 2 (10kΩ) |
|---------------|------------------|----------------|----------------|
| **12V pp** | 556× (excellent) | 135× (excellent) | 69× (excellent) |
| **24V pp** | 141× (excellent) | 34× (excellent) | 17× (excellent) |
| **48V pp** | 40× (excellent) | 8.5× (good) | 5.5× (adequate) |
| **60V pp** | 30× (excellent) | 5.4× (adequate) | 4.5× (adequate) |

**Engineering Practice:** Minimum 3× safety factor recommended for reliable operation.

### Component Rating Recommendations

- **Current design (100kΩ):** 0.1W adequate for all operating conditions (TVS clamping reduces power at high voltages)
- **Option 1 (1kΩ):** 0.1W adequate for all operating conditions (5.4× safety factor at 60V, no TVS conduction)
- **Option 2 (10kΩ):** **0.1W adequate for all operating conditions** (4.5× safety factor at 60V with TVS clamping)

### Key Findings

- **TVS clamping begins at 48V pp** for 10kΩ and 100kΩ configurations
- **1kΩ configuration avoids TVS conduction** up to 60V pp, maintaining linear operation
- **All configurations meet minimum 3× safety factor** with 0.1W rating across full operating range
- **TVS clamping reduces actual power** below small-signal predictions for affected configurations

### Engine Off Power Consumption
- Stator at ~12V DC (through rectifier diodes)
- **Coupling capacitor blocks all DC current**
- Only capacitor leakage: ~2-3µA
- Power consumption: **Negligible (~microwatts)** for all options

## Transfer Function Analysis

The complete transfer function includes both the high-pass filter formed by the coupling capacitor and the voltage divider effect:

```
H(jω) = [R_bias / (R_bias + 4.7kΩ)] × [jωτ / (1 + jωτ)]
where τ = R_bias × 20µF (coupling time constant)
```

## Frequency Response Calculations

### Reactance Values
| Frequency | X_c(20µF) | X_c(6.8nF) | Coupling Effect | Shunt Effect |
|-----------|-----------|------------|-----------------|--------------|
| 5 Hz | 1592Ω | 4.68MΩ | Significant | Negligible |
| 10 Hz | 796Ω | 2.34MΩ | Moderate | Negligible |
| 50 Hz | 159Ω | 468kΩ | Minor | Minor |
| 100 Hz | 80Ω | 234kΩ | Minimal | Minor |
| 1000 Hz | 8Ω | 23.4kΩ | Negligible | Moderate |
| 2000 Hz | 4Ω | 11.7kΩ | Negligible | Significant |

### Required Stator Amplitude Analysis

**Required stator amplitude to produce 25mVpp at LM2907 pin:**

| Frequency | 100kΩ Bias | 10kΩ Bias | 1kΩ Bias | Notes |
|-----------|------------|-----------|----------|-------|
| **5 Hz** | 36.8mVpp | 52.8mVpp | 694mVpp | High-pass attenuation dominates |
| **10 Hz** | 29.5mVpp | 42.1mVpp | 278mVpp | Moderate high-pass attenuation |
| **50 Hz** | 26.8mVpp | 38.1mVpp | 154mVpp | Minor high-pass effect |
| **100 Hz** | 26.4mVpp | 37.4mVpp | 145mVpp | Minimal high-pass effect |
| **1000 Hz** | 26.2mVpp | 36.8mVpp | 142mVpp | High-frequency baseline |
| **2000 Hz** | 26.2mVpp | 36.7mVpp | 141mVpp | Shunt capacitor starts affecting 1kΩ |

### Detailed Calculations

#### 100kΩ Bias Resistor
| Frequency | Coupling Gain | Divider Gain | Total Gain | Required Input |
|-----------|---------------|--------------|------------|----------------|
| 5 Hz | 0.628 | 0.955 | 0.600 | 41.7mVpp |
| 10 Hz | 0.783 | 0.955 | 0.748 | 33.4mVpp |
| 50 Hz | 0.950 | 0.955 | 0.907 | 27.6mVpp |
| 100 Hz | 0.975 | 0.955 | 0.931 | 26.8mVpp |
| 1000 Hz | 0.998 | 0.955 | 0.953 | 26.2mVpp |
| 2000 Hz | 0.999 | 0.955 | 0.954 | 26.2mVpp |

#### 10kΩ Bias Resistor
| Frequency | Coupling Gain | Divider Gain | Total Gain | Required Input |
|-----------|---------------|--------------|------------|----------------|
| 5 Hz | 0.628 | 0.680 | 0.427 | 58.5mVpp |
| 10 Hz | 0.783 | 0.680 | 0.532 | 47.0mVpp |
| 50 Hz | 0.950 | 0.680 | 0.646 | 38.7mVpp |
| 100 Hz | 0.975 | 0.680 | 0.663 | 37.7mVpp |
| 1000 Hz | 0.998 | 0.680 | 0.679 | 36.8mVpp |
| 2000 Hz | 0.999 | 0.680 | 0.679 | 36.8mVpp |

#### 1kΩ Bias Resistor
| Frequency | Coupling Gain | Divider Gain | Total Gain | Required Input |
|-----------|---------------|--------------|------------|----------------|
| 5 Hz | 0.628 | 0.176 | 0.111 | 225mVpp |
| 10 Hz | 0.783 | 0.176 | 0.138 | 181mVpp |
| 50 Hz | 0.950 | 0.176 | 0.167 | 150mVpp |
| 100 Hz | 0.975 | 0.176 | 0.172 | 145mVpp |
| 1000 Hz | 0.998 | 0.176 | 0.176 | 142mVpp |
| 2000 Hz | 0.999 | 0.175 | 0.175 | 143mVpp |

## Key Frequency Dependencies

### High-Pass Corner Frequencies
- **100kΩ**: f_c = 0.08 Hz (minimal impact above 5 Hz)
- **10kΩ**: f_c = 0.8 Hz (minor impact at 5-10 Hz)
- **1kΩ**: f_c = 8.0 Hz (significant impact below 50 Hz)

### Signal Strength Requirements vs Frequency

**Relative to 100kΩ baseline at 1000 Hz (26.2mVpp):**

| Frequency | 100kΩ Factor | 10kΩ Factor | 1kΩ Factor |
|-----------|--------------|-------------|------------|
| 5 Hz | 1.40× | 2.01× | 8.59× |
| 10 Hz | 1.13× | 1.61× | 6.91× |
| 50 Hz | 1.02× | 1.45× | 5.73× |
| 100 Hz | 1.00× | 1.43× | 5.54× |
| 1000 Hz | 1.00× | 1.40× | 5.42× |
| 2000 Hz | 1.00× | 1.40× | 5.42× |

## Engineering Implications

### Low Frequency Performance (5-10 Hz)
- **100kΩ**: Excellent performance, <40% penalty at 5 Hz
- **10kΩ**: Good performance, <60% penalty at 5 Hz  
- **1kΩ**: Poor performance, 8-9× penalty at 5 Hz

### Mid Frequency Performance (50-100 Hz)
- **100kΩ**: Baseline reference performance
- **10kΩ**: 40-45% signal strength penalty
- **1kΩ**: 5.5× signal strength penalty

### High Frequency Performance (1-2 kHz)
- **100kΩ**: Baseline reference (26.2mVpp)
- **10kΩ**: Consistent 40% penalty (36.8mVpp)
- **1kΩ**: Consistent 5.4× penalty (142mVpp)

## Recovery Time Analysis (Corrected)

### Exponential Recovery Formula

When the coupling capacitor has stored charge, recovery follows:
```
V(t) = V_initial × e^(-t/τ)
```

For practical recovery to LM2907 threshold (~25mVpp):
```
t_recovery = -τ × ln(1 - α)
where α = required recovery fraction
```

**Recovery Definition:** Time until LM2907 pin voltage swing exceeds ~25mVpp detection threshold (not complete return to baseline).

| Recovery Target | Time Factor | 100kΩ (τ=2.0s) | 10kΩ (τ=0.2s) | 1kΩ (τ=0.02s) |
|----------------|-------------|-----------------|---------------|---------------|
| 90% recovery | 2.3τ | 4.6 seconds | **0.46 seconds** | 0.046 seconds |
| 95% recovery | 3.0τ | 6.0 seconds | **0.60 seconds** | 0.060 seconds |
| 99% recovery | 4.6τ | 9.2 seconds | **0.92 seconds** | 0.092 seconds |

**Practical LM2907 Recovery:**
Based on ~25mVpp minimum input threshold for reliable frequency detection, 90% recovery provides adequate signal restoration for normal operation.

## DC Drift Analysis

### Mechanism 1: Field Collapse (Primary Issue)
When alternator field cuts off, the coupling capacitor retains charge creating a DC offset that decays with time constant τ = R_bias × C_coupling. This is the **primary cause** of the 5-10 second dropout.

### Mechanism 2: TVS Clamping Effects (Secondary)
TVS clamping effects occur when node voltage exceeds ~13V:

**TVS Conduction Thresholds:**
- **12V-24V pp operation**: No TVS conduction for any configuration
- **48V pp operation**: TVS conducts for 10kΩ and 100kΩ (node peaks: 16.3V, 22.9V)
- **60V pp operation**: TVS conducts for 10kΩ and 100kΩ (node peaks: 20.4V, 28.6V)
- **1kΩ configuration**: No TVS conduction up to 60V pp (max node peak: 5.3V)

**During field collapse**: Signal levels return to small values (<24V pp), so TVS effects become negligible for the dropout recovery problem.

## Technical Tradeoffs Summary

| Parameter | Current (100kΩ) | Option 1 (1kΩ) | Option 2 (10kΩ) | Assessment |
|-----------|------------------|----------------|-----------------|------------|
| Recovery Time | 4.6 seconds | 0.046 seconds | **0.46 seconds** | All options solve dropout |
| High-Pass Corner | 0.08 Hz | 8 Hz | **0.8 Hz** | 1kΩ attenuates low RPM |
| Signal Sensitivity | Reference | 5.4× worse | **40% penalty** | 1kΩ needs stronger signals |
| Power @ 48V pp | ~2-3mW | 11.8mW | **~15-20mW** | All acceptable with 0.1W |
| TVS Immunity @ 48V | Clamps | **No clamp** | Clamps | 1kΩ avoids nonlinearity |
| TVS Immunity @ 60V | Clamps | **No clamp** | Clamps | 1kΩ avoids nonlinearity |
| Component Rating | 0.1W adequate | **0.1W adequate** | **0.1W adequate** | All meet 3× safety minimum |
| Implementation | N/A | 1 resistor | **1 resistor** | Both simple changes |

## Recommendation

**Option 2 (10kΩ bias resistor)** provides the optimal balance:

**Advantages:**
- **Fast recovery:** 0.46 seconds (90% recovery) eliminates dropout problem
- **Acceptable sensitivity:** 40% signal strength penalty vs current design
- **Minimal frequency impact:** 0.8Hz corner barely affects engine frequencies (>5Hz)
- **Simple implementation:** Single component change
- **Adequate power rating:** 0.1W provides 4.5× safety margin at 60V pp with TVS clamping
- **Better EMI immunity:** Lower impedance node vs 100kΩ

**Limitations:**
- **TVS conduction at 48V+:** Node voltage exceeds 13V at 48V pp and above
- **Signal strength requirement:** Verify stator provides >37mVpp for reliable operation

**Requirements:**
- **Component change:** Replace 100kΩ bias resistor with 10kΩ, 0.1W rated
- **Voltage consideration:** TVS clamping occurs at 48V pp and above
- **Recovery verification:** Confirm 0.5-second recovery meets system requirements

**Alternative:**
If sub-0.1 second recovery is required and linear operation up to 60V pp is needed, Option 1 (1kΩ) provides 0.046-second recovery with no TVS conduction, but with 5.4× sensitivity penalty and potential low-RPM detection issues due to 8Hz high-pass corner.

## Implementation
**Replace 100kΩ bias resistor with 10kΩ, 0.1W rated component**

**Benefits:**
- Eliminates 5-10 second LM2907 dropout
- Maintains reasonable signal sensitivity 
- Preserves low-frequency response
- Adequate power rating with 0.1W resistor
- Improved EMI immunity

**Verification Steps:**
1. Confirm 0.5-second recovery meets system response requirements
2. Test signal detection capability with 40% sensitivity reduction
3. For >48V pp operation, verify TVS clamping behavior
4. Validate power dissipation under actual stator voltage conditions