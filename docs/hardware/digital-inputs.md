# Digital Inputs

## Design Requirements
- **Input voltage range**: 5V to 54V on any of 6 digital input channels that use optical isolators
- **Temperature range**: -40°F to 180°F (-40°C to 82°C)
- **Lifetime target**: 10+ years at typical voltages (~12V), 3-6 years at maximum voltage (54V)

## How an Optocoupler Works

The VO615A optocoupler provides electrical isolation between input circuit and ESP32 GPIO using light transmission:

### Input Side (LED)
1. Input voltage applied across current limiting resistor and IR LED
2. Forward current flows through LED, producing infrared light
3. Light intensity proportional to forward current

### Output (ESP32 GPIO) Side (Phototransistor)
1. IR light strikes phototransistor base region
2. Light creates electron-hole pairs, acting as base current
3. Transistor turns on, allowing collector current to flow
4. Collector current = LED current × CTR (Current Transfer Ratio)
5. When sufficient collector current flows, voltage at collector drops low enough to register as logic LOW on ESP32
6. Voltage otherwise pulled high (3.3V) by pull up resistor to 3.3V Supply

### Isolation Benefits
- **Protection**: High voltage transients on input side cannot damage ESP32
- **Flexibility**: Can accept input signals from 5V to 60V (lower life at higher voltages)

## Component Selection

### Optocoupler: VO615A-9
- **CTR**: 200-400% minimum (vs 50-600% for base VO615A)
- **Reason**: Higher CTR enables reliable switching at low input voltages (5V), minimal extra cost

### Current Limiting Resistor: 6.8kΩ, 1W, 2512 SMD
- **Calculation basis**:
  - At 5V: IF = (5V - 1.43V) / 6.8kΩ = 0.53 mA
  - At 54V: IF = (54V - 1.43V) / 6.8kΩ = 7.7 mA
- **Power rating**: 404mW at 54V requires 1W resistor for safety margin

## Performance Analysis

| Input Voltage | LED Current | vs Min (1mA) | Lifetime | Power (Watts) | Power (mA@12V) |
|---------------|-------------|--------------|----------|---------------|----------------|
| 5V | 0.53 mA | 53% of min | 20+ years | 1.9 mW | 0.16 mA |
| 12V | 1.57 mA | 157% of min | 20+ years | 17 mW | 1.4 mA |
| 28V | 3.91 mA | 391% of min | 10+ years | 104 mW | 8.7 mA |
| 54V | 7.7 mA | 770% of min | 3-6 years | 404 mW | 33.7 mA |

**Power consumption notes**:
- Power is consumed **only when Input Signal is active** (ie external switch is ON)
- **No power consumption when Input Signal = 0V** (ie external switch OFF)
- LED power = Forward current × 1.43V (LED forward voltage)
- Resistor power = Forward current² × 6.8kΩ

## ESP32 Output Side Design

### Pull-up Resistor Selection
The ESP32 side requires a pull-up resistor from the optocoupler collector to 3.3V:

**Chosen value: 13kΩ**

**Analysis**:
- **Logic HIGH**: When LED is OFF, phototransistor is OFF, collector pulled to 3.3V through 13kΩ
- **Logic LOW**: When LED is ON, phototransistor conducts, pulling collector toward ground
- **Switching threshold**: ESP32 typically switches at ~1.65V (50% of 3.3V)

**Current requirements for reliable LOW**:
- Need collector current to create sufficient voltage drop: V = 3.3V - (Ic × 13kΩ)
- For logic LOW (< 1.65V): Ic > (3.3V - 1.65V) / 13kΩ = 127 µA minimum
- With VO615A-9's 200% minimum CTR: Ic = 0.53mA × 200% = 1.06mA (>> 127µA required)

**ESP32 side power consumption**:
- **When ignition OFF**: 3.3V / 13kΩ = 0.25mA = 0.83mW = 0.07mA@12V
- **When ignition ON**: Negligible (phototransistor saturated, ~0.3V across pull-up)

### Complete Circuit Analysis

**Total system power consumption**:

| Condition | LED Side | ESP32 Side | Total Power | Total (mA@12V) |
|-----------|----------|-------------|-------------|----------------|
| **Ignition OFF** | 0 mW | 0.83 mW | 0.83 mW | 0.07 mA |
| **Ignition ON (12V)** | 17 mW | ~0 mW | 17 mW | 1.4 mA |
| **Ignition ON (54V)** | 404 mW | ~0 mW | 404 mW | 33.7 mA |

## LED Current Guidelines
- **1-3mA**: Conservative operation (20+ year lifetime)
- **3-8mA**: Moderate operation (5-10 year lifetime)  
- **8-15mA**: Aggressive operation (1-5 year lifetime)

## Design Validation
- **5V switching**: 0.53mA with 200% CTR provides reliable switching despite being below 1mA datasheet specification
- **54V survival**: 7.7mA current provides marginal 3-6 year life if 100% duty cycle
- **Temperature performance**: VO615A-9's high CTR provides margin for temperature derating across -40°F to 180°F range
- **Component availability**: Both VO615A-9 and 6.8kΩ 1W resistors are standard, readily available parts

## Schematic Implementation
```
FORCEFLOAT ──[6.8kΩ, 1W]──[LED|──── GND
                               |
                               |  (Optical coupling)
                               |
ESP32_GPIO ──[13kΩ]──3.3V      |
        └─────────────[Collector|Emitter]──── GND
```

## Final Bill of Materials
- **U1**: VO615A-9 optocoupler
- **R1**: 6.8kΩ ±1%, 1W, 2512 SMD resistor  
- **R2**: 13kΩ ±5%, 1/8W, 0603 SMD pull-up resistor