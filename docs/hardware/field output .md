
# Alternator Field Drive 

## Purpose
This circuit provides a high-side PWM drive to the gate of the N-channel MOSFET controlling the alternator field coil. The goal is to achieve efficient, reliable operation up to 100kHz PWM while minimizing quiescent current when the circuit is inactive.

## Source Signal
- **Origin**: ESP32 PWM-capable GPIO
- **Voltage**: 3.3V logic level

## Main Components 
| Component     | Function                                         |
|---------------|--------------------------------------------------|
| LM5109A       | High-speed high-side gate driver (10-12V VDD)    |
| MT3608        | Boost converter from 5V to 12V for gate driver   |
| BSC072N08NS5  | N-channel MOSFET to switch field coil            |
| 74LVC1G04     | Inverter to enable/disable the 5V rail via logic |
| BSS84         | P-MOSFET to gate 74LVC1G04 enable logic          |
| Bootstrap Diode (BAS16LT1G) | Charges bootstrap cap during low-side on |
| Bootstrap Cap (100nF–1uF, 100V) | Supplies transient gate drive voltage |
| Gate Resistor (~10Ω) | Limits gate inrush and dampens ringing     |

## Operating Voltages
- **Logic Source**: 3.3V from ESP32
- **Driver Supply (VDD)**: 10–12V from MT3608
- **PWM Frequency Target**: Up to 100kHz

## Power Management
- 5V supply is software-controlled via Field Enable + ON/OFF + Alert! logic
- If 5V is off, MT3608 also ceases operation (since input is cut), minimizing idle current
- No TVS or gate clamp added (assumes clean layout and short traces)

## Performance Goals
- Reliable switching with >10V Vgs
- Minimal idle current draw when disabled
- Efficient field coil control with minimal MOSFET dissipation

## Notes
- Bootstrap capacitor sizing must support fast switching without voltage sag

## MOSFET Thermal and Current Limit Analysis

### Key Parameters

- **Gate drive voltage (V<sub>GS</sub>)**: 12 V
- **Estimated R<sub>DS(on)</sub> at 12 V**: ≈ 5.5 mΩ (conservative extrapolation from datasheet graph)
- **PCB configuration**: 4-layer board with good copper and via stitching
- **No external heatsink**
- **Max junction temperature (T<sub>J,max</sub>)**: 100 °C
- **Ambient temperature (T<sub>A</sub>)**: 25 °C
- **Estimated thermal resistance (R<sub>θJA</sub>)**: ~25 °C/W (based on good PCB design for SON-8 package)

---

### Max Power Dissipation

\[
\Delta T = T_{J,max} - T_A = 100 - 25 = 75^\circ C
\]

\[
P_{max} = \frac{75}{25} = 3\,W
\]

---

### Max Continuous Current

\[
P = I^2 \cdot R_{DS(on)} \Rightarrow
I_{max} = \sqrt{\frac{3}{0.0055}} \approx 23.3\,A
\]

---

### Practical Limits

| Limit Type              | Estimate             |
|-------------------------|----------------------|
| Power-limited current   | ~23 A                |
| Safe continuous current | **15–20 A** (margin) |
| Likely failure point    | > 23 A (thermal)     |

**Note:** Above 20 A, trace heating, solder joint fatigue, and junction rise make failure more likely without forced cooling or heatsinking.  Ambient temperature assumption of 25C is not worst case scenario.


## Revision Info
- Original driver (MAX15054) replaced due to low Vgs drive at 5V rail
- New system uses external 12V boost to properly saturate the MOSFET gate
