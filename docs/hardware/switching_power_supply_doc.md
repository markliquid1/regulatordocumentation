# Switching Power Supply

## Purpose
This dual-stage switching power supply converts wide-range input voltage to regulated 5V and 3.3V rails. The system prioritizes efficiency while providing clean, stable power for digital and analog circuits.

## Input Voltage Range
- **VIN_2-60**: 4.2V to 65V (recommended operating range)
- **Absolute maximum**: Up to 70V (stress rating)
- **Typical operating voltage**: 12V (automotive/industrial applications)

## Output Rails

### 5V Rail Specifications
| Parameter | Value |
|-----------|-------|
| Output Voltage | 5.0V ±2% |
| Maximum Current | 1A |
| Input Range | 4.2V - 65V |
| Peak Efficiency | 94% @ 300-500mA |
| Full Load Efficiency | 92% @ 1A |
| Power Dissipation @ 1A | 0.43W |

### 3.3V Rail Specifications  
| Parameter | Value |
|-----------|-------|
| Output Voltage | 3.3V ±2% |
| Maximum Current | 2A |
| Input Source | 5V rail |
| Peak Efficiency | 95.5% @ 500mA |
| Full Load Efficiency | 92% @ 2A |
| Power Dissipation @ 2A | 0.57W |

## Main Components

### 5V Stage (LMR36510ADDA)
| Component | Function |
|-----------|----------|
| LMR36510ADDA | Wide-input synchronous buck converter |
| L3 (SRP6540-220M) | 22µH switching inductor |
| R42/R48 | Feedback network (10kΩ/24.9kΩ) |
| C55 (100nF) | Bootstrap capacitor |
| C56-C58 | Output filtering (22µF × 3) |

### 3.3V Stage (TLV62569DBV)
| Component | Function |
|-----------|----------|
| TLV62569DBV | High-efficiency step-down converter |
| L1 (2.2µH) | Switching inductor |
| R25 (100kΩ) | Enable control resistor |
| C22/C23 | Input filtering |
| C24 (22µF) | Output filtering |

## Efficiency Performance

### 5V Rail (12V Input)
| Load Current | Output Power | Efficiency | Power Loss |
|:------------:|:------------:|:----------:|:----------:|
| 1 mA | 5 mW | 80% | 1 mW |
| 10 mA | 50 mW | 89% | 6 mW |
| 100 mA | 0.5 W | 92% | 43 mW |
| 300 mA | 1.5 W | **94%** | 96 mW |
| 500 mA | 2.5 W | **94%** | 0.16 W |
| 1 A | 5.0 W | 92% | 0.43 W |

### 3.3V Rail (5V Input)
| Load Current | Output Power | Efficiency | Power Loss |
|:------------:|:------------:|:----------:|:----------:|
| 1 mA | 3 mW | 90% | 0 mW |
| 10 mA | 33 mW | 94.5% | 2 mW |
| 100 mA | 0.33 W | 95% | 17 mW |
| 500 mA | 1.65 W | **95.5%** | 78 mW |
| 1 A | 3.30 W | 95% | 0.17 W |
| 2 A | 6.60 W | 92% | 0.57 W |

## Current Limitations

### Maximum Continuous Current
| Rail | Thermal Limit | Safe Continuous | Failure Point |
|------|---------------|-----------------|---------------|
| 5V | 1A (datasheet) | **1A** | >1A (overcurrent protection) |
| 3.3V | 2A (datasheet) | **2A** | >2A (overcurrent protection) |

### Practical Operating Limits
- **5V rail**: Limited to 1A by LMR36510ADDA current capability
- **3.3V rail**: Limited to 2A by TLV62569DBV current capability  
- **Combined power**: 11.6W maximum (5W + 6.6W)
- **Input current at 12V**: ~1A maximum at full load

## System Efficiency Analysis

### Overall Efficiency (12V → 3.3V)
When both converters operate in series:

| 3.3V Load | 5V Stage | 3.3V Stage | **Overall** |
|:---------:|:--------:|:----------:|:-----------:|
| 100 mA | 92% | 95% | **87.4%** |
| 500 mA | 94% | 95.5% | **89.8%** |
| 1 A | 92% | 95% | **87.4%** |

### Power Budget at Maximum Load
- **3.3V rail**: 6.6W @ 2A
- **5V rail power required**: 7.17W (including 3.3V stage losses)
- **12V input power**: 8.2W (including 5V stage losses)
- **Total system efficiency**: 80.5% at maximum 3.3V load

## Control and Enable Logic
- **3.3V Enable**: Controlled via 5V rail through R25 (100kΩ)
- **Enable threshold**: ~1.4V (typical for TLV62569)
- **Soft-start**: Integrated in both converters
- **Shutdown current**: <10µA when disabled

## Performance Goals
- **High efficiency**: >90% at typical operating loads
- **Wide input range**: Support 12V-48V automotive/industrial systems  
- **Clean regulation**: <50mV ripple on both rails
- **Fast transient response**: <50µs settling time
- **Low quiescent current**: Minimize idle power consumption

## Thermal Considerations
- **5V stage**: 0.43W dissipation at 1A requires adequate copper area
- **3.3V stage**: 0.57W dissipation at 2A may require thermal vias
- **PCB design**: 4-layer board recommended for thermal management
- **Component spacing**: Allow airflow around inductors and ICs

## Notes
- Both converters include integrated synchronous rectification for high efficiency
- Output capacitor ESR affects stability and ripple performance
- Input filtering critical for EMI compliance
- Enable sequencing: 5V must be stable before 3.3V rail activates

## Revision Info
- Design optimized for automotive 12V nominal input
- Component selection prioritizes efficiency over cost
- Thermal analysis assumes good PCB design practices