# Learning Mode System

## Overview
The Learning Mode System implements adaptive control for marine alternator regulation, automatically optimizing current output based on real-world thermal performance. Unlike traditional fixed-setpoint regulators, this system uses thermal feedback to learn safe operating limits for each RPM range.

## Core Problem
Traditional alternator regulators use conservative fixed current limits to prevent overheating under worst-case conditions. This results in:

- 20–40% under-utilization of alternator capacity
- No adaptation to actual thermal conditions
- Manual tuning requirements
- Poor seasonal efficiency

## Learning Mode Solution
### Key Components
- 10-point RPM vs current lookup table that adapts over time
- Automatic reduction when thermal limits exceeded
- Gradual increases during extended safe operation
- PID control integration for precise regulation
- Persistent learning across power cycles

## System Architecture
```
Temperature Sensors → Overheat Detection → Learning Engine → RPM Current Tables → PID Controller → Field PWM
```

## Data Structures
```cpp
#define RPM_TABLE_SIZE 10
float rpmCurrentTable[RPM_TABLE_SIZE] = {0, 15, 25, 35, 40, 45, 50, 50, 45, 40};
int rpmTableRPMPoints[RPM_TABLE_SIZE] = {100, 600, 1100, 1600, 2100, 2600, 3100, 3600, 4100, 4600};
unsigned long lastOverheatTime[RPM_TABLE_SIZE] = {0};
int overheatCount[RPM_TABLE_SIZE] = {0};
unsigned long cumulativeNoOverheatTime[RPM_TABLE_SIZE] = {0};
```

## Learning Algorithms

### RPM Table Lookup
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

  if (RPM >= rpmTableRPMPoints[RPM_TABLE_SIZE - 1]) {
    currentRPMTableIndex = RPM_TABLE_SIZE - 1;
  }

  if (currentRPMTableIndex >= 0 && currentRPMTableIndex < RPM_TABLE_SIZE) {
    learningTargetFromRPM = rpmCurrentTable[currentRPMTableIndex];
  } else {
    learningTargetFromRPM = -1;
  }
}
```

### Target Current Calculation
```cpp
// Step 3: Apply learning table limitation
getLearningTargetFromRPM();
if (learningTargetFromRPM > 0) {
  uTargetAmps = min((float)uTargetAmps, learningTargetFromRPM);
}

// Step 5: Apply overheat penalty
if (overheatingPenaltyTimer > 0) {
  uTargetAmps -= overheatingPenaltyAmps;
  overheatingPenaltyTimer -= FieldAdjustmentInterval;
}
```

### Overheat Learning
```cpp
// Record overheat event
lastOverheatTime[currentRPMTableIndex] = millis();
overheatCount[currentRPMTableIndex]++;
totalOverheats++;

// Reduce current table entry
rpmCurrentTable[currentRPMTableIndex] -= LearningDownStep;

// Reduce neighbor points if enabled
if (EnableNeighborLearning) {
  float neighborReduction = LearningDownStep * NeighborLearningFactor;
  if (currentRPMTableIndex > 0) {
    rpmCurrentTable[currentRPMTableIndex - 1] -= neighborReduction;
  }
  if (currentRPMTableIndex < RPM_TABLE_SIZE - 1) {
    rpmCurrentTable[currentRPMTableIndex + 1] -= neighborReduction;
  }
}

// Start penalty timer
overheatingPenaltyTimer = MaxPenaltyDuration;
overheatingPenaltyAmps = AlternatorNominalAmps * (MaxPenaltyPercent / 100.0);
```

...

(The markdown file is truncated here for brevity.)