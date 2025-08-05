# Battery Management & State Tracking

## Overview

The battery management system implements real-time state-of-charge (SOC) calculation using coulomb counting with Peukert correction, charge efficiency compensation, and dynamic calibration. The system tracks energy flow, runtime statistics, and provides comprehensive battery monitoring for lead-acid and lithium battery systems.

## State of Charge Calculation

### Core Algorithm

The SOC calculation runs every 2 seconds in `UpdateBatterySOC()`:

```cpp
void UpdateBatterySOC(unsigned long elapsedMillis) {
  float elapsedSeconds = elapsedMillis / 1000.0f;
  
  // Get current measurements
  float currentBatteryVoltage = getBatteryVoltage();
  float batteryCurrentForSoC = getBatteryCurrent();
  
  // Update scaled values for web interface
  Voltage_scaled = currentBatteryVoltage * 100;
  BatteryCurrent_scaled = batteryCurrentForSoC * 100;
  AlternatorCurrent_scaled = MeasuredAmps * 100;
  BatteryPower_scaled = (Voltage_scaled * BatteryCurrent_scaled) / 100;
  
  // Calculate Ah change using floating point precision
  static float coulombAccumulator_Ah = 0.0f;
  float batteryCurrent_A = BatteryCurrent_scaled / 100.0f;
  float deltaAh = (batteryCurrent_A * elapsedSeconds) / 3600.0f;
  
  if (BatteryCurrent_scaled >= 0) {
    // Charging: apply charge efficiency
    float batteryDeltaAh = deltaAh * (ChargeEfficiency_scaled / 100.0f);
    coulombAccumulator_Ah += batteryDeltaAh;
  } else {
    // Discharging: apply Peukert compensation
    float dischargeCurrent_A = abs(batteryCurrent_A);
    float peukertThreshold = BatteryCapacity_Ah / 100.0f;  // C/100 rate
    
    if (dischargeCurrent_A > peukertThreshold) {
      float peukertExponent = PeukertExponent_scaled / 100.0f;
      dischargeCurrent_A = constrain(dischargeCurrent_A, 0.1f, BatteryCapacity_Ah);
      
      float currentRatio = PeukertRatedCurrent_A / dischargeCurrent_A;
      float peukertFactor = pow(currentRatio, peukertExponent - 1.0f);
      peukertFactor = constrain(peukertFactor, 0.5f, 2.0f);
      
      float batteryDeltaAh = deltaAh * peukertFactor;
      coulombAccumulator_Ah += batteryDeltaAh;
    } else {
      coulombAccumulator_Ah += deltaAh;  // No Peukert correction below C/100
    }
  }
  
  // Update coulomb count when accumulator reaches 0.01Ah
  if (abs(coulombAccumulator_Ah) >= 0.01f) {
    int deltaAh_scaled = (int)(coulombAccumulator_Ah * 100.0f);
    CoulombCount_Ah_scaled += deltaAh_scaled;
    coulombAccumulator_Ah -= (deltaAh_scaled / 100.0f);
  }
  
  // Calculate SOC
  CoulombCount_Ah_scaled = constrain(CoulombCount_Ah_scaled, 0, BatteryCapacity_Ah * 100);
  float SoC_float = (float)CoulombCount_Ah_scaled / (BatteryCapacity_Ah * 100.0f) * 100.0f;
  SOC_percent = (int)(SoC_float * 100);  // Store as percentage × 10000 for 2 decimal precision
}
```

### Peukert Correction

The Peukert equation compensates for capacity reduction at high discharge rates:

**Standard Form**: `Cp = C20 * (I20/I)^(n-1)`

**Implementation**:
- **Threshold**: Only applied above C/100 discharge rate
- **Exponent**: User-configurable, typically 1.05-1.25 
- **Current ratio**: `PeukertRatedCurrent_A / dischargeCurrent_A`
- **Bounds**: Factor constrained to 0.5-2.0 for safety

**Example**:
```
Battery: 300Ah, Peukert exponent: 1.12
C/20 rate: 15A, Current discharge: 60A (C/5)
Current ratio: 15/60 = 0.25
Peukert factor: (0.25)^(1.12-1) = (0.25)^0.12 = 0.74
Effective capacity reduction: 26% at C/5 rate
```

### Charge Efficiency

Charging efficiency accounts for energy losses during charge:

```cpp
if (BatteryCurrent_scaled >= 0) {
  // Charging: apply efficiency factor
  float batteryDeltaAh = deltaAh * (ChargeEfficiency_scaled / 100.0f);
  coulombAccumulator_Ah += batteryDeltaAh;
}
```

**Typical Values**:
- **Lead-acid**: 85-95% efficiency
- **AGM**: 90-98% efficiency  
- **LiFePO4**: 95-99% efficiency
- **Gel**: 85-92% efficiency

### Current Source Selection

Battery current uses dedicated source selection for SOC accuracy:

```cpp
float getBatteryCurrent() {
  float batteryCurrent = 0;
  
  switch (BatteryCurrentSource) {
    case 0:  // INA228 Battery Shunt (default)
      if (INADisconnected == 0 && !isnan(Bcur)) {
        batteryCurrent = Bcur;  // Includes offset and dynamic gain
      }
      break;
      
    case 1:  // NMEA2K Battery
      // Implementation pending
      batteryCurrent = -Bcur;  // Temporary inversion
      break;
      
    case 3:  // Victron Battery
      if (abs(VictronCurrent) > 0.1) {
        batteryCurrent = VictronCurrent;  // External source, no inversion
      } else {
        batteryCurrent = Bcur;  // Fallback to INA228
      }
      break;
      
    default:
      batteryCurrent = Bcur;
      break;
  }
  
  return batteryCurrent;
}
```

## Full Charge Detection

### Detection Algorithm

Full charge resets SOC to 100% when conditions are met:

```cpp
// Check conditions: low current + high voltage + time
if ((abs(BatteryCurrent_scaled) <= (TailCurrent * BatteryCapacity_Ah / 100)) && 
    (Voltage_scaled >= ChargedVoltage_Scaled)) {
  
  FullChargeTimer += elapsedSeconds;
  
  if (FullChargeTimer >= ChargedDetectionTime) {
    SOC_percent = 10000;  // 100.00%
    CoulombCount_Ah_scaled = BatteryCapacity_Ah * 100;
    FullChargeDetected = true;
    coulombAccumulator_Ah = 0.0f;
    queueConsoleMessage("BATTERY: Full charge detected - SoC reset to 100%");
    applySocGainCorrection();  // Trigger dynamic calibration
  }
} else {
  FullChargeTimer = 0;
  FullChargeDetected = false;
}
```

### Detection Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| TailCurrent | 2 | 1-10 | % of capacity for "full" current |
| ChargedVoltage_Scaled | 1450 | 1200-1600 | Voltage threshold (V × 100) |
| ChargedDetectionTime | 1000 | 60-3600 | Time at conditions (seconds) |

**Example**: 300Ah battery, 2% tail current = 6A threshold

## Dynamic Calibration

### Automatic Shunt Gain Correction

The system learns and corrects shunt measurement errors over time:

```cpp
void applySocGainCorrection() {
  if (AutoShuntGainCorrection == 0 || AmpSrc != 1) return;
  
  unsigned long now = millis();
  if (now - lastGainCorrectionTime < MIN_GAIN_CORRECTION_INTERVAL) return;
  
  // Calculate capacity error
  float expectedCapacity = BatteryCapacity_Ah;
  float calculatedCapacity = CoulombCount_Ah_scaled / 100.0f;
  
  // Sanity checks
  if (calculatedCapacity < 10 || expectedCapacity < 10) return;
  
  float errorRatio = abs(expectedCapacity - calculatedCapacity) / expectedCapacity;
  if (errorRatio > MAX_REASONABLE_ERROR) return;  // 20% maximum
  
  // Calculate new gain factor
  float desiredCorrectionFactor = expectedCapacity / calculatedCapacity;
  float currentFactor = DynamicShuntGainFactor;
  float newFactor = currentFactor * desiredCorrectionFactor;
  
  // Apply rate limiting
  float maxChange = currentFactor * MAX_GAIN_ADJUSTMENT_PER_CYCLE;  // 5% max
  if (newFactor > currentFactor + maxChange) {
    newFactor = currentFactor + maxChange;
  } else if (newFactor < currentFactor - maxChange) {
    newFactor = currentFactor - maxChange;
  }
  
  // Apply bounds
  newFactor = constrain(newFactor, MIN_DYNAMIC_GAIN_FACTOR, MAX_DYNAMIC_GAIN_FACTOR);
  
  DynamicShuntGainFactor = newFactor;
  lastGainCorrectionTime = now;
}
```

**Correction Logic**:
- **Trigger**: Every full charge detection
- **Error calculation**: Actual vs expected capacity
- **Rate limiting**: Maximum 5% change per cycle
- **Bounds**: 0.8-1.2 gain factor range
- **Time interval**: Minimum 1 hour between corrections

### Automatic Current Zero Calibration

The system automatically zeros alternator current readings:

```cpp
void checkAutoZeroTriggers() {
  if (AutoAltCurrentZero == 0 || autoZeroStartTime > 0 || RPM < 200) return;
  
  unsigned long now = millis();
  bool shouldTrigger = false;
  String triggerReason = "";
  
  // Time-based trigger (1 hour)
  if (now - lastAutoZeroTime >= AUTO_ZERO_INTERVAL) {
    shouldTrigger = true;
    triggerReason = "scheduled interval";
  }
  
  // Temperature-based trigger (20°F change)
  float currentTemp = (TempSource == 0) ? AlternatorTemperatureF : temperatureThermistor;
  if (lastAutoZeroTemp > -900 && abs(currentTemp - lastAutoZeroTemp) >= AUTO_ZERO_TEMP_DELTA) {
    shouldTrigger = true;
    triggerReason = "temperature change (" + String(abs(currentTemp - lastAutoZeroTemp), 1) + "°F)";
  }
  
  if (shouldTrigger) {
    startAutoZero(triggerReason);
  }
}

void processAutoZero() {
  if (autoZeroStartTime == 0) return;
  
  unsigned long elapsed = millis() - autoZeroStartTime;
  
  if (elapsed < AUTO_ZERO_DURATION) {
    // Force field to minimum and disable
    dutyCycle = MinDuty;
    setDutyPercent((int)dutyCycle);
    digitalWrite(4, 0);
    
    // Accumulate readings after 2 second settling
    if (elapsed > 2000) {
      autoZeroAccumulator += MeasuredAmps;
      autoZeroSampleCount++;
    }
  } else {
    // Calculate average and restore operation
    if (autoZeroSampleCount > 0) {
      DynamicAltCurrentZero = autoZeroAccumulator / autoZeroSampleCount;
    }
    
    lastAutoZeroTime = millis();
    lastAutoZeroTemp = (TempSource == 0) ? AlternatorTemperatureF : temperatureThermistor;
    autoZeroStartTime = 0;
    autoZeroAccumulator = 0.0;
    autoZeroSampleCount = 0;
  }
}
```

**Auto-Zero Process**:
1. **Trigger conditions**: 1 hour interval OR 20°F temperature change
2. **Zero procedure**: 10 seconds at minimum field with settling time
3. **Measurement**: Average readings over 8-second period
4. **Application**: New zero offset stored and applied
5. **Frequency**: Thermal drift compensation

## Energy Tracking

### Energy Calculation

Real-time energy tracking with precision accumulators:

```cpp
// Calculate power
float batteryPower_W = BatteryPower_scaled / 100.0f;
float energyDelta_Wh = (batteryPower_W * elapsedSeconds) / 3600.0f;

float alternatorPower_W = (currentBatteryVoltage * MeasuredAmps);
float altEnergyDelta_Wh = (alternatorPower_W * elapsedSeconds) / 3600.0f;

// Energy accumulation with precision preservation
static float chargedEnergyAccumulator = 0.0f;
static float dischargedEnergyAccumulator = 0.0f;
static float alternatorEnergyAccumulator = 0.0f;

if (BatteryCurrent_scaled > 0) {
  // Charging
  chargedEnergyAccumulator += energyDelta_Wh;
  if (chargedEnergyAccumulator >= 1.0f) {
    ChargedEnergy += (int)chargedEnergyAccumulator;
    chargedEnergyAccumulator -= (int)chargedEnergyAccumulator;
  }
} else if (BatteryCurrent_scaled < 0) {
  // Discharging
  dischargedEnergyAccumulator += abs(energyDelta_Wh);
  if (dischargedEnergyAccumulator >= 1.0f) {
    DischargedEnergy += (int)dischargedEnergyAccumulator;
    dischargedEnergyAccumulator -= (int)dischargedEnergyAccumulator;
  }
}

// Alternator energy
if (altEnergyDelta_Wh > 0) {
  alternatorEnergyAccumulator += altEnergyDelta_Wh;
  if (alternatorEnergyAccumulator >= 1.0f) {
    AlternatorChargedEnergy += (int)alternatorEnergyAccumulator;
    alternatorEnergyAccumulator -= (int)alternatorEnergyAccumulator;
  }
}
```

**Energy Variables**:
- `ChargedEnergy`: Total energy into battery (Wh)
- `DischargedEnergy`: Total energy out of battery (Wh) 
- `AlternatorChargedEnergy`: Total alternator output (Wh)

### Fuel Consumption Calculation

Estimates fuel consumption based on alternator energy output:

```cpp
if (altEnergyDelta_Wh > 0) {
  // Convert energy to fuel consumption
  float energyJoules = altEnergyDelta_Wh * 3600.0f;  // Wh to J
  const float engineEfficiency = 0.30f;             // 30% thermal efficiency
  const float alternatorEfficiency = 0.50f;         // 50% mechanical efficiency
  float fuelEnergyUsed_J = energyJoules / (engineEfficiency * alternatorEfficiency);
  
  const float dieselEnergy_J_per_mL = 36000.0f;     // Diesel energy content
  float fuelUsed_mL = fuelEnergyUsed_J / dieselEnergy_J_per_mL;
  
  // Accumulate with precision
  static float fuelAccumulator = 0.0f;
  fuelAccumulator += fuelUsed_mL;
  if (fuelAccumulator >= 1.0f) {
    AlternatorFuelUsed += (int)fuelAccumulator;
    fuelAccumulator -= (int)fuelAccumulator;
  }
}
```

**Fuel Calculation Assumptions**:
- Engine thermal efficiency: 30%
- Alternator mechanical efficiency: 50%
- Combined system efficiency: 15%
- Diesel energy content: 36 kJ/mL (10 kWh/liter)

## Runtime Tracking

### Engine Runtime

Tracks engine operation based on RPM:

```cpp
void UpdateEngineRuntime(unsigned long elapsedMillis) {
  bool engineIsRunning = (RPM > 100 && RPM < 6000);
  
  if (engineIsRunning) {
    engineRunAccumulator += elapsedMillis;
    
    if (engineRunAccumulator >= 1000) {  // Update every second
      int secondsRun = engineRunAccumulator / 1000;
      EngineRunTime += secondsRun;
      
      // Calculate engine cycles (RPM integration)
      EngineCycles += (RPM * secondsRun) / 60;
      
      engineRunAccumulator %= 1000;  // Keep remainder
    }
  }
  
  engineWasRunning = engineIsRunning;
}
```

### Alternator Runtime

Tracks alternator active time based on output current:

```cpp
bool alternatorIsOn = (AlternatorCurrent_scaled > CurrentThreshold * 100);

if (alternatorIsOn) {
  alternatorOnAccumulator += elapsedMillis;
  if (alternatorOnAccumulator >= 60000) {  // Update every minute
    AlternatorOnTime += alternatorOnAccumulator / 60000;
    alternatorOnAccumulator %= 60000;
  }
}
```

**Runtime Variables**:
- `EngineRunTime`: Total engine seconds
- `EngineCycles`: RPM × time integration
- `AlternatorOnTime`: Active alternator minutes

## Time to Full/Empty Calculation

### Calculation Algorithm

Estimates time to full charge or empty discharge:

```cpp
void calculateChargeTimes() {
  static unsigned long lastCalcTime = 0;
  if (millis() - lastCalcTime < AnalogInputReadInterval) return;
  lastCalcTime = millis();
  
  float currentAmps = getTargetAmps();
  
  if (currentAmps > 0.01) {  // Charging
    float currentSoC = SOC_percent / 100.0;  // Convert from scaled format
    float remainingCapacity = BatteryCapacity_Ah * (100.0 - currentSoC) / 100.0;
    timeToFullChargeMin = (int)(remainingCapacity / currentAmps * 60.0);
    timeToFullDischargeMin = -999;  // Not applicable
  } else if (currentAmps < -0.01) {  // Discharging
    float currentSoC = SOC_percent / 100.0;
    float availableCapacity = BatteryCapacity_Ah * currentSoC / 100.0;
    timeToFullDischargeMin = (int)(availableCapacity / (-currentAmps) * 60.0);
    timeToFullChargeMin = -999;  // Not applicable
  } else {
    timeToFullChargeMin = -999;  // No significant current
    timeToFullDischargeMin = -999;
  }
}
```

**Calculation Notes**:
- Uses current amp rate for projection
- Does not account for charge taper or Peukert effects
- Provides reasonable estimate for planning purposes
- Updates every 2 seconds with current conditions

## Charging Stage Management

### Bulk/Float Stage Logic

Implements multi-stage charging algorithm:

```cpp
void updateChargingStage() {
  float currentVoltage = getBatteryVoltage();
  
  if (inBulkStage) {
    ChargingVoltageTarget = BulkVoltage;  // Typically 13.9V for 12V system
    
    if (currentVoltage >= ChargingVoltageTarget) {
      if (bulkCompleteTimer == 0) {
        bulkCompleteTimer = millis();
      } else if (millis() - bulkCompleteTimer > bulkCompleteTime) {
        inBulkStage = false;
        floatStartTime = millis();
        queueConsoleMessage("CHARGING: Bulk stage complete, switching to float");
      }
    } else {
      bulkCompleteTimer = 0;
    }
  } else {
    ChargingVoltageTarget = FloatVoltage;  // Typically 13.4V for 12V system
    
    // Return to bulk if time expires or voltage drops
    if ((millis() - floatStartTime > FLOAT_DURATION * 1000) || 
        (currentVoltage < FloatVoltage - 0.5)) {
      inBulkStage = true;
      bulkCompleteTimer = 0;
      floatStartTime = millis();
      queueConsoleMessage("CHARGING: Returning to bulk stage");
    }
  }
}
```

**Stage Parameters**:
- `BulkVoltage`: Higher voltage for fast charging (13.9V default)
- `FloatVoltage`: Lower voltage for maintenance (13.4V default) 
- `bulkCompleteTime`: Time at bulk voltage before float (1 second default)
- `FLOAT_DURATION`: Maximum float time before returning to bulk (12 hours default)

## Configuration Parameters

### Core Battery Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| BatteryCapacity_Ah | 300 | 50-2000 | Battery capacity in amp-hours |
| PeukertExponent_scaled | 105 | 100-130 | Peukert exponent × 100 (1.05) |
| ChargeEfficiency_scaled | 99 | 80-100 | Charging efficiency % |
| ChargedVoltage_Scaled | 1450 | 1200-1600 | Full charge voltage × 100 |
| TailCurrent | 2 | 1-10 | % of capacity for full detection |
| ChargedDetectionTime | 1000 | 60-3600 | Time at full conditions (seconds) |

### Current Measurement

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| ShuntResistanceMicroOhm | 100 | 50-1000 | Shunt resistance (µΩ) |
| BatteryCOffset | 0 | -50-50 | Battery current offset (A) |
| AlternatorCOffset | 0 | -50-50 | Alternator current offset (A) |
| CurrentThreshold | 1 | 0.1-10 | Minimum current for tracking (A) |
| InvertBattAmps | 0 | 0-1 | Invert battery current polarity |
| InvertAltAmps | 1 | 0-1 | Invert alternator current polarity |

### Dynamic Calibration

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| AutoShuntGainCorrection | 0 | 0-1 | Enable automatic gain correction |
| AutoAltCurrentZero | 0 | 0-1 | Enable automatic zero correction |
| MIN_GAIN_CORRECTION_INTERVAL | 3600000 | 1800000-86400000 | Min time between corrections (ms) |
| MAX_GAIN_ADJUSTMENT_PER_CYCLE | 0.05 | 0.01-0.2 | Max gain change per cycle |
| AUTO_ZERO_INTERVAL | 3600000 | 1800000-86400000 | Auto-zero interval (ms) |
| AUTO_ZERO_TEMP_DELTA | 20.0 | 5.0-50.0 | Temperature change trigger (°F) |

### Charging Stages

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| BulkVoltage | 13.9 | 12.0-16.0 | Bulk charging voltage |
| FloatVoltage | 13.4 | 12.0-16.0 | Float charging voltage |
| bulkCompleteTime | 1000 | 100-10000 | Time at bulk before float (ms) |
| FLOAT_DURATION | 43200 | 3600-86400 | Float duration (seconds) |

## Data Storage and Persistence

### NVS Storage

Critical battery data is stored in non-volatile storage:

```cpp
void saveNVSData() {
  nvs_handle_t nvs_handle;
  esp_err_t err = nvs_open("storage", NVS_READWRITE, &nvs_handle);
  
  // Battery state
  nvs_set_i32(nvs_handle, "SOC_percent", (int32_t)SOC_percent);
  nvs_set_i32(nvs_handle, "CoulombCount", (int32_t)CoulombCount_Ah_scaled);
  
  // Energy tracking
  nvs_set_u32(nvs_handle, "ChargedEnergy", (uint32_t)ChargedEnergy);
  nvs_set_u32(nvs_handle, "DischrgdEnergy", (uint32_t)DischargedEnergy);
  nvs_set_u32(nvs_handle, "AltChrgdEnergy", (uint32_t)AlternatorChargedEnergy);
  nvs_set_i32(nvs_handle, "AltFuelUsed", (int32_t)AlternatorFuelUsed);
  
  // Runtime tracking
  nvs_set_i32(nvs_handle, "EngineRunTime", (int32_t)EngineRunTime);
  nvs_set_i32(nvs_handle, "EngineCycles", (int32_t)EngineCycles);
  nvs_set_i32(nvs_handle, "AltOnTime", (int32_t)AlternatorOnTime);
  
  // Dynamic calibration
  nvs_set_blob(nvs_handle, "ShuntGain", &DynamicShuntGainFactor, sizeof(float));
  nvs_set_blob(nvs_handle, "AltZero", &DynamicAltCurrentZero, sizeof(float));
  
  nvs_commit(nvs_handle);
  nvs_close(nvs_handle);
}
```

**Persistent Data**:
- SOC state and coulomb count
- Energy totals (charged/discharged/alternator)
- Runtime statistics
- Dynamic calibration factors
- Fuel consumption estimates

### Update Intervals

| Function | Interval | Purpose |
|----------|----------|---------|
| UpdateBatterySOC | 2000ms | Core SOC calculation |
| calculateChargeTimes | 2000ms | Time to full/empty |
| saveNVSData | 10000ms | Persistent storage |
| applySocGainCorrection | On full charge | Dynamic calibration |
| processAutoZero | 1 hour / temp change | Current zero calibration |

## Error Handling

### Sensor Validation

Battery current measurement includes extensive validation:

```cpp
float getBatteryCurrent() {
  static unsigned long lastWarningTime = 0;
  const unsigned long WARNING_INTERVAL = 10000;
  
  switch (BatteryCurrentSource) {
    case 0:  // INA228
      if (INADisconnected == 0 && !isnan(Bcur)) {
        return Bcur;
      } else {
        if (millis() - lastWarningTime > WARNING_INTERVAL) {
          queueConsoleMessage("WARNING: INA228 unavailable, using fallback");
          lastWarningTime = millis();
        }
        return 0;  // Safe default
      }
      break;
      
    case 3:  // Victron
      if (abs(VictronCurrent) > 0.1) {
        return VictronCurrent;
      } else {
        queueConsoleMessage("Victron current not available, using INA228");
        return Bcur;
      }
      break;
  }
}
```

### Bounds Checking

All SOC calculations include bounds enforcement:

```cpp
// Constrain coulomb count to physical limits
CoulombCount_Ah_scaled = constrain(CoulombCount_Ah_scaled, 0, BatteryCapacity_Ah * 100);

// Constrain SOC percentage
float SoC_float = (float)CoulombCount_Ah_scaled / (BatteryCapacity_Ah * 100.0f) * 100.0f;
SOC_percent = (int)(SoC_float * 100);  // 0-10000 range (0.00-100.00%)

// Constrain Peukert factor
peukertFactor = constrain(peukertFactor, 0.5f, 2.0f);

// Constrain dynamic gain factor
DynamicShuntGainFactor = constrain(DynamicShuntGainFactor, 
                                  MIN_DYNAMIC_GAIN_FACTOR, 
                                  MAX_DYNAMIC_GAIN_FACTOR);
```

### Recovery Procedures

System includes automatic recovery from various failure modes:

- **Sensor failure**: Automatic fallback to secondary sensors
- **Invalid readings**: Bounds checking and error indication
- **Calibration drift**: Automatic gain and zero correction
- **Power cycle**: Full state restoration from NVS storage
- **Coulomb counter overflow**: Reset to known state at full charge

The battery management system provides comprehensive monitoring and state tracking with automatic calibration and error recovery, enabling accurate SOC determination across a wide range of operating conditions and battery types.