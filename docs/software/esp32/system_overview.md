# System Overview

## Purpose & Design Philosophy

The Xengineering Alternator Regulator controls alternator field output using ESP32-based PWM control with multi-source sensor inputs. The system provides battery monitoring, web-based configuration, and adaptive learning algorithms for thermal management.

### Core Design Principles

- **Safety**: Multiple protection layers prevent alternator, battery, and electrical system damage
- **Learning Mode**: Adaptive alternator control based on temperature feedback and overheat history
- **Real-time Interface**: Web-based monitoring and control accessible from any device
- **Reliability**: Watchdog protection, error handling, and automatic recovery systems
- **Open Source**: Full transparency for modification and community development

## System Architecture

### Hardware Foundation
```
ESP32-S3 (16MB Flash) ← Main Controller
├── ADS1115 ← 4-channel analog input (battery V, alt current, RPM, temp)
├── INA228 ← High-precision battery monitor with hardware overvoltage protection
├── OneWire ← DS18B20 temperature sensors
├── Field MOSFET Driver ← PWM control of alternator field
├── Digital I/O ← Ignition, alarms, switches, BMS signals
├── Communication Interfaces ← NMEA2K, NMEA0183, Victron VE.Direct
└── WiFi ← Web interface and OTA updates
```

### Dual-Core Software Architecture

The ESP32-S3 dual-core architecture separates critical functions for optimal performance and reliability:

**Core 1 (Application Core)**:
- Main control loop with field control algorithms
- Sensor data acquisition (ADS1115, INA228)
- Safety monitoring and alarm systems
- Web interface data transmission
- Watchdog monitoring (15-second timeout)

**Core 0 (Protocol Core)**:
- WiFi and network communication
- TempTask for DS18B20 temperature reading
- System background tasks
- Independent health monitoring

### TempTask Architecture

Temperature reading requires special handling due to DS18B20 timing constraints:

```cpp
void TempTask(void *parameter) {
  // Runs independently on Core 0
  for (;;) {
    // Update heartbeat to show task is alive
    lastTempTaskHeartbeat = millis();
    
    // Only read temperature every 10 seconds
    if (millis() - lastTempRead < 10000) {
      vTaskDelay(pdMS_TO_TICKS(1000));
      continue;
    }
    
    // Trigger conversion
    sensors.requestTemperaturesByAddress(tempDeviceAddress);
    
    // Wait 5 seconds for conversion (cannot be interrupted)
    for (int i = 0; i < 25; i++) {
      vTaskDelay(pdMS_TO_TICKS(200));
      lastTempTaskHeartbeat = millis();  // Update heartbeat during wait
    }
    
    // Read and validate result
    if (sensors.readScratchPad(tempDeviceAddress, scratchPad)) {
      // Process temperature data...
      MARK_FRESH(IDX_ALTERNATOR_TEMP);
    }
    
    lastTempRead = millis();
  }
}
```

**Why Separate Task?**
- **5-second blocking operation**: DS18B20 conversion time would freeze main loop
- **Core isolation**: Temperature reading doesn't interfere with field control
- **Independent monitoring**: Separate health checking system
- **Continuous operation**: Runs while main loop handles critical control functions

## Main Loop Structure

### Setup Phase (`setup()`)
The initialization sequence establishes all system components:

```cpp
void setup() {
  Serial.begin(115200);
  
  // System initialization
  initializeNVS();                    // Non-volatile storage for persistent data
  loadNVSData();                      // Restore previous session data
  ensureLittleFS();                   // File system for settings and web files
  InitSystemSettings();               // Load all configuration from files
  
  // Hardware initialization  
  if (hardwarePresent == 1) {
    initializeHardware();             // Sensors, communication interfaces
  }
  
  // Create TempTask on Core 0
  xTaskCreatePinnedToCore(TempTask, "TempTask", 4096, NULL, 0, &tempTaskHandle, 0);
  
  // Network and security
  loadPasswordHash();                 // Web interface authentication
  setupWiFi();                        // Network connectivity
  
  // Safety systems
  setupWatchdog();                    // 15-second hang detection
  
  Serial.println("=== SETUP COMPLETE ===");
}
```

**Key Setup Functions**:
- `initializeNVS()`: Persistent data storage initialization
- `initializeHardware()`: Sensor and communication interfaces  
- `setupWiFi()`: Network configuration and connection
- `InitSystemSettings()`: Load settings from LittleFS files
- `setupWatchdog()`: Configure system hang protection

### Watchdog Protection System

The watchdog system provides critical safety protection against software hangs:

**What is a Watchdog?**
A watchdog timer is a hardware-based safety mechanism that monitors software execution:
- **15-second timeout**: If the main loop doesn't "feed" the watchdog, the ESP32 automatically reboots
- **Safety purpose**: During reboot, all GPIO pins reset to LOW, which immediately turns OFF the alternator field
- **Hang detection**: Prevents infinite loops, memory corruption, or other failures from leaving the field energized
- **Hardware independence**: Operates independently of software, providing ultimate fail-safe protection

**Watchdog Setup**:
```cpp
void setupWatchdog() {
  // Try to add main task to existing watchdog (Arduino framework often pre-initializes)
  esp_err_t add_result = esp_task_wdt_add(NULL);
  if (add_result == ESP_OK) {
    queueConsoleMessage("Watchdog already active - main task added");
  } else {
    // Create new watchdog if none exists
    esp_task_wdt_config_t wdt_config = {
      .timeout_ms = 15000,   // 15 seconds
      .idle_core_mask = 0,   // Don't monitor idle cores
      .trigger_panic = true  // Reboot on timeout
    };
    esp_task_wdt_init(&wdt_config);
    esp_task_wdt_add(NULL);
    queueConsoleMessage("Watchdog created and main task added");
  }
}
```

**Task Coverage**:
- **Main loop (Core 1)**: Monitored by ESP32 watchdog timer
- **TempTask (Core 0)**: Independent heartbeat monitoring system
- **System tasks**: Background tasks monitored by FreeRTOS

### Main Loop Operation (`loop()`)

The main control loop executes every 50-100ms with strict timing control:

```cpp
void loop() {
  esp_task_wdt_reset();              // Feed 15-second watchdog
  starttime = esp_timer_get_time();  // Performance timing
  
  // Core sensor reading
  ReadAnalogInputs();                // ADS1115 state machine
  if (VeData == 1) ReadVEData();     // Victron VE.Direct
  if (NMEA2KData == 1) NMEA2000.ParseMessages();
  
  // Battery and engine monitoring
  UpdateEngineRuntime(elapsedMillis);
  UpdateBatterySOC(elapsedMillis);
  
  // Field control (main function)
  if (LearningMode == 1) {
    AdjustFieldLearnMode();          // Adaptive control algorithm
  } else {
    AdjustField();                   // Standard control algorithm  
  }
  
  // Safety and health monitoring
  CheckAlarms();                     // Temperature, voltage, current limits
  checkTempTaskHealth();             // TempTask monitoring
  calculateThermalStress();          // Alternator lifetime modeling
  updateSystemHealthMetrics();       // CPU, memory, stack monitoring
  
  // User interface and data logging
  SendWifiData();                    // Real-time web interface updates
  UpdateDisplay();                   // Local OLED display
  
  // Periodic maintenance
  checkAndRestart();                 // Scheduled system restart (2 hours)
  
  // Performance monitoring
  endtime = esp_timer_get_time();
  LoopTime = (endtime - starttime);
  if (LoopTime > MaxLoopTime) MaxLoopTime = LoopTime;
}
```

**Loop Timing**:
- Target: 50-100ms per iteration
- Watchdog timeout: 15 seconds
- Performance monitoring: Track maximum loop time
- Automatic restart: Every 2 hours for maintenance

## Data Flow Architecture

### Sensor Data Pipeline
```
Hardware Sensors → ESP32 GPIO/I2C → Data Validation → Global Variables → Web Interface
     ↓                ↓                    ↓               ↓              ↓
ADS1115/INA228 → ReadAnalogInputs() → Sanity Checks → MeasuredAmps → SendWifiData()
OneWire Temp   → TempTask()         → Range Limits  → AlternatorTemperatureF
NMEA2K/VE.Direct → Parse Functions → Protocol Valid → VictronVoltage
```

### Control Flow
```
Sensor Inputs → Target Calculation → Field Control → PWM Output → Alternator Response
     ↓              ↓                    ↓              ↓              ↓
Battery V,I → uTargetAmps (Hi/Low/RPM) → dutyCycle → setDutyPercent() → Field Current
Temperature → Safety Overrides        → Constraints → MOSFET Driver → Heat Generation
```

### Data Freshness System
The system tracks sensor data age to detect failed sensors:

```cpp
// Data timestamp tracking
enum DataIndex {
  IDX_ALTERNATOR_TEMP = 0,
  IDX_BATTERY_V,
  IDX_MEASURED_AMPS,
  // ... 17 total indices
};
unsigned long dataTimestamps[MAX_DATA_INDICES];

// Mark data fresh when successfully read
#define MARK_FRESH(index) dataTimestamps[index] = millis()

// Check if data is stale (>10 seconds old)
#define IS_STALE(index) (millis() - dataTimestamps[index] > DATA_TIMEOUT)
```

**Benefits**:
- Web interface shows red indicators for stale data
- Prevents control decisions based on old sensor readings
- Enables automatic sensor fallback strategies

## Memory Management

### Partition Scheme (16MB ESP32-S3)
```
nvs         : 20KB   - Non-volatile storage (persistent variables)
otadata     : 8KB    - OTA boot selection
factory     : 4MB    - Factory firmware (GPIO15 recovery)
ota_0       : 4MB    - Production firmware (OTA updates)
factory_fs  : 2MB    - Factory web files
prod_fs     : 2MB    - Production web files (OTA updated)
userdata    : 3.9MB  - Data logs and configuration
coredump    : 64KB   - Debug crash dumps
```

### Runtime Memory Management
- **Heap monitoring**: Continuous tracking with console warnings
- **Stack monitoring**: FreeRTOS task stack usage analysis for both cores
- **Fragmentation tracking**: Heap fragmentation percentage
- **Console message queue**: Fixed-size circular buffer (10 messages)
- **Task isolation**: TempTask has independent 4KB stack allocation

**Memory Safety**:
- Main loop watchdog protection prevents hung tasks
- TempTask heartbeat monitoring prevents temperature reading failures
- Stack overflow detection for critical tasks
- Automatic restart if heap falls below 20KB
- Fixed-size buffers to prevent dynamic allocation issues

## Safety Systems

### Hardware Protection
- **INA228 overvoltage protection**: Hardware-level field shutdown at BulkVoltage + 0.1V
- **Emergency field collapse**: Immediate MOSFET disable on voltage spikes
- **Watchdog timer**: 15-second timeout with automatic restart and GPIO reset

### Dual-Layer Monitoring
```cpp
// Main loop monitoring (Core 1)
void loop() {
  esp_task_wdt_reset();  // Feed main watchdog every 50-100ms
  // ... field control and safety systems ...
}

// TempTask monitoring (Core 0)
void checkTempTaskHealth() {
  if (millis() - lastTempTaskHeartbeat > TEMP_TASK_TIMEOUT) {
    // TempTask appears hung - take safety action
    if (dutyCycle > MinDuty + 20) {
      dutyCycle = MinDuty + 10;  // Reduce field for safety
    }
    queueConsoleMessage("CRITICAL: TempTask hung - reducing field for safety");
  }
}
```

### Software Protection Layers
```cpp
// Multiple safety checks in field control
if (currentBatteryVoltage > (ChargingVoltageTarget + 0.2)) {
  digitalWrite(4, 0);           // Emergency field shutdown
  dutyCycle = MinDuty;
  fieldCollapseTime = millis(); // 10-second lockout
  return;
}

// Temperature protection with task health validation
if (!tempTaskHealthy || TempToUse > TemperatureLimitF) {
  dutyCycle -= 2 * dutyStep;    // Aggressive reduction
}

// Current limits
if (Bcur > MaximumAllowedBatteryAmps) {
  dutyCycle -= dutyStep;        // Battery protection
}
```

### Alarm Conditions
- High alternator temperature
- Battery voltage too high/low
- Excessive alternator or battery current
- Sensor failures (stale data detection)
- System health issues (low memory, stack overflow)
- TempTask monitoring failures

## Configuration Management

### Settings Storage
- **LittleFS files**: Human-readable text files for each setting
- **NVS storage**: Binary data for runtime variables and statistics
- **Persistent variables**: Battery SOC, energy totals, thermal damage accumulation
- **Session variables**: Reset on each boot (current maximums, loop times)

### Default Value System
```cpp
// Example: Target current setting
if (!LittleFS.exists("/TargetAmps.txt")) {
  writeFile(LittleFS, "/TargetAmps.txt", String(TargetAmps).c_str());
} else {
  TargetAmps = readFile(LittleFS, "/TargetAmps.txt").toInt();
}
```

**Benefits**:
- Settings survive firmware updates
- Easy troubleshooting (readable files)
- Factory reset capability
- Individual setting modification without recompilation

## Performance Characteristics

### Timing Requirements
- **Main loop (Core 1)**: 50-100ms typical, 15000ms maximum (watchdog limit)
- **TempTask (Core 0)**: 10-second cycle with 5-second blocking conversion
- **Sensor reading**: ADS1115 ~20ms per channel, INA228 ~5ms
- **Field adjustment**: Every 50ms (configurable)
- **Web updates**: Every 50ms (real-time data), 2-3 seconds (status data)

### Core Utilization
- **Core 1 CPU load**: 15-25% average (field control, sensors, safety)
- **Core 0 CPU load**: 5-15% average (WiFi, TempTask, system tasks)
- **Total system load**: 20-40% peak during web interface updates
- **Memory usage**: ~200KB typical (320KB available)
- **Network bandwidth**: ~2KB/second for real-time web interface

### Resource Usage
- **Flash memory**: ~1.4MB firmware (4MB available)
- **Task stacks**: 4KB TempTask, 8KB main loop, 4KB WiFi tasks
- **Heap allocation**: Fixed-size buffers, minimal dynamic allocation
- **I2C bandwidth**: ~2kHz for continuous sensor operation

### Scalability Limits
- **Maximum sensor channels**: 4 analog (ADS1115), expandable via I2C
- **Web clients**: 5-10 simultaneous connections tested
- **Data logging**: 180 days local storage in userdata partition
- **Settings**: 100+ individual parameters in LittleFS files
- **Task capacity**: Limited by ESP32 FreeRTOS task scheduler

## Development Environment

### Required Tools
- **Arduino IDE 2.x** with ESP32 board package
- **Custom partition scheme**: `partitions.csv` in sketch folder
- **Libraries**: 20+ external libraries for sensors and communication
- **Build configuration**: ESP32-S3, 16MB flash, 240MHz CPU, PSRAM enabled

### Build Process
```bash
# Board configuration
FQBN="esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom,CPUFreq=240,PSRAMMode=enabled"

# Compilation
arduino-cli compile --fqbn $FQBN --output-dir ./build .

# OTA package creation
tar --format=ustar -cf firmware.tar -C build firmware.bin data/
```

### Core Assignment Verification
```cpp
// Verify task core assignments during development
void printTaskInfo() {
  TaskHandle_t tempTaskHandle = xTaskGetHandle("TempTask");
  TaskHandle_t loopTaskHandle = xTaskGetCurrentTaskHandle();
  
  Serial.printf("TempTask running on core: %d\n", xTaskGetAffinity(tempTaskHandle));
  Serial.printf("Main loop running on core: %d\n", xTaskGetAffinity(loopTaskHandle));
}
```

This system overview provides the foundation for understanding the detailed subsystem documentation that follows. The dual-core architecture with independent safety monitoring ensures reliable alternator control while maintaining responsive user interfaces and comprehensive system protection.