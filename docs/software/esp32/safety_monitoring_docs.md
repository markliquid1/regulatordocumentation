# Safety & Monitoring Systems

## Overview

The regulator implements multiple independent safety systems to protect the alternator, battery, electrical system, and controller hardware. These systems operate at different response times and protection levels, from immediate hardware shutdowns to adaptive learning algorithms.

## Protection Layer Architecture

### Layer 1: Hardware Protection (< 1ms response)
- **INA228 overvoltage detection**: Independent hardware threshold monitoring
- **GPIO emergency shutdown**: Software-triggered immediate MOSFET disable
- **Flyback diode protection**: Field coil inductive spike suppression

### Layer 2: Real-time Software Protection (50ms response)
- **Emergency field collapse**: Voltage spike detection and rapid shutdown
- **Temperature limits**: Immediate field reduction on overheating
- **Current limits**: Battery and alternator current protection

### Layer 3: Adaptive Protection (seconds to minutes)
- **Learning mode penalties**: Thermal history-based current reduction
- **Charging stage management**: Bulk/float transitions for battery protection
- **Dynamic sensor validation**: Cross-checking redundant measurements

### Layer 4: System Health Monitoring (continuous)
- **Watchdog protection**: 15-second hang detection with restart
- **Memory monitoring**: Heap and stack overflow detection
- **Performance tracking**: Loop time and CPU utilization analysis

## Hardware Protection Systems

### INA228 Overvoltage Protection

The INA228 provides independent hardware monitoring with configurable voltage thresholds:

```cpp
// Configure hardware overvoltage threshold
uint16_t thresholdLSB = (uint16_t)(VoltageHardwareLimit / 0.003125);
INA.setBusOvervoltageTH(thresholdLSB);
INA.setDiagnoseAlertBit(INA228_DIAG_BUS_OVER_LIMIT);

// Enable latching alert with ALERT pin control
Wire.beginTransmission(INA.getAddress());
Wire.write(0x0F);  // ALERT_MASK_ENABLE register
Wire.write(0x98);  // Enable ALERT pin + latch + bus overvoltage
Wire.write(0x00);
Wire.endTransmission();
```

**Operation**:
- **Threshold**: BulkVoltage + 0.1V (typically 14.0V for 12V systems)
- **Response time**: <1ms hardware detection
- **Action**: Pulls ALERT pin low, sets status bit
- **Recovery**: Automatic when voltage drops below threshold

**Monitoring Loop**:
```cpp
// Check for hardware overvoltage alert
uint16_t alertStatus = readINA228AlertRegister(INA.getAddress());
if (!inaOvervoltageLatched && (alertStatus & 0x0080)) {
  inaOvervoltageLatched = true;
  inaOvervoltageTime = millis();
  queueConsoleMessage("INA228 hardware overvoltage detected! Field disabled until corrected");
}

// Auto-clear after 10 seconds if condition resolved
if (inaOvervoltageLatched && millis() - inaOvervoltageTime >= 10000) {
  clearINA228AlertLatch(INA.getAddress());
  if (!(readINA228AlertRegister(INA.getAddress()) & 0x0080)) {
    inaOvervoltageLatched = false;
    queueConsoleMessage("INA228 overvoltage condition cleared");
  }
}
```

### Field Coil Protection

Inductive kickback protection prevents MOSFET damage during field switching:

**Circuit Design**:
```
Battery 12V → Field Coil (2-6Ω, ~50mH) → MOSFET → Ground
                ↑
            Flyback Diode (1N4007 or equivalent)
```

**Protection Mechanisms**:
- **Flyback diode**: Provides current path during MOSFET turn-off
- **MOSFET ratings**: Selected for 2x maximum field current and voltage
- **PWM frequency**: 15kHz above audible range, below EMI concerns

## Real-Time Software Protection

### Emergency Field Collapse

Immediate field shutdown on dangerous voltage spikes:

```cpp
void emergencyFieldCollapse() {
  if (currentBatteryVoltage > (ChargingVoltageTarget + 0.2)) {
    digitalWrite(4, 0);              // Immediate MOSFET disable
    dutyCycle = MinDuty;
    setDutyPercent((int)dutyCycle);
    fieldCollapseTime = currentTime;
    queueConsoleMessage("EMERGENCY: Field collapsed - voltage spike (" + 
                       String(currentBatteryVoltage, 2) + "V)");
    return;  // Exit field control immediately
  }
}
```

**Recovery Logic**:
```cpp
// Maintain shutdown for 10 seconds
if (fieldCollapseTime > 0 && (currentTime - fieldCollapseTime) < FIELD_COLLAPSE_DELAY) {
  digitalWrite(4, 0);
  dutyCycle = MinDuty;
  setDutyPercent((int)dutyCycle);
  return;
}

// Clear flag and resume normal operation
if (fieldCollapseTime > 0 && (currentTime - fieldCollapseTime) >= FIELD_COLLAPSE_DELAY) {
  fieldCollapseTime = 0;
  queueConsoleMessage("Field collapse delay expired - normal operation resumed");
}
```

**Parameters**:
- **Trigger threshold**: ChargingVoltageTarget + 0.2V
- **Lockout time**: 10 seconds (FIELD_COLLAPSE_DELAY)
- **Response time**: <50ms (next control cycle)

### Temperature Protection

Hierarchical temperature protection with increasing severity:

```cpp
void temperatureProtection() {
  // Determine active temperature source
  if (TempSource == 0) {
    TempToUse = AlternatorTemperatureF;  // OneWire DS18B20
  } else {
    TempToUse = temperatureThermistor;   // ADS1115 thermistor
  }
  
  // Progressive protection levels
  if (!IgnoreTemperature && TempToUse > TemperatureLimitF) {
    if (dutyCycle > (MinDuty + 2 * dutyStep)) {
      dutyCycle -= 2 * dutyStep;  // Aggressive 1.6% reduction
      queueConsoleMessage("Temperature limit reached, backing off...");
    }
  }
  
  // Emergency temperature shutdown
  if (TempToUse > (TemperatureLimitF + 20)) {
    dutyCycle = MinDuty;
    digitalWrite(4, 0);
    queueConsoleMessage("EMERGENCY: Temperature excessive - field shutdown");
  }
}
```

**Temperature Sources**:
- **OneWire DS18B20**: Digital sensor, ±0.5°C accuracy, noise immune
- **Thermistor**: Analog sensor via ADS1115, faster response
- **Source selection**: User configurable via TempSource setting

### Current Limiting

Multiple current protection systems prevent equipment damage:

```cpp
void currentProtection() {
  // Battery current protection
  if (Bcur > MaximumAllowedBatteryAmps && dutyCycle > (MinDuty + dutyStep)) {
    dutyCycle -= dutyStep;
    queueConsoleMessage("Battery current limit reached, backing off...");
  }
  
  // Alternator current protection (implicit via voltage regulation)
  // High alternator current → voltage drop → increased field → voltage recovery
  
  // BMS current limit integration
  if (bmsLogic == 1) {
    bmsSignalActive = !digitalRead(36);
    if ((bmsLogicLevelOff == 0 && !bmsSignalActive) || 
        (bmsLogicLevelOff == 1 && bmsSignalActive)) {
      // BMS requesting charge stop
      chargingEnabled = false;
      queueConsoleMessage("BMS requesting charge stop");
    }
  }
}
```

### Voltage Protection

Multi-level voltage protection prevents overcharging:

```cpp
void voltageProtection() {
  float currentBatteryVoltage = getBatteryVoltage();
  
  // Standard voltage regulation
  if (currentBatteryVoltage > ChargingVoltageTarget && 
      dutyCycle > (MinDuty + 3 * dutyStep)) {
    dutyCycle -= 3 * dutyStep;  // Most aggressive: 2.4% reduction
    queueConsoleMessage("Voltage limit reached, backing off...");
  }
  
  // Cross-validation between voltage sensors
  if (abs(BatteryV - IBV) > 0.1) {
    queueConsoleMessage("Voltage sensor disagreement - Field shut off for safety!");
    digitalWrite(33, HIGH);  // Alarm
    digitalWrite(4, 0);      // Field disable
    dutyCycle = MinDuty;
    return;
  }
}
```

**Voltage Sources Compared**:
- `BatteryV`: ADS1115 via voltage divider (1MΩ/75kΩ)
- `IBV`: INA228 high-precision measurement
- **Cross-validation**: 0.1V maximum disagreement allowed

## Alarm System

### Alarm Conditions

The system monitors multiple parameters and triggers alarms when limits are exceeded:

```cpp
void CheckAlarms() {
  static unsigned long lastRunTime = 0;
  if (millis() - lastRunTime < 250) return;  // 250ms update rate
  lastRunTime = millis();
  
  bool currentAlarmCondition = false;
  String alarmReason = "";
  
  if (AlarmActivate == 1) {
    // Temperature alarm
    if (TempAlarm > 0 && TempToUse > TempAlarm) {
      currentAlarmCondition = true;
      alarmReason = "High alternator temperature: " + String(TempToUse) + 
                   "°F (limit: " + String(TempAlarm) + "°F)";
    }
    
    // Voltage alarms
    float currentVoltage = getBatteryVoltage();
    if (VoltageAlarmHigh > 0 && currentVoltage > VoltageAlarmHigh) {
      currentAlarmCondition = true;
      alarmReason = "High battery voltage: " + String(currentVoltage, 2) + 
                   "V (limit: " + String(VoltageAlarmHigh) + "V)";
    }
    
    if (VoltageAlarmLow > 0 && currentVoltage < VoltageAlarmLow && currentVoltage > 8.0) {
      currentAlarmCondition = true;
      alarmReason = "Low battery voltage: " + String(currentVoltage, 2) + 
                   "V (limit: " + String(VoltageAlarmLow) + "V)";
    }
    
    // Current alarms
    if (CurrentAlarmHigh > 0 && MeasuredAmps > CurrentAlarmHigh) {
      currentAlarmCondition = true;
      alarmReason = "High alternator current: " + String(MeasuredAmps, 1) + 
                   "A (limit: " + String(CurrentAlarmHigh) + "A)";
    }
  }
  
  // Apply alarm output with latching logic
  processAlarmOutput(currentAlarmCondition, alarmReason);
}
```

### Alarm Processing and Latching

```cpp
void processAlarmOutput(bool currentCondition, String reason) {
  static bool previousAlarmState = false;
  bool outputAlarmState = false;
  
  // Handle alarm test (always works regardless of AlarmActivate)
  if (AlarmTest == 1) {
    if (alarmTestStartTime == 0) {
      alarmTestStartTime = millis();
      queueConsoleMessage("ALARM TEST: Testing buzzer for 2 seconds");
    }
    
    if (millis() - alarmTestStartTime < ALARM_TEST_DURATION) {
      currentCondition = true;
    } else {
      AlarmTest = 0;
      alarmTestStartTime = 0;
    }
  }
  
  // Handle manual latch reset
  if (ResetAlarmLatch == 1) {
    alarmLatch = false;
    ResetAlarmLatch = 0;
    queueConsoleMessage("ALARM LATCH: Manually reset");
  }
  
  // Latching logic
  if (AlarmLatchEnabled == 1) {
    if (currentCondition) alarmLatch = true;
    outputAlarmState = alarmLatch;
  } else {
    outputAlarmState = currentCondition;
  }
  
  // Final output control
  bool finalOutput = false;
  if (AlarmTest == 1 || (AlarmActivate == 1 && outputAlarmState)) {
    finalOutput = true;
  }
  
  digitalWrite(33, finalOutput ? HIGH : LOW);
  
  // Console messaging
  if (currentCondition != previousAlarmState) {
    if (currentCondition) {
      queueConsoleMessage("ALARM ACTIVATED: " + reason);
    } else if (AlarmLatchEnabled == 0) {
      queueConsoleMessage("ALARM CLEARED");
    }
    previousAlarmState = currentCondition;
  }
}
```

**Alarm Features**:
- **Configurable thresholds**: User-adjustable limits for all parameters
- **Latching mode**: Alarm stays on until manually reset
- **Test function**: 2-second buzzer test
- **Manual reset**: Web interface reset capability

## System Health Monitoring

### Watchdog Protection

15-second watchdog timer prevents system hangs:

```cpp
void setupWatchdog() {
  esp_task_wdt_config_t wdt_config = {
    .timeout_ms = 15000,   // 15 seconds
    .idle_core_mask = 0,   // Don't monitor idle cores
    .trigger_panic = true  // Reboot on timeout
  };
  esp_task_wdt_init(&wdt_config);
  esp_task_wdt_add(NULL);  // Add main loop task
  queueConsoleMessage("Watchdog enabled: 15 second timeout for safety");
}

void loop() {
  esp_task_wdt_reset();  // Feed watchdog every loop iteration
  // ... main loop code ...
}
```

**Protection Logic**:
- **Timeout**: 15 seconds without watchdog reset
- **Action**: Hardware reset of ESP32
- **Recovery**: Full system restart with safety state
- **Field safety**: GPIO pins reset to LOW → field disabled

### Memory Monitoring

Continuous heap and stack monitoring with early warning:

```cpp
void updateSystemHealthMetrics() {
  // Heap monitoring
  rawFreeHeap = esp_get_free_heap_size();
  FreeHeap = rawFreeHeap / 1024;  // Convert to KB
  MinFreeHeap = esp_get_minimum_free_heap_size() / 1024;
  FreeInternalRam = heap_caps_get_free_size(MALLOC_CAP_INTERNAL) / 1024;
  
  // Fragmentation calculation
  if (rawFreeHeap == 0) {
    Heapfrag = 100;
  } else {
    Heapfrag = 100 - ((heap_caps_get_largest_free_block(MALLOC_CAP_8BIT) * 100) / rawFreeHeap);
  }
  
  // Critical heap warnings (throttled)
  unsigned long now = millis();
  if (FreeHeap < 20 && (now - lastHeapWarningTime > WARNING_THROTTLE_INTERVAL)) {
    queueConsoleMessage("CRITICAL: Heap dangerously low (" + String(FreeHeap) + "KB)");
    lastHeapWarningTime = now;
  }
}
```

**Stack Monitoring**:
```cpp
void printBasicTaskStackInfo() {
  numTasks = uxTaskGetNumberOfTasks();
  if (numTasks > MAX_TASKS) numTasks = MAX_TASKS;
  
  tasksCaptured = uxTaskGetSystemState(taskArray, numTasks, NULL);
  
  for (int i = 0; i < tasksCaptured; i++) {
    stackBytes = taskArray[i].usStackHighWaterMark * sizeof(StackType_t);
    const char* taskName = taskArray[i].pcTaskName;
    
    if (stackBytes < 256) {
      queueConsoleMessage("CRITICAL: " + String(taskName) + " stack very low (" + 
                         String(stackBytes) + "B)");
    }
  }
}
```

### Performance Monitoring

Loop timing and CPU utilization tracking:

```cpp
void loop() {
  starttime = esp_timer_get_time();  // Start timing
  
  // ... main loop execution ...
  
  endtime = esp_timer_get_time();
  LoopTime = (endtime - starttime);  // Microseconds
  
  if (LoopTime > 5000000) {  // 5 seconds
    queueConsoleMessage("WARNING: Loop took " + String(LoopTime / 1000) + 
                       "ms - potential watchdog risk");
  }
  
  // Track maximum times
  if (LoopTime > MaximumLoopTime) MaximumLoopTime = LoopTime;
  if (LoopTime > MaxLoopTime) MaxLoopTime = LoopTime;  // Persistent across sessions
}
```

**CPU Load Tracking**:
```cpp
void updateCpuLoad() {
  // Get IDLE task runtime counters
  for (int i = 0; i < taskCount; i++) {
    if (strcmp(taskSnapshot[i].pcTaskName, "IDLE0") == 0) {
      idle0Time = taskSnapshot[i].ulRunTimeCounter;
    } else if (strcmp(taskSnapshot[i].pcTaskName, "IDLE1") == 0) {
      idle1Time = taskSnapshot[i].ulRunTimeCounter;
    }
  }
  
  // Calculate CPU load as percentage
  unsigned long deltaIdle0 = idle0Time - lastIdle0Time;
  unsigned long deltaIdle1 = idle1Time - lastIdle1Time;
  unsigned long timeDiff = millis() - lastCheckTime;
  
  cpuLoadCore0 = 100 - ((deltaIdle0 * 100) / (timeDiff * 100));
  cpuLoadCore1 = 100 - ((deltaIdle1 * 100) / (timeDiff * 100));
  
  cpuLoadCore0 = constrain(cpuLoadCore0, 0, 100);
  cpuLoadCore1 = constrain(cpuLoadCore1, 0, 100);
}
```

## Data Validation and Sensor Safety

### Sensor Freshness Tracking

Data age monitoring prevents control decisions based on stale readings:

```cpp
// Global data freshness system
enum DataIndex {
  IDX_ALTERNATOR_TEMP = 0,
  IDX_BATTERY_V,
  IDX_MEASURED_AMPS,
  // ... 17 total sensor data indices
  MAX_DATA_INDICES = 17
};

unsigned long dataTimestamps[MAX_DATA_INDICES];
const unsigned long DATA_TIMEOUT = 10000;  // 10 seconds

// Mark data fresh when successfully read
#define MARK_FRESH(index) dataTimestamps[index] = millis()

// Check if data is stale
#define IS_STALE(index) (millis() - dataTimestamps[index] > DATA_TIMEOUT)

// Conditional assignment for stale data
#define SET_IF_STALE(index, variable, staleValue) \
  if (IS_STALE(index)) { variable = staleValue; }
```

**Usage in Control Systems**:
```cpp
// Check temperature data freshness for safety
unsigned long tempAge = currentTime - dataTimestamps[IDX_ALTERNATOR_TEMP];
bool tempDataVeryStale = (tempAge > 30000);  // 30 seconds
if (tempDataVeryStale) {
  queueConsoleMessage("OneWire sensor stale - sensor dead or disconnected");
  digitalWrite(33, HIGH);  // Sound alarm
}
```

### Sensor Cross-Validation

Multiple sensors for critical measurements with disagreement detection:

```cpp
float getBatteryVoltage() {
  float selectedVoltage = 0;
  static unsigned long lastWarningTime = 0;
  
  switch (BatteryVoltageSource) {
    case 0:  // INA228 (preferred)
      if (!IS_STALE(IDX_IBV) && IBV > 8.0 && IBV < 70.0) {
        selectedVoltage = IBV;
      } else {
        if (millis() - lastWarningTime > 10000) {
          queueConsoleMessage("INA228 unavailable, falling back to ADS1115");
          lastWarningTime = millis();
        }
        selectedVoltage = BatteryV;  // Automatic fallback
      }
      break;
      
    case 1:  // ADS1115 (backup)
      if (!IS_STALE(IDX_BATTERY_V) && BatteryV > 8.0 && BatteryV < 70.0) {
        selectedVoltage = BatteryV;
      } else {
        queueConsoleMessage("ADS1115 unavailable, falling back to INA228");
        selectedVoltage = IBV;
      }
      break;
  }
  
  // Final validation
  if (selectedVoltage < 8.0 || selectedVoltage > 70.0 || isnan(selectedVoltage)) {
    queueConsoleMessage("CRITICAL: No valid battery voltage found!");
    selectedVoltage = 999;  // Error indication
  }
  
  return selectedVoltage;
}
```

## Thermal Stress Monitoring

### Alternator Lifetime Modeling

Continuous thermal stress calculation based on Arrhenius equation:

```cpp
void calculateThermalStress() {
  unsigned long now = millis();
  if (now - lastThermalUpdateTime < THERMAL_UPDATE_INTERVAL) return;
  
  float elapsedSeconds = (now - lastThermalUpdateTime) / 1000.0f;
  lastThermalUpdateTime = now;
  
  // Calculate component temperatures
  float T_winding_F = TempToUse + WindingTempOffset;      // User-configurable offset
  float T_bearing_F = TempToUse + 40.0f;                  // Fixed offset
  float T_brush_F = TempToUse + 50.0f;                    // Fixed offset
  
  // Calculate alternator RPM
  float Alt_RPM = RPM * PulleyRatio;
  
  // Calculate component life expectancies (hours)
  float T_winding_K = (T_winding_F - 32.0f) * 5.0f / 9.0f + 273.15f;
  float L_insul = L_REF_INSUL * exp(EA_INSULATION / BOLTZMANN_K * 
                                   (1.0f / T_winding_K - 1.0f / T_REF_K));
  
  float L_grease = L_REF_GREASE * pow(0.5f, (T_bearing_F - 158.0f) / 18.0f) * 
                   (6000.0f / max(Alt_RPM, 100.0f));
  
  float temp_factor = 1.0f + 0.0025f * (T_brush_F - 150.0f);
  float L_brush = (L_REF_BRUSH * 6000.0f / max(Alt_RPM, 100.0f)) / max(temp_factor, 0.1f);
  
  // Accumulate damage over time
  float hours_elapsed = elapsedSeconds / 3600.0f;
  CumulativeInsulationDamage += hours_elapsed / L_insul;
  CumulativeGreaseDamage += hours_elapsed / L_grease;
  CumulativeBrushDamage += hours_elapsed / L_brush;
  
  // Calculate remaining life percentages
  InsulationLifePercent = (1.0f - CumulativeInsulationDamage) * 100.0f;
  GreaseLifePercent = (1.0f - CumulativeGreaseDamage) * 100.0f;
  BrushLifePercent = (1.0f - CumulativeBrushDamage) * 100.0f;
  
  // Constrain to 0-100% range
  InsulationLifePercent = constrain(InsulationLifePercent, 0.0f, 100.0f);
  GreaseLifePercent = constrain(GreaseLifePercent, 0.0f, 100.0f);
  BrushLifePercent = constrain(BrushLifePercent, 0.0f, 100.0f);
}
```

**Thermal Model Constants**:
- `L_REF_INSUL`: 100,000 hours at 100°C reference
- `L_REF_GREASE`: 40,000 hours at 158°F reference  
- `L_REF_BRUSH`: 5,000 hours at 6000 RPM, 150°F
- `EA_INSULATION`: 1.0 eV activation energy
- `BOLTZMANN_K`: 8.617×10⁻⁵ eV/K

## Console Logging and Debugging

### Message Queue System

Thread-safe circular buffer for system messages:

```cpp
// Fixed-size circular buffer for thread safety
struct ConsoleMessage {
  char message[128];      // Fixed size prevents dynamic allocation
  unsigned long timestamp;
};

#define CONSOLE_QUEUE_SIZE 10
ConsoleMessage consoleQueue[CONSOLE_QUEUE_SIZE];
volatile int consoleHead = 0;
volatile int consoleTail = 0;
volatile int consoleCount = 0;

void queueConsoleMessage(String message) {
  // Prevent stack overflow from oversized messages
  if (message.length() > 120) {
    message = message.substring(0, 120) + "...";
  }
  
  // Thread-safe circular buffer operation
  int nextHead = (consoleHead + 1) % CONSOLE_QUEUE_SIZE;
  int localTail = consoleTail;
  
  if (nextHead != localTail) {  // Not full
    strncpy(consoleQueue[consoleHead].message, message.c_str(), 127);
    consoleQueue[consoleHead].message[127] = '\0';
    consoleQueue[consoleHead].timestamp = millis();
    consoleHead = nextHead;
    
    if (consoleCount < CONSOLE_QUEUE_SIZE) {
      consoleCount++;
    }
  }
  // If full, oldest message is automatically overwritten
}
```

### Warning Throttling

Prevents console spam from repeated conditions:

```cpp
void updateSystemHealthMetrics() {
  unsigned long now = millis();
  
  // Critical heap warnings (throttled to 30 seconds)
  if (FreeHeap < 20 && (now - lastHeapWarningTime > WARNING_THROTTLE_INTERVAL)) {
    queueConsoleMessage("CRITICAL: Heap dangerously low (" + String(FreeHeap) + "KB)");
    lastHeapWarningTime = now;
  }
  
  // Stack warnings (throttled)
  if (stackBytes < 256 && (now - lastStackWarningTime > WARNING_THROTTLE_INTERVAL)) {
    queueConsoleMessage("CRITICAL: " + String(taskName) + " stack very low");
    lastStackWarningTime = now;
  }
}
```

## Configuration Parameters

### Safety Thresholds

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| TemperatureLimitF | 150 | 100-250 | Alternator temperature limit (°F) |
| VoltageAlarmHigh | 15 | 12-18 | High voltage alarm threshold |
| VoltageAlarmLow | 11 | 8-12 | Low voltage alarm threshold |
| CurrentAlarmHigh | 100 | 50-200 | High current alarm threshold |
| MaximumAllowedBatteryAmps | 100 | 50-500 | Battery current protection limit |

### System Health Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| WARNING_THROTTLE_INTERVAL | 30000 | 5000-300000 | Warning message throttle (ms) |
| DATA_TIMEOUT | 10000 | 1000-60000 | Sensor freshness timeout (ms) |
| FIELD_COLLAPSE_DELAY | 10000 | 5000-30000 | Emergency shutdown lockout (ms) |
| ALARM_TEST_DURATION | 2000 | 1000-10000 | Alarm test duration (ms) |

### Thermal Model Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| WindingTempOffset | 50.0 | 0-100 | Winding temperature offset (°F) |
| PulleyRatio | 2.0 | 1.0-4.0 | Engine to alternator speed ratio |
| THERMAL_UPDATE_INTERVAL | 10000 | 1000-60000 | Thermal calculation rate (ms) |

## Error Recovery Procedures

### Automatic Recovery Systems
- **Watchdog reset**: Complete system restart on hang
- **Sensor fallback**: Automatic switch to backup sensors
- **Field collapse recovery**: 10-second lockout with automatic resume
- **Memory management**: Garbage collection and stack monitoring

### Manual Recovery Options
- **Factory reset**: GPIO15 pin recovery mode
- **Alarm reset**: Web interface manual reset
- **Configuration reset**: Delete individual setting files
- **Complete restart**: 2-hour automatic maintenance restart

This comprehensive safety and monitoring system ensures reliable operation while providing detailed diagnostics and protection against all major failure modes.