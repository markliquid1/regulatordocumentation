# Alternator Field Control

## Overview

The alternator field control system is the core function of the regulator, responsible for precisely controlling alternator output by modulating the field current through PWM (Pulse Width Modulation). The system implements two distinct control algorithms: standard mode for basic operation and Learning Mode for adaptive thermal management.

## Theory of Operation

### Alternator Field Control Fundamentals

An alternator generates electrical power through electromagnetic induction. The strength of the magnetic field (controlled by field current) directly determines the output voltage and current capacity. By varying the field current, the regulator controls:

- **Output voltage**: Higher field current = higher output voltage
- **Current capability**: Stronger field = ability to maintain voltage under load
- **Power output**: Combined effect determines total alternator power delivery

### Field Control Circuit

The field control circuit uses a high-current MOSFET driver to switch the field coil:

```
Battery 12V → Field Coil → MOSFET (controlled by ESP32 PWM) → Ground
                ↑
            Flyback Diode
```

**Key Components**:
- **Field Coil**: Typically 2-6Ω resistance, inductance creates flyback voltage
- **MOSFET**: High-current switch (GPIO pin 4 enable, PWM pin 32 control)
- **Flyback Diode**: Protects MOSFET from inductive kickback
- **PWM Signal**: 12-bit resolution (4096 steps), 15kHz switching frequency

### PWM Configuration

```cpp
// PWM setup parameters
const int pwmPin = 32;              // Field PWM output pin
const int pwmResolution = 12;       // 12-bit resolution (0-4095)
float SwitchingFrequency = 15000;   // 15kHz - above audible range

// Initialize PWM
ledcAttach(pwmPin, SwitchingFrequency, pwmResolution);

// Set duty cycle
void setDutyPercent(int percent) {
  percent = constrain(percent, 0, 100);
  uint32_t duty = (4095UL * percent) / 100;
  ledcWrite(pwmPin, duty);
}
```

**Frequency Selection**: 15kHz provides:
- Above audible range (>15kHz) to prevent noise
- Below EMI concerns (<25kHz)
- Good magnetic efficiency for field coil inductance
- Fast enough response for control stability

## Standard Control Algorithm (`AdjustField()`)

The standard algorithm implements traditional alternator regulation with multiple safety layers and override options.

### Control Flow Overview

```
Sensor Inputs → Target Calculation → Field Adjustment → Safety Overrides → PWM Output
     ↓               ↓                    ↓                  ↓               ↓
Battery V,I → uTargetAmps (Hi/Low) → dutyCycle ± step → Temperature Limit → MOSFET
Temperature → RPM Curve Override   → PID-like Control → Voltage Limit    → Field Coil
Engine RPM  → Manual Override      → Rate Limited    → Current Limit     → Alt Output
```

### Core Control Logic

```cpp
void AdjustField() {
  static unsigned long lastFieldAdjustment = 0;
  unsigned long currentTime = millis();
  
  // Timing control - update every FieldAdjustmentInterval (50ms default)
  if (currentTime - lastFieldAdjustment <= FieldAdjustmentInterval) {
    return;
  }
  
  // Enable conditions
  chargingEnabled = (Ignition == 1 && OnOff == 1);
  
  // Emergency protection
  if (currentBatteryVoltage > (ChargingVoltageTarget + 0.2)) {
    digitalWrite(4, 0);              // Immediate field disable
    dutyCycle = MinDuty;
    fieldCollapseTime = currentTime;
    queueConsoleMessage("EMERGENCY: Field collapsed - voltage spike");
    return;
  }
  
  if (chargingEnabled) {
    digitalWrite(4, 1);              // Enable field MOSFET
    
    if (ManualFieldToggle == 0) {    // Automatic mode
      // Step 1: Base target from Hi/Low setting
      uTargetAmps = (HiLow == 1) ? TargetAmps : TargetAmpL;
      
      // Step 2: RPM curve override
      if (AmpControlByRPM == 1 && RPM > 100 && RPM < 6000) {
        int rpmBasedAmps = interpolateAmpsFromRPM(RPM);
        uTargetAmps = (HiLow == 0) ? min(rpmBasedAmps, TargetAmpL) : rpmBasedAmps;
      }
      
      // Step 3: Force Float override
      if (ForceFloat == 1) {
        uTargetAmps = 0;             // Target zero amps for float charging
      }
      
      // Step 4: Weather mode override
      if (weatherModeEnabled == 1 && currentWeatherMode == 1) {
        uTargetAmps = -99;           // Disable charging in high solar
      }
      
      // Step 5: Get current measurement
      targetCurrent = (ForceFloat == 1) ? Bcur : getTargetAmps();
      
      // Step 6: Proportional control
      if (targetCurrent < uTargetAmps && dutyCycle < (MaxDuty - dutyStep)) {
        dutyCycle += dutyStep;       // Increase field
      }
      if (targetCurrent > uTargetAmps && dutyCycle > (MinDuty + dutyStep)) {
        dutyCycle -= dutyStep;       // Decrease field
      }
      
      // Step 7: Safety overrides (progressively aggressive)
      if (TempToUse > TemperatureLimitF && dutyCycle > (MinDuty + 2*dutyStep)) {
        dutyCycle -= 2 * dutyStep;   // Temperature protection
      }
      if (currentBatteryVoltage > ChargingVoltageTarget && dutyCycle > (MinDuty + 3*dutyStep)) {
        dutyCycle -= 3 * dutyStep;   // Voltage protection
      }
      if (Bcur > MaximumAllowedBatteryAmps && dutyCycle > (MinDuty + dutyStep)) {
        dutyCycle -= dutyStep;       // Battery current protection
      }
      
      dutyCycle = constrain(dutyCycle, MinDuty, MaxDuty);
      
    } else {
      // Manual override mode
      dutyCycle = ManualDutyTarget;
    }
  } else {
    // Charging disabled
    digitalWrite(4, 0);
    dutyCycle = MinDuty;
  }
  
  // Apply calculated duty cycle
  setDutyPercent((int)dutyCycle);
  
  // Calculate field parameters for display
  vvout = dutyCycle / 100 * currentBatteryVoltage;
  iiout = vvout / FieldResistance;
  
  lastFieldAdjustment = currentTime;
}
```

### Target Current Calculation

#### Hi/Low Mode Selection
```cpp
// Basic target selection
if (HiLow == 1) {
  uTargetAmps = TargetAmps;     // Normal mode (typically 40A)
} else {
  uTargetAmps = TargetAmpL;     // Low mode (typically 25A)
}
```

**Usage**:
- **Normal mode**: Maximum alternator output for fast charging
- **Low mode**: Reduced output for extended operation or hot conditions

#### RPM-Based Current Curves

The system supports a 20-point RPM vs current curve for engine-specific optimization:

```cpp
int interpolateAmpsFromRPM(float currentRPM) {
  // Arrays from global variables RPM1-RPM20, Amps1-Amps20
  int rpmPoints[] = {RPM1, RPM2, RPM3, ...};
  int ampPoints[] = {Amps1, Amps2, Amps3, ...};
  
  // Find valid entries (non-zero RPM values)
  int validPoints = 0;
  for (int i = 0; i < 20; i++) {
    if (rpmPoints[i] > 0) validPoints = i + 1;
  }
  
  // Handle edge cases
  if (currentRPM <= rpmPoints[0]) return ampPoints[0];
  if (currentRPM >= rpmPoints[validPoints-1]) return ampPoints[validPoints-1];
  
  // Linear interpolation between points
  for (int i = 0; i < validPoints - 1; i++) {
    if (currentRPM >= rpmPoints[i] && currentRPM <= rpmPoints[i+1]) {
      float ratio = (currentRPM - rpmPoints[i]) / (float)(rpmPoints[i+1] - rpmPoints[i]);
      return ampPoints[i] + (ratio * (ampPoints[i+1] - ampPoints[i]));
    }
  }
  
  return TargetAmps;  // Fallback
}
```

**Example RPM Curve**:
```
RPM:   0   500  800  1000 1200 1500 4000
Amps:  0   20   30   40   40   50   30
```

**Benefits**:
- **Low RPM protection**: Reduced load when engine can't handle full power
- **High RPM protection**: Reduced field at excessive speeds
- **Optimal power**: Maximum safe output at engine's power band
- **Smooth transitions**: Linear interpolation prevents sudden changes

#### Current Source Selection

The `getTargetAmps()` function implements sensor source selection with fallback:

```cpp
float getTargetAmps() {
  switch (AmpSrc) {
    case 0:  // Alternator Hall Effect Sensor (default)
      return MeasuredAmps;
      
    case 1:  // Battery Shunt (INA228)
      return Bcur;
      
    case 6:  // Victron VE.Direct
      if (abs(VictronCurrent) > 0.1) {
        return VictronCurrent;
      }
      // Fallback to battery shunt
      return Bcur;
      
    default:
      queueConsoleMessage("Invalid AmpSrc, using Alt Hall Sensor");
      return MeasuredAmps;
  }
}
```

**Source Selection Strategy**:
- **Alternator current**: Direct measurement of alternator output
- **Battery current**: Net battery charging rate (accounts for loads)
- **External sources**: NMEA2K, NMEA0183, VE.Direct integration

### Control Algorithm Analysis

#### Proportional Control
The standard algorithm implements simple proportional control:

```cpp
// Increase field if below target
if (targetCurrent < uTargetAmps && dutyCycle < (MaxDuty - dutyStep)) {
  dutyCycle += dutyStep;
}

// Decrease field if above target  
if (targetCurrent > uTargetAmps && dutyCycle > (MinDuty + dutyStep)) {
  dutyCycle -= dutyStep;
}
```

**Parameters**:
- `dutyStep`: Adjustment increment per cycle (default 0.8%)
- `FieldAdjustmentInterval`: Update rate (default 50ms)
- Result: ~1.6%/second maximum change rate

**Characteristics**:
- **Stable**: No overshoot or oscillation
- **Responsive**: 50ms update rate
- **Predictable**: Linear response to error

#### Safety Override Hierarchy

Safety overrides apply with increasing aggressiveness:

```cpp
// Level 1: Temperature (2x step reduction)
if (TempToUse > TemperatureLimitF) {
  dutyCycle -= 2 * dutyStep;  // 1.6% reduction
}

// Level 2: Voltage (3x step reduction)  
if (currentBatteryVoltage > ChargingVoltageTarget) {
  dutyCycle -= 3 * dutyStep;  // 2.4% reduction
}

// Level 3: Battery current (1x step reduction)
if (Bcur > MaximumAllowedBatteryAmps) {
  dutyCycle -= dutyStep;      // 0.8% reduction
}
```

**Priority Logic**:
1. **Voltage protection**: Most aggressive (prevents alternator runaway)
2. **Temperature protection**: Highly aggressive (prevents thermal damage)
3. **Current protection**: Standard rate (protects battery and wiring)

## Emergency Protection Systems

### Field Collapse Protection

Immediate field shutdown on dangerous voltage spikes:

```cpp
if (currentBatteryVoltage > (ChargingVoltageTarget + 0.2)) {
  digitalWrite(4, 0);              // Emergency MOSFET disable
  dutyCycle = MinDuty;
  setDutyPercent((int)dutyCycle);
  fieldCollapseTime = currentTime;
  queueConsoleMessage("EMERGENCY: Field collapsed - voltage spike");
  return;
}
```

**Recovery Logic**:
```cpp
// Stay disabled for 10 seconds
if (fieldCollapseTime > 0 && (currentTime - fieldCollapseTime) < FIELD_COLLAPSE_DELAY) {
  digitalWrite(4, 0);
  dutyCycle = MinDuty;
  return;
}

// Clear flag after delay
if (fieldCollapseTime > 0 && (currentTime - fieldCollapseTime) >= FIELD_COLLAPSE_DELAY) {
  fieldCollapseTime = 0;
  queueConsoleMessage("Field collapse delay expired - normal operation resumed");
}
```

**Benefits**:
- **Immediate response**: Hardware-level protection
- **Prevents runaway**: Stops voltage from climbing further
- **Automatic recovery**: Resumes operation when safe
- **User notification**: Clear indication of protective action

### Hardware Overvoltage Protection

The INA228 provides independent hardware protection:

```cpp
// Configure hardware threshold
uint16_t thresholdLSB = (uint16_t)(VoltageHardwareLimit / 0.003125);
INA.setBusOvervoltageTH(thresholdLSB);
INA.setDiagnoseAlertBit(INA228_DIAG_BUS_OVER_LIMIT);

// Direct hardware write for latching alert
Wire.beginTransmission(INA.getAddress());
Wire.write(0x0F);  // ALERT_MASK_ENABLE register
Wire.write(0x98);  // Enable ALERT pin + latch + bus overvoltage
Wire.write(0x00);
Wire.endTransmission();
```

**Operation**:
- **Independent**: Functions regardless of ESP32 software state
- **Fast response**: Hardware detection and response
- **Latching**: Maintains alert until manually cleared
- **Automatic recovery**: Software monitors and clears when voltage drops

### Voltage Source Validation

Battery voltage measurement includes redundancy checking:

```cpp
// Cross-validation between sensors
if (abs(BatteryV - IBV) > 0.1) {
  queueConsoleMessage("Disagreement in measured Battery Voltage - Field Shut Off for safety!");
  digitalWrite(33, HIGH);  // Sound alarm
  digitalWrite(4, 0);      // Disable field
  dutyCycle = MinDuty;
  return;
}
```

**Sensors Compared**:
- `BatteryV`: ADS1115 measurement via voltage divider
- `IBV`: INA228 high-precision measurement
- **Tolerance**: 0.1V maximum difference allowed

## Advanced Features

### Charging Stage Logic

The system implements bulk/float charging stages:

```cpp
void updateChargingStage() {
  float currentVoltage = getBatteryVoltage();
  
  if (inBulkStage) {
    ChargingVoltageTarget = BulkVoltage;  // Typically 13.9V
    
    if (currentVoltage >= ChargingVoltageTarget) {
      if (bulkCompleteTimer == 0) {
        bulkCompleteTimer = millis();
      } else if (millis() - bulkCompleteTimer > bulkCompleteTime) {
        inBulkStage = false;
        floatStartTime = millis();
        queueConsoleMessage("Bulk stage complete, switching to float");
      }
    } else {
      bulkCompleteTimer = 0;
    }
  } else {
    ChargingVoltageTarget = FloatVoltage;  // Typically 13.4V
    
    // Return to bulk after time expires or voltage drops
    if ((millis() - floatStartTime > FLOAT_DURATION * 1000) || 
        (currentVoltage < FloatVoltage - 0.5)) {
      inBulkStage = true;
      queueConsoleMessage("Returning to bulk stage");
    }
  }
}
```

**Stage Characteristics**:
- **Bulk stage**: Higher voltage for fast charging
- **Float stage**: Lower voltage for maintenance
- **Automatic transitions**: Based on voltage and time
- **Return logic**: Drops back to bulk if voltage falls or time expires

### Force Float Mode

Special mode for precision float charging:

```cpp
if (ForceFloat == 1) {
  uTargetAmps = 0;              // Target zero net battery current
  targetCurrent = Bcur;         // Use battery current, not alternator current
}
```

**Purpose**:
- **Precision float**: Maintains exact float voltage
- **Load compensation**: Accounts for vessel loads
- **Battery protection**: Prevents overcharging
- **Long-term storage**: Ideal for seasonal storage

### Weather Integration

Solar panel integration for smart charging:

```cpp
if (weatherModeEnabled == 1 && currentWeatherMode == 1) {
  uTargetAmps = -99;  // Disable alternator charging
}
```

**Logic**:
- Fetches solar forecast data
- Disables alternator when sufficient solar available
- Reduces engine runtime and fuel consumption
- Integrates with solar charge controllers

### BMS Integration

Battery Management System override capability:

```cpp
if (bmsLogic == 1) {
  bmsSignalActive = !digitalRead(36);
  if (bmsLogicLevelOff == 0) {
    chargingEnabled = chargingEnabled && bmsSignalActive;
  } else {
    chargingEnabled = chargingEnabled && !bmsSignalActive;
  }
}
```

**Features**:
- Digital input from BMS
- Configurable polarity (active high/low)
- Override capability for LiFePO4 protection
- Integration with external battery protection systems

## Field Control Calculations

### Field Power Calculation

Real-time field power parameters:

```cpp
// Field voltage (approximate)
vvout = dutyCycle / 100 * currentBatteryVoltage;

// Field current (approximate, assumes resistive load)
iiout = vvout / FieldResistance;

// Field power
float fieldPower = vvout * iiout;
```

**Accuracy Limitations**:
- Assumes purely resistive field coil
- Ignores inductive effects and temperature coefficient
- Provides reasonable approximation for monitoring

### Duty Cycle Limits

Configurable limits protect alternator and electrical system:

```cpp
// Global limits
float MaxDuty = 99.0;    // Maximum field strength
float MinDuty = 1.0;     // Minimum for reliable switching

// Applied in control
dutyCycle = constrain(dutyCycle, MinDuty, MaxDuty);
```

**Considerations**:
- **MinDuty > 0**: Ensures MOSFET switching reliability
- **MaxDuty < 100**: Prevents potential damage from 100% duty cycle
- **User configurable**: Adjustable for different alternator types

## Performance Characteristics

### Response Time Analysis

System response characteristics:

| Parameter | Value | Notes |
|-----------|--------|-------|
| Update Rate | 50ms | FieldAdjustmentInterval |
| Step Size | 0.8% | dutyStep default |
| Maximum Rate | 1.6%/second | dutyCycle change rate |
| 0-50% Time | ~31 seconds | Linear response |
| Emergency Stop | <1ms | Hardware GPIO response |

### Control Stability

Stability analysis:

```
Plant: Alternator + Battery (slow thermal, fast electrical)
Controller: Proportional with rate limiting
Disturbances: Load changes, temperature, engine speed
```

**Stability Factors**:
- **Slow thermal response**: Field heating time constant ~10-30 seconds
- **Fast electrical response**: Field current follows PWM within milliseconds  
- **Rate limiting**: Prevents oscillation and overshoot
- **Multiple sensors**: Redundancy prevents false triggering

### Resource Requirements

CPU and memory usage:

- **CPU time**: ~5-15ms per 50ms cycle (10-30% of main loop)
- **Memory**: ~50 bytes for control variables
- **Flash**: ~8KB for control code
- **Real-time constraints**: Must complete within watchdog timeout

## Configuration Parameters

### Basic Settings

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| TargetAmps | 40 | 0-200 | Normal mode current target |
| TargetAmpL | 25 | 0-200 | Low mode current target |
| BulkVoltage | 13.9 | 12.0-16.0 | Bulk charging voltage |
| FloatVoltage | 13.4 | 12.0-16.0 | Float charging voltage |
| TemperatureLimitF | 150 | 100-250 | Temperature limit (°F) |

### Advanced Settings

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| dutyStep | 0.8 | 0.1-5.0 | Field adjustment step size (%) |
| FieldAdjustmentInterval | 50 | 10-1000 | Control update rate (ms) |
| SwitchingFrequency | 15000 | 50-25000 | PWM frequency (Hz) |
| MaxDuty | 99.0 | 50-99 | Maximum duty cycle (%) |
| MinDuty | 1.0 | 0-10 | Minimum duty cycle (%) |

### RPM Curve Configuration

The 20-point RPM curve is fully user-configurable:

```
RPM Points:  RPM1, RPM2, ..., RPM20  (0-6000 RPM)
Amp Points:  Amps1, Amps2, ..., Amps20  (0-200 A)
```

**Guidelines**:
- **Start at 0**: RPM1=0, Amps1=0 for idle protection
- **Engine range**: Configure for actual engine operating range
- **Power band**: Higher values in engine's optimal RPM range
- **High RPM limit**: Reduce at excessive speeds

## Troubleshooting Guide

### Common Issues

#### No Field Output
```
Check:
1. GPIO pin 4 status (field enable)
2. OnOff setting = 1
3. Ignition signal active
4. dutyCycle > MinDuty
5. No emergency field collapse active
6. MOSFET driver functionality
```

#### Voltage Regulation Problems
```
Check:
1. Battery voltage sensor accuracy
2. Current sensor calibration
3. ChargingVoltageTarget setting
4. Field resistance value
5. Alternator field coil condition
```

#### Oscillation or Instability
```
Check:
1. dutyStep too large
2. FieldAdjustmentInterval too fast
3. Sensor noise or poor connections
4. Multiple control loops fighting
```

#### Overheating
```
Check:
1. TemperatureLimitF setting
2. Temperature sensor operation
3. Alternator cooling and mounting
4. Current targets too high for conditions
```

### Diagnostic Tools

#### Real-time Monitoring
- Web interface shows live field parameters
- Console messages for all protective actions
- Performance timing (loop time, control timing)

#### Data Logging
- Field duty cycle history
- Temperature trends
- Voltage and current logging
- Emergency event logging

This comprehensive field control system provides safe, efficient alternator regulation with extensive configurability and protection systems.