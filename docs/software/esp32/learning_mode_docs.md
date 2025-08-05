# Learning Mode System

## Overview

The Learning Mode System represents a significant advancement in alternator control technology, implementing adaptive algorithms that automatically optimize alternator output based on real-world thermal performance. Unlike traditional fixed-setpoint regulators, this system learns from operating experience to maximize safe alternator output while preventing overheating.

## Innovation and Design Philosophy

### Problem Statement

Traditional alternator regulators use fixed current setpoints that must be conservatively set to prevent overheating under worst-case conditions. This approach results in:

- **Under-utilization**: Safe current often set 20-40% below actual capability
- **Temperature blindness**: No adaptation to actual thermal conditions
- **Manual tuning required**: Expensive marine service calls for optimization
- **Seasonal inefficiency**: Same settings for all weather conditions

### Learning Mode Solution

The Learning Mode System addresses these limitations through:

- **Adaptive current tables**: RPM-specific current limits that adjust based on experience
- **Thermal feedback**: Real-time temperature monitoring with overheat learning
- **PID control integration**: Precise current regulation with professional-grade control
- **Automatic optimization**: Gradual increases during safe operation
- **Long-term memory**: Persistent learning across power cycles and seasons

### Key Innovations

1. **Dynamic Current Tables**: 10-point RPM vs current lookup table that adapts over time
2. **Overheat Learning**: Automatic reduction of limits when thermal limits are exceeded
3. **Safe Operation Learning**: Gradual increases during extended safe operation
4. **Neighbor Learning**: Adjacent RPM points learn from each other's experience
5. **Penalty System**: Temporary conservative operation after overheats
6. **PID Integration**: Professional control algorithms for precise regulation

## System Architecture

### Core Components

```
Thermal Sensors → Overheat Detection → Learning Engine → RPM Current Tables → PID Controller → Field PWM
      ↓                    ↓                 ↓               ↓                  ↓             ↓
DS18B20/Thermistor → Temperature Limit → Table Updates → Lookup Current → Precise Control → MOSFET Drive
```

### Data Structures

#### RPM Current Table
```cpp
#define RPM_TABLE_SIZE 10
float rpmCurrentTable[RPM_TABLE_SIZE] = {0, 15, 25, 35, 40, 45, 50, 50, 45, 40};
int rpmTableRPMPoints[RPM_TABLE_SIZE] = {100, 600, 1100, 1600, 2100, 2600, 3100, 3600, 4100, 4600};
```

**Table Structure**:
- **10 RPM breakpoints**: Cover typical marine engine range 100-4600 RPM
- **Current values**: Start conservatively, adapt upward during safe operation
- **Persistent storage**: Saved to NVS, survives power cycles
- **Bounds checking**: Constrained to MinTableValue - MaxTableValue range

#### Learning History Arrays
```cpp
unsigned long lastOverheatTime[RPM_TABLE_SIZE] = {0};       // Timestamp of last overheat per RPM
int overheatCount[RPM_TABLE_SIZE] = {0};                    // Total overheat events per RPM
unsigned long cumulativeNoOverheatTime[RPM_TABLE_SIZE] = {0}; // Safe operation time per RPM
```

**Historical Data**:
- **Per-RPM tracking**: Independent learning for each operating point
- **Long-term memory**: Events remembered across sessions
- **Statistical analysis**: Cumulative safe operation time tracking
- **Diagnostic information**: Overheat frequency and patterns

#### Learning State Variables
```cpp
unsigned long overheatingPenaltyTimer = 0;   // Remaining penalty time (ms)
float overheatingPenaltyAmps = 0.0;          // Current penalty amount (A)
int currentRPMTableIndex = -1;               // Active table entry
unsigned long learningSessionStartTime = 0; // Session start for current RPM
unsigned long lastTableUpdateTime = 0;      // Throttling for table updates
```

## Learning Algorithms

### RPM Table Lookup

The system determines current limits based on engine RPM:

```cpp
void getLearningTargetFromRPM() {
  currentRPMTableIndex = -1;
  
  // Find RPM range
  for (int i = 0; i < RPM_TABLE_SIZE - 1; i++) {
    if (RPM >= rpmTableRPMPoints[i] && RPM < rpmTableRPMPoints[i + 1]) {
      currentRPMTableIndex = i;
      break;
    }
  }
  
  // Handle edge case: RPM above highest table point
  if (RPM >= rpmTableRPMPoints[RPM_TABLE_SIZE - 1]) {
    currentRPMTableIndex = RPM_TABLE_SIZE - 1;
  }
  
  // Set target from table
  if (currentRPMTableIndex >= 0 && currentRPMTableIndex < RPM_TABLE_SIZE) {
    learningTargetFromRPM = rpmCurrentTable[currentRPMTableIndex];
  } else {
    learningTargetFromRPM = -1;  // Invalid RPM range
  }
}
```

**Lookup Strategy**:
- **Linear ranges**: Simple RPM ranges for reliable operation
- **Edge handling**: Proper behavior at table boundaries
- **Index tracking**: Global variable for learning algorithm access
- **Error indication**: -1 value for out-of-range conditions

### Target Current Calculation

Learning Mode modifies the standard target calculation:

```cpp
// Step 1: Base target from Hi/Low setting
if (HiLow == 1) {
  uTargetAmps = TargetAmps;
} else {
  uTargetAmps = TargetAmpL;
}

// Step 2: Apply standard RPM curve if enabled
if (AmpControlByRPM == 1 && RPM > 100 && RPM < 6000) {
  int rpmBasedAmps = interpolateAmpsFromRPM(RPM);
  uTargetAmps = (HiLow == 0) ? min(rpmBasedAmps, TargetAmpL) : rpmBasedAmps;
}

// Step 3: Apply learning table limitation
getLearningTargetFromRPM();
if (learningTargetFromRPM > 0) {
  uTargetAmps = min((float)uTargetAmps, learningTargetFromRPM);
}

// Step 4: Ambient temperature correction
if (EnableAmbientCorrection == 1) {
  ambientTempCorrection = (ambientTemp - AmbientTempBaseline) * AmbientTempCorrectionFactor;
  uTargetAmps += ambientTempCorrection;
} else {
  ambientTempCorrection = 0;
}

// Step 5: Apply overheat penalty
if (overheatingPenaltyTimer > 0) {
  uTargetAmps -= overheatingPenaltyAmps;
  overheatingPenaltyTimer -= FieldAdjustmentInterval;
  if (overheatingPenaltyTimer <= 0) {
    overheatingPenaltyTimer = 0;
    overheatingPenaltyAmps = 0;
    queueConsoleMessage("Learning: Overheat penalty expired");
  }
}

finalLearningTarget = uTargetAmps;  // Store for display
```

**Multi-layered Approach**:
- **Standard limits**: Normal Hi/Low and RPM curves still apply
- **Learning constraint**: Table values provide upper bounds
- **Environmental correction**: Temperature compensation
- **Penalty system**: Temporary conservative operation

### Overheat Detection and Learning

When temperature exceeds limits, the system learns and adapts:

```cpp
void processOverheatLearning() {
  if (TempToUse > TemperatureLimitF) {
    if (!LearningDownwardEnabled) return;
    
    // Record overheat event
    lastOverheatTime[currentRPMTableIndex] = millis();
    overheatCount[currentRPMTableIndex]++;
    totalOverheats++;
    totalLearningEvents++;
    
    // Apply learning reduction to current point
    rpmCurrentTable[currentRPMTableIndex] -= LearningDownStep;
    
    // Apply neighbor reductions if enabled
    if (EnableNeighborLearning) {
      float neighborReduction = LearningDownStep * NeighborLearningFactor;
      if (currentRPMTableIndex > 0) {
        rpmCurrentTable[currentRPMTableIndex - 1] -= neighborReduction;
      }
      if (currentRPMTableIndex < RPM_TABLE_SIZE - 1) {
        rpmCurrentTable[currentRPMTableIndex + 1] -= neighborReduction;
      }
    }
    
    // Start overheat penalty timer
    overheatingPenaltyTimer = MaxPenaltyDuration;
    overheatingPenaltyAmps = AlternatorNominalAmps * (MaxPenaltyPercent / 100.0);
    
    // Bound table values
    for (int i = 0; i < RPM_TABLE_SIZE; i++) {
      rpmCurrentTable[i] = constrain(rpmCurrentTable[i], MinTableValue, MaxTableValue);
    }
    
    lastTableUpdateTime = millis();
    
    if (AutoSaveLearningTable) {
      saveLearningTableToNVS();
    }
    
    if (ShowLearningDebugMessages) {
      queueConsoleMessage("Learning: Overheat at " + String(rpmTableRPMPoints[currentRPMTableIndex]) + 
                         " RPM, reduced to " + String(rpmCurrentTable[currentRPMTableIndex], 1) + "A");
    }
  }
}
```

**Overheat Response Strategy**:
- **Immediate reduction**: Current RPM point reduced by LearningDownStep
- **Neighbor learning**: Adjacent RPM points also reduced (configurable)
- **Penalty period**: Temporary additional reduction for safety
- **Persistent memory**: Event recorded for long-term analysis
- **Bounds enforcement**: Table values kept within safe ranges

### Safe Operation Learning

During extended safe operation, the system gradually increases limits:

```cpp
void processSafeOperationLearning() {
  if (LearningUpwardEnabled && sessionTime > SafeOperationThreshold) {
    // Add to cumulative safe operation time
    cumulativeNoOverheatTime[currentRPMTableIndex] += sessionTime;
    totalSafeHours += sessionTime / 3600000;  // Convert ms to hours
    
    // Every SafeOperationThreshold of cumulative safe operation, increase by UpStep
    unsigned long safeIntervals = cumulativeNoOverheatTime[currentRPMTableIndex] / SafeOperationThreshold;
    if (safeIntervals > 0) {
      float increase = safeIntervals * LearningUpStep;
      rpmCurrentTable[currentRPMTableIndex] += increase;
      totalLearningEvents++;
      
      // Reset cumulative timer (we've "used up" this safe time)
      cumulativeNoOverheatTime[currentRPMTableIndex] = 0;
      
      // Bound to reasonable limits
      rpmCurrentTable[currentRPMTableIndex] = constrain(rpmCurrentTable[currentRPMTableIndex], 
                                                       MinTableValue, MaxTableValue);
      
      lastTableUpdateTime = millis();
      
      if (AutoSaveLearningTable) {
        saveLearningTableToNVS();
      }
      
      if (ShowLearningDebugMessages) {
        queueConsoleMessage("Learning: Safe operation at " + String(rpmTableRPMPoints[currentRPMTableIndex]) + 
                           " RPM, increased to " + String(rpmCurrentTable[currentRPMTableIndex], 1) + "A");
      }
    }
  }
}
```

**Safe Learning Strategy**:
- **Time threshold**: Must operate safely for SafeOperationThreshold (default 30 seconds)
- **Cumulative tracking**: Builds up safe operation credit over time
- **Gradual increases**: Small increments (LearningUpStep) for safety
- **Credit consumption**: Safe time is "spent" when used for increases
- **Conservative approach**: Much slower increases than decreases

### Penalty System

After overheating, the system applies temporary additional current reduction:

```cpp
// Initialize penalty
overheatingPenaltyTimer = MaxPenaltyDuration;  // Default: 60 seconds
overheatingPenaltyAmps = AlternatorNominalAmps * (MaxPenaltyPercent / 100.0);  // Default: 15% of nominal

// Apply penalty in target calculation
if (overheatingPenaltyTimer > 0) {
  uTargetAmps -= overheatingPenaltyAmps;  // Additional reduction
  overheatingPenaltyTimer -= FieldAdjustmentInterval;  // Count down
}
```

**Penalty Characteristics**:
- **Duration**: 60 seconds default, configurable
- **Magnitude**: 15% of alternator nominal rating default
- **Purpose**: Ensures cooling period after overheat
- **Automatic expiry**: Self-clearing after timeout

## PID Control Integration

### PID Controller Setup

Learning Mode uses professional PID control for precise current regulation:

```cpp
#include <PID_v1.h>

// PID variables
double pidInput = 0.0;      // Current measured amps
double pidOutput = 0.0;     // PID duty cycle output  
double pidSetpoint = 0.0;   // Current target amps
PID currentPID(&pidInput, &pidOutput, &pidSetpoint, PidKp, PidKi, PidKd, DIRECT);

// PID initialization
if (!pidInitialized) {
  currentPID.SetOutputLimits(MinDuty, MaxDuty);
  currentPID.SetSampleTime(PidSampleTime);  // Default: 100ms
  currentPID.SetTunings(PidKp, PidKi, PidKd);  // Default: 0.5, 0.0, 0.0
  currentPID.SetMode(AUTOMATIC);
  pidInitialized = true;
}
```

### PID Control Loop

```cpp
// Set PID inputs
pidInput = targetCurrent;           // Measured current (alternator or battery)
pidSetpoint = uTargetAmps;          // Learning target (after all modifications)

// Execute PID calculation
currentPID.Compute();

// Use PID output as duty cycle
dutyCycle = pidOutput;
```

**PID Benefits Over Simple Proportional**:
- **Precise regulation**: Eliminates steady-state error with integral term
- **Smooth response**: Derivative term reduces overshoot
- **Professional control**: Industry-standard algorithm
- **Tunable performance**: Adjustable for different alternator characteristics

### PID Tuning Guidelines

#### Default Settings
```cpp
float PidKp = 0.5;    // Proportional gain
float PidKi = 0.0;    // Integral gain (start with 0)
float PidKd = 0.0;    // Derivative gain (start with 0)
unsigned long PidSampleTime = 100;  // 100ms update rate
```

#### Tuning Process
1. **Start with P-only**: Set Ki=0, Kd=0, tune Kp for stable response
2. **Add integral**: Gradually increase Ki to eliminate steady-state error
3. **Add derivative**: Small Kd values to reduce overshoot if needed
4. **Sample time**: Match to system response characteristics

#### Performance Characteristics
- **Proportional gain**: Higher values = faster response, risk of instability
- **Integral gain**: Eliminates offset, can cause overshoot if too high
- **Derivative gain**: Reduces overshoot, sensitive to noise
- **Sample time**: Should match dominant time constants (thermal ~10-30s)

## Configuration Parameters

### Learning Control Switches

| Parameter | Default | Description |
|-----------|---------|-------------|
| LearningMode | 0 | 0=disabled, 1=enabled |
| LearningPaused | 0 | Temporarily pause learning |
| LearningUpwardEnabled | 1 | Allow upward table adjustments |
| LearningDownwardEnabled | 1 | Allow downward table adjustments |
| IgnoreLearningDuringPenalty | 1 | Block learning during penalty period |
| EnableNeighborLearning | 1 | Update adjacent RPM points |
| EnableAmbientCorrection | 0 | Apply temperature correction |
| ShowLearningDebugMessages | 1 | Verbose console output |

### Learning Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| AlternatorNominalAmps | 100 | 50-500 | Alternator rating for penalty calculation |
| LearningUpStep | 1.0 | 0.1-10.0 | Table increase amount (A) |
| LearningDownStep | 2.0 | 0.1-10.0 | Table decrease amount (A) |
| AmbientTempCorrectionFactor | -0.5 | -5.0-5.0 | A per °F correction |
| AmbientTempBaseline | 75.0 | 32-120 | Baseline temp (°F) |
| MinLearningInterval | 30000 | 5000-300000 | Min time between updates (ms) |
| SafeOperationThreshold | 30000 | 10000-300000 | Time for upward learning (ms) |

### PID Tuning Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| PidKp | 0.5 | 0.0-10.0 | Proportional gain |
| PidKi | 0.0 | 0.0-1.0 | Integral gain |
| PidKd | 0.0 | 0.0-1.0 | Derivative gain |
| PidSampleTime | 100 | 50-1000 | PID update interval (ms) |

### Table Bounds & Safety

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| MaxTableValue | 120.0 | 10-500 | Maximum table entry (A) |
| MinTableValue | 0.0 | 0-50 | Minimum table entry (A) |
| MaxPenaltyPercent | 15.0 | 0-50 | Max penalty as % of nominal |
| MaxPenaltyDuration | 60000 | 10000-300000 | Max penalty time (ms) |
| NeighborLearningFactor | 0.25 | 0.0-1.0 | Neighbor reduction factor |

## Data Management

### Persistent Storage

Learning data is stored in NVS (Non-Volatile Storage) for persistence across power cycles:

```cpp
void saveLearningTableToNVS() {
  nvs_handle_t nvs_handle;
  esp_err_t err = nvs_open("learning", NVS_READWRITE, &nvs_handle);
  
  // Save table data
  nvs_set_blob(nvs_handle, "rpmTable", rpmCurrentTable, sizeof(rpmCurrentTable));
  nvs_set_blob(nvs_handle, "overheatCount", overheatCount, sizeof(overheatCount));
  nvs_set_blob(nvs_handle, "lastOverheat", lastOverheatTime, sizeof(lastOverheatTime));
  nvs_set_blob(nvs_handle, "cumulativeTime", cumulativeNoOverheatTime, sizeof(cumulativeNoOverheatTime));
  
  // Save diagnostic counters
  nvs_set_u32(nvs_handle, "totalEvents", (uint32_t)totalLearningEvents);
  nvs_set_u32(nvs_handle, "totalOverheats", (uint32_t)totalOverheats);
  nvs_set_u32(nvs_handle, "totalSafeHours", (uint32_t)totalSafeHours);
  
  nvs_commit(nvs_handle);
  nvs_close(nvs_handle);
}
```

**Storage Benefits**:
- **Power cycle survival**: Learning persists through restarts
- **Seasonal memory**: Remembers previous summer/winter performance
- **Long-term optimization**: Improves over months/years of operation
- **Factory reset capability**: Can restore defaults when needed

### Auto-Save System

```cpp
// Throttled automatic saving
static unsigned long lastSaveTime = 0;
if (millis() - lastSaveTime < LearningTableSaveInterval) {
  return;  // Too soon, skip save
}
lastSaveTime = millis();
saveLearningTableToNVS();
```

**Save Strategy**:
- **Throttled saves**: Maximum every 5 minutes to reduce NVS wear
- **Event-triggered**: Saves after significant learning events
- **User control**: AutoSaveLearningTable enables/disables automatic saving
- **Manual trigger**: ResetLearningTable button forces immediate save

## Learning Diagnostics

### Real-time Statistics

The system calculates diagnostic information for monitoring learning progress:

```cpp
void calculateLearningDiagnostics() {
  // Calculate average table value
  float sum = 0;
  for (int i = 0; i < RPM_TABLE_SIZE; i++) {
    sum += rpmCurrentTable[i];
  }
  averageTableValue = sum / RPM_TABLE_SIZE;
  
  // Find time since last overheat
  timeSinceLastOverheat = 0;
  unsigned long mostRecentOverheat = 0;
  for (int i = 0; i < RPM_TABLE_SIZE; i++) {
    if (lastOverheatTime[i] > mostRecentOverheat) {
      mostRecentOverheat = lastOverheatTime[i];
    }
  }
  if (mostRecentOverheat > 0) {
    timeSinceLastOverheat = millis() - mostRecentOverheat;
  }
}
```

### Web Interface Display

Learning diagnostics are transmitted to the web interface:

```cpp
// Real-time learning status (CSVData3)
SafeInt(currentRPMTableIndex),                   // Active table entry
SafeInt(pidInitialized ? 1 : 0),                 // PID status
SafeInt(overheatCount[0]),                       // Per-RPM overheat counts
SafeInt(overheatCount[1]),
// ... all 10 RPM points
SafeInt(cumulativeNoOverheatTime[0] / 3600000),  // Safe hours per RPM
SafeInt(cumulativeNoOverheatTime[1] / 3600000),
// ... all 10 RPM points
SafeInt(totalLearningEvents),                    // Overall statistics
SafeInt(totalOverheats),
SafeInt(totalSafeHours),
SafeInt(averageTableValue * 100),
SafeInt(timeSinceLastOverheat / 1000),
SafeInt(learningTargetFromRPM, 100),             // Current learning values
SafeInt(ambientTempCorrection, 100),
SafeInt(finalLearningTarget, 100),
SafeInt(overheatingPenaltyTimer / 1000),
SafeInt(overheatingPenaltyAmps, 100)
```

**Dashboard Information**:
- **Current status**: Active RPM range, current target, penalty status
- **Historical data**: Overheat counts, safe operation hours per RPM
- **Learning progress**: Total events, average table values, recent changes
- **Performance**: Time since last overheat, penalty remaining

## Advanced Features

### Neighbor Learning

When an overheat occurs, adjacent RPM points are also reduced:

```cpp
if (EnableNeighborLearning) {
  float neighborReduction = LearningDownStep * NeighborLearningFactor;  // Default: 2.0 * 0.25 = 0.5A
  if (currentRPMTableIndex > 0) {
    rpmCurrentTable[currentRPMTableIndex - 1] -= neighborReduction;
  }
  if (currentRPMTableIndex < RPM_TABLE_SIZE - 1) {
    rpmCurrentTable[currentRPMTableIndex + 1] -= neighborReduction;
  }
}
```

**Benefits**:
- **Thermal coupling**: Adjacent RPM ranges often have similar thermal characteristics
- **Smoother curves**: Prevents sharp discontinuities in current vs RPM
- **Conservative approach**: Reduces risk of overheating at neighboring points
- **Faster learning**: Spreads experience across related operating points

### Ambient Temperature Correction

Compensates current limits based on ambient temperature:

```cpp
if (EnableAmbientCorrection == 1) {
  ambientTempCorrection = (ambientTemp - AmbientTempBaseline) * AmbientTempCorrectionFactor;
  uTargetAmps += ambientTempCorrection;
}
```

**Example**:
- AmbientTempBaseline = 75°F
- AmbientTempCorrectionFactor = -0.5 A/°F
- Current ambient = 95°F
- Correction = (95 - 75) * (-0.5) = -10A reduction

**Seasonal Adaptation**:
- **Summer operation**: Reduced current limits in hot weather
- **Winter operation**: Increased limits in cold weather
- **Automatic adjustment**: No manual intervention required
- **User configurable**: Correction factor adjustable for different installations

### Learning Memory Duration

Old overheat events eventually expire:

```cpp
unsigned long LearningMemoryDuration = 2592000000;  // 30 days in milliseconds

// Check for expired events during learning updates
if (millis() - lastOverheatTime[currentRPMTableIndex] > LearningMemoryDuration) {
  // Consider gradual forgetting or event expiry
}
```

**Memory Benefits**:
- **Forgiveness**: System eventually forgets old overheats
- **Seasonal adaptation**: Summer overheats don't permanently limit winter operation
- **Progressive optimization**: Allows gradual recovery from conservative settings
- **Configurable duration**: Adjustable based on operational patterns

### Dry Run Mode

Test learning algorithms without modifying actual table values:

```cpp
if (LearningDryRunMode == 1) {
  // Calculate what changes would be made
  float proposedChange = LearningDownStep;
  queueConsoleMessage("DRY RUN: Would reduce " + String(rpmTableRPMPoints[currentRPMTableIndex]) + 
                     " RPM by " + String(proposedChange, 1) + "A");
  // Don't actually modify rpmCurrentTable[]
  return;
}
```

**Testing Benefits**:
- **Safe experimentation**: Test learning behavior without risk
- **Algorithm verification**: Ensure learning logic works as expected
- **Training tool**: Understand learning behavior before enabling
- **Development aid**: Debug learning algorithms safely

## Performance Analysis

### Learning Rate Characteristics

| Scenario | Time to Change | Magnitude | Recovery Time |
|----------|----------------|-----------|---------------|
| Single overheat | Immediate | -2.0A (default) | 30 seconds penalty |
| Neighbor reduction | Immediate | -0.5A (default) | Gradual recovery |
| Safe operation learning | 30+ seconds | +1.0A (default) | Permanent until overheat |
| Penalty expiry | 60 seconds | Return to normal | N/A |

### Convergence Analysis

**Conservative start**: Default table starts 20-30% below typical alternator capability
**Learning speed**: 
- Downward: Fast (immediate on overheat)
- Upward: Slow (30+ seconds safe operation required)
**Steady state**: Converges to maximum safe current for each RPM range
**Stability**: Once learned, stable operation until conditions change significantly

### Memory Requirements

| Component | Memory Usage | Storage |
|-----------|--------------|---------|
| RPM Current Table | 40 bytes | NVS persistent |
| Overheat History | 40 bytes | NVS persistent |
| Safe Time Tracking | 40 bytes | NVS persistent |
| PID Variables | 24 bytes | RAM only |
| Learning State | 32 bytes | RAM only |
| **Total** | **176 bytes** | **120 bytes persistent** |

## Implementation Guide

### Enabling Learning Mode

1. **Initial Setup**:
```cpp
LearningMode = 1;                    // Enable learning
LearningPaused = 0;                  // Ensure not paused
AutoSaveLearningTable = 1;           // Enable auto-save
ShowLearningDebugMessages = 1;       // Enable diagnostics
```

2. **Basic Configuration**:
```cpp
AlternatorNominalAmps = 100;         // Set to actual alternator rating
TemperatureLimitF = 150;             // Conservative temperature limit
LearningUpStep = 1.0;               // Conservative increase rate
LearningDownStep = 2.0;             // Aggressive decrease rate
```

3. **PID Tuning Start**:
```cpp
PidKp = 0.5;                        // Start with proportional only
PidKi = 0.0;                        // Add integral later
PidKd = 0.0;                        // Add derivative if needed
```

### Commissioning Process

#### Phase 1: Baseline Establishment (First 100 hours)
- Start with conservative table values
- Enable all learning features
- Monitor for overheats and responses
- Verify temperature sensors accurate
- Check PID stability

#### Phase 2: Optimization (100-500 hours)
- Adjust learning parameters based on observed behavior
- Fine-tune PID coefficients if needed
- Verify neighbor learning effectiveness
- Monitor long-term trends

#### Phase 3: Mature Operation (500+ hours)
- System should be well-optimized
- Primarily maintenance monitoring
- Seasonal adaptation should be automatic
- Occasional parameter adjustments for major changes

### Troubleshooting Learning Mode

#### Problem: No Learning Occurring
```
Check:
1. LearningMode = 1
2. LearningPaused = 0  
3. Valid RPM range (100-6000)
4. Temperature sensor working
5. MinLearningInterval not too large
```

#### Problem: Overly Conservative Operation
```
Check:
1. LearningUpStep too small
2. SafeOperationThreshold too large
3. Recent overheats preventing increases
4. Penalty system still active
5. Neighbor learning over-correcting
```

#### Problem: Frequent Overheating
```
Check:
1. LearningDownStep too small
2. Table MaxTableValue too high
3. Alternator installation/cooling issues
4. Temperature sensor calibration
5. Penalty duration too short
```

#### Problem: PID Instability
```
Check:
1. PidKp too high (reduce)
2. PidSampleTime too fast
3. Sensor noise affecting feedback
4. Multiple control loops interfering
5. Derivative gain causing oscillation
```

### Performance Optimization

#### Aggressive Learning Setup
For faster optimization in controlled environments:
```cpp
LearningUpStep = 2.0;               // Faster increases
SafeOperationThreshold = 15000;     // Shorter safe periods
LearningDownStep = 3.0;             // More aggressive reductions
MaxPenaltyDuration = 30000;         // Shorter penalties
```

#### Conservative Learning Setup
For critical applications requiring maximum safety:
```cpp
LearningUpStep = 0.5;               // Slower increases  
SafeOperationThreshold = 60000;     // Longer safe periods required
LearningDownStep = 5.0;             // Very aggressive reductions
MaxPenaltyDuration = 120000;        // Longer penalties
NeighborLearningFactor = 0.5;       // More neighbor impact
```

The Learning Mode System represents a significant advancement in marine alternator control, providing automatic optimization that improves over time while maintaining safety as the highest priority. This intelligent system adapts to real-world conditions, maximizing alternator output while preventing thermal damage through advanced algorithms and machine learning principles.