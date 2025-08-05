# Configuration & Development

## Overview

This section covers the complete configuration system, development environment setup, build procedures, and troubleshooting methods for the Xengineering Alternator Regulator. The system uses a hierarchical configuration approach with web-based settings management and comprehensive debugging tools.

## Configuration System Architecture

### Configuration Hierarchy

The regulator uses a multi-layer configuration system:

1. **Firmware defaults**: Hard-coded values in source code
2. **LittleFS settings files**: User-configurable parameters (persistent)
3. **NVS runtime data**: Frequently updated operational data
4. **Web interface overrides**: Real-time parameter changes
5. **Emergency defaults**: Safe fallback values for critical failures

### Settings Categories

**Basic Operation**:
- Target currents (Hi/Low modes)
- Voltage thresholds (bulk/float)
- Temperature limits
- Field control parameters

**Advanced Control**:
- RPM-based current curves (20 points)
- Learning mode parameters
- PID tuning coefficients
- Safety override thresholds

**System Configuration**:
- Sensor source selection
- Communication interface enables
- WiFi and network settings
- Alarm and protection parameters

**Calibration Data**:
- Sensor offsets and scaling
- Dynamic correction factors
- Thermal model parameters
- Hardware-specific adjustments

## Settings Storage Implementation

### LittleFS File-Based Settings

Each setting is stored as an individual text file for easy troubleshooting:

```cpp
void InitSystemSettings() {
  // Example: Target current setting
  if (!LittleFS.exists("/TargetAmps.txt")) {
    // Create file with firmware default
    writeFile(LittleFS, "/TargetAmps.txt", String(TargetAmps).c_str());
  } else {
    // Load saved value
    TargetAmps = readFile(LittleFS, "/TargetAmps.txt").toInt();
  }
  
  // Example: Floating point setting
  if (!LittleFS.exists("/BulkVoltage.txt")) {
    writeFile(LittleFS, "/BulkVoltage.txt", String(BulkVoltage).c_str());
  } else {
    BulkVoltage = readFile(LittleFS, "/BulkVoltage.txt").toFloat();
  }
  
  // Example: Scaled integer setting (voltage × 100)
  if (!LittleFS.exists("/ChargedVoltage.txt")) {
    writeFile(LittleFS, "/ChargedVoltage.txt", String(ChargedVoltage_Scaled).c_str());
  } else {
    ChargedVoltage_Scaled = readFile(LittleFS, "/ChargedVoltage.txt").toInt();
  }
}
```

**File Examples**:
```
/TargetAmps.txt → "40"
/BulkVoltage.txt → "13.9"
/TemperatureLimitF.txt → "150"
/FieldAdjustmentInterval.txt → "50"
/RPMScalingFactor.txt → "2000"
```

### Web Interface Settings Handler

The mega-handler processes all setting changes from the web interface:

```cpp
server.on("/get", HTTP_GET, [](AsyncWebServerRequest *request) {
  // Password validation
  if (!request->hasParam("password") || 
      strcmp(request->getParam("password")->value().c_str(), requiredPassword) != 0) {
    request->send(403, "text/plain", "Forbidden");
    return;
  }
  
  bool foundParameter = false;
  String inputMessage;
  
  // Basic settings
  if (request->hasParam("TargetAmps")) {
    foundParameter = true;
    inputMessage = request->getParam("TargetAmps")->value();
    writeFile(LittleFS, "/TargetAmps.txt", inputMessage.c_str());
    TargetAmps = inputMessage.toInt();
  }
  
  // Floating point settings
  else if (request->hasParam("BulkVoltage")) {
    foundParameter = true;
    inputMessage = request->getParam("BulkVoltage")->value();
    writeFile(LittleFS, "/BulkVoltage.txt", inputMessage.c_str());
    BulkVoltage = inputMessage.toFloat();
  }
  
  // Scaled settings (user enters 1.12, stored as 112)
  else if (request->hasParam("PeukertExponent")) {
    foundParameter = true;
    inputMessage = request->getParam("PeukertExponent")->value();
    writeFile(LittleFS, "/PeukertExponent.txt", inputMessage.c_str());
    PeukertExponent_scaled = (int)(inputMessage.toFloat() * 100);
  }
  
  // Time conversion (user enters hours, stored as seconds)
  else if (request->hasParam("FLOAT_DURATION")) {
    foundParameter = true;
    inputMessage = request->getParam("FLOAT_DURATION")->value();
    int hours = inputMessage.toInt();
    int seconds = hours * 3600;
    writeFile(LittleFS, "/FLOAT_DURATION.txt", String(seconds).c_str());
    FLOAT_DURATION = seconds;
  }
  
  // RPM table handling (multiple parameters per form submission)
  if (request->hasParam("RPM1")) {
    foundParameter = true;
    inputMessage = request->getParam("RPM1")->value();
    writeFile(LittleFS, "/RPM1.txt", inputMessage.c_str());
    RPM1 = inputMessage.toInt();
  }
  
  // ... continues for 100+ parameters
  
  if (!foundParameter) {
    inputMessage = "No message sent, the request_hasParam found no match";
  }
  
  request->send(200, "text/plain", inputMessage);
});
```

**Parameter Processing Notes**:
- **Immediate effect**: Settings changes apply instantly
- **Persistent storage**: All changes saved to LittleFS
- **Input validation**: Basic range checking and type conversion
- **Password protection**: All changes require admin password
- **Echo response**: Web interface receives confirmation

### Reset Functions

Individual reset buttons for operational data:

```cpp
// Reset maximum value trackers
else if (request->hasParam("ResetTemp")) {
  foundParameter = true;
  MaxAlternatorTemperatureF = 0;
  writeFile(LittleFS, "/MaxAlternatorTemperatureF.txt", "0");
  queueConsoleMessage("Max Alternator Temp: Reset requested from web interface");
}

else if (request->hasParam("ResetCurrent")) {
  foundParameter = true;
  MeasuredAmpsMax = 0;
  writeFile(LittleFS, "/MeasuredAmpsMax.txt", "0");
  queueConsoleMessage("Max Battery Current: Reset requested from web interface");
}

// Reset runtime counters
else if (request->hasParam("ResetEngineRunTime")) {
  foundParameter = true;
  EngineRunTime = 0;
  queueConsoleMessage("Engine Run Time: Reset requested from web interface");
}

// Manual SOC adjustment
else if (request->hasParam("ManualSOCPoint")) {
  foundParameter = true;
  inputMessage = request->getParam("ManualSOCPoint")->value();
  writeFile(LittleFS, "/ManualSOCPoint.txt", inputMessage.c_str());
  ManualSOCPoint = inputMessage.toInt();
  SOC_percent = ManualSOCPoint * 100;  // Convert to scaled format
  CoulombCount_Ah_scaled = (ManualSOCPoint * BatteryCapacity_Ah);
  queueConsoleMessage("SoC manually set to: " + String(ManualSOCPoint) + "%");
}
```

## Calibration Procedures

### Current Sensor Calibration

**Alternator Current (Hall Effect Sensor)**:
1. Ensure alternator is off (0A output)
2. Note displayed alternator current reading
3. Set `AlternatorCOffset` to negative of reading
4. Verify reading shows 0A at zero output
5. Apply known load and verify accuracy

**Battery Current (INA228 Shunt)**:
1. Disconnect all loads and charging sources
2. Note displayed battery current (should be 0A)
3. Set `BatteryCOffset` to negative of reading
4. Apply known load and verify accuracy

**Dynamic Auto-Zero** (automatic calibration):
```cpp
// Enable automatic zero correction
AutoAltCurrentZero = 1;  // Enables hourly auto-zero cycles

// Manual trigger for immediate auto-zero
void startAutoZero(String reason) {
  autoZeroStartTime = millis();
  queueConsoleMessage("Auto-zero: Starting alternator current zeroing (" + reason + ")");
}
```

### Voltage Sensor Calibration

**Battery Voltage Sources**:
- **INA228**: Factory calibrated, typically ±0.1% accuracy
- **ADS1115**: Voltage divider (1MΩ/75kΩ), may need trimming
- **VE.Direct**: Victron calibrated, use as reference

**Cross-Validation**:
```cpp
// Automatic disagreement detection
if (abs(BatteryV - IBV) > 0.1) {
  queueConsoleMessage("Voltage sensor disagreement - Field shut off for safety!");
  // Triggers emergency field shutdown
}
```

### Temperature Sensor Calibration

**OneWire DS18B20**:
- Factory calibrated ±0.5°C
- No user calibration required
- Multiple sensors averaged if installed

**Thermistor Calibration**:
```cpp
// Thermistor parameters (user configurable)
float R_fixed = 10000.0;     // Series resistor (measured value)
float Beta = 3950.0;         // Thermistor Beta constant
float R0 = 10000.0;          // Resistance at T0
float T0_C = 25.0;           // Reference temperature (°C)
```

**Calibration Procedure**:
1. Measure actual series resistor value
2. Update `R_fixed` parameter
3. Compare readings to known temperature
4. Adjust `Beta` if necessary (typically 3900-4000K)

### RPM Sensor Calibration

```cpp
// RPM scaling factor (user adjustable)
int RPMScalingFactor = 2000;  // Adjust until display matches tachometer
```

**Calibration Procedure**:
1. Run engine at known RPM (use trusted tachometer)
2. Note displayed RPM on regulator
3. Calculate correction factor: `New_Factor = Old_Factor × (Actual_RPM / Displayed_RPM)`
4. Update `RPMScalingFactor` and verify accuracy

## Development Environment Setup

### Required Software

**Arduino IDE Configuration**:
```
Arduino IDE: 2.3.0 or later
ESP32 Board Package: 3.0.0 or later
Board: ESP32S3 Dev Module
Flash Size: 16MB
Partition Scheme: Custom (partitions.csv)
CPU Frequency: 240MHz
PSRAM: Enabled
```

**Required Libraries**:
```cpp
// Core libraries (Arduino Library Manager)
#include <OneWire.h>           // DS18B20 temperature sensors
#include <DallasTemperature.h> // Dallas temperature library
#include <U8g2lib.h>           // OLED display
#include <ADS1115_lite.h>      // ADS1115 ADC
#include <WiFi.h>              // ESP32 WiFi
#include <AsyncTCP.h>          // Async TCP for web server
#include <ESPAsyncWebServer.h> // Async web server
#include <LittleFS.h>          // File system
#include <ArduinoJson.h>       // JSON parsing
#include <PID_v1.h>            // PID controller
#include <TinyGPSPlus.h>       // GPS parsing
#include <HTTPClient.h>        // HTTP client
#include <WiFiClientSecure.h>  // HTTPS client

// Custom libraries (included in project)
#include "VeDirectFrameHandler.h"  // Victron VE.Direct
#include "INA228.h"                // INA228 library
#include <NMEA2000_CAN.h>          // NMEA2000 for ESP32
#include <N2kMessages.h>           // NMEA2000 messages
```

**Library Installation Notes**:
- Use **mathieucarbou** versions of AsyncTCP and ESPAsyncWebServer
- INA228 library may require manual installation
- NMEA2000 libraries from Timo Lappalainen

### Project Structure

```
XRegulator/
├── XRegulator.ino              # Main sketch file
├── 2_importantFunctions.ino    # Core functions
├── 3_nonImportantFunctions.ino # Utility functions
├── partitions.csv              # Custom partition table
├── data/                       # Web interface files
│   ├── index.html
│   ├── app.js
│   ├── style.css
│   └── favicon.ico
└── libraries/                  # Custom libraries
    ├── INA228/
    ├── VeDirectFrameHandler/
    └── ...
```

### Custom Partition Table

**partitions.csv**:
```csv
# Name,   Type, SubType, Offset,  Size,     Flags
nvs,      data, nvs,     0x9000,  0x5000,
otadata,  data, ota,     0xe000,  0x2000,
factory,  app,  factory, 0x10000, 0x400000,
ota_0,    app,  ota_0,   0x410000,0x400000,
factory_fs,data,spiffs,  0x810000,0x200000,
prod_fs,  data, spiffs,  0xa10000,0x200000,
userdata, data, spiffs,  0xc10000,0x3e0000,
coredump, data, coredump,0xff0000,0x10000,
```

**Partition Functions**:
- **factory**: Recovery firmware (GPIO15 boot)
- **ota_0**: Production firmware (OTA updates)
- **factory_fs**: Recovery web files
- **prod_fs**: Production web files (OTA updated)
- **userdata**: Data logging and configuration
- **coredump**: Debug crash dumps

### Build Configuration

**Board Configuration**:
```
FQBN: esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom,CPUFreq=240,PSRAMMode=enabled
```

**Compiler Flags**:
```cpp
// Enable features
#define configGENERATE_RUN_TIME_STATS 1  // CPU usage tracking
#define CORE_DEBUG_LEVEL 2               // Moderate debug output

// Memory optimization
#define CONFIG_ARDUHAL_LOG_DEFAULT_LEVEL 2  // Reduce log spam
```

### Development Workflow

**1. Initial Setup**:
```bash
# Clone repository
git clone https://github.com/xengineering/alternator-regulator.git
cd alternator-regulator

# Install board package
arduino-cli core install esp32:esp32

# Install libraries
arduino-cli lib install "OneWire" "DallasTemperature" "U8g2" "ArduinoJson" "PID"
```

**2. Configuration**:
```bash
# Copy partition table to sketch folder
cp partitions.csv /path/to/sketch/

# Upload filesystem (first time only)
arduino-cli upload --fqbn esp32:esp32:esp32s3 --port /dev/ttyUSB0 --upload-field filesystem data/
```

**3. Compilation**:
```bash
# Compile
arduino-cli compile --fqbn esp32:esp32:esp32s3 --output-dir ./build .

# Upload firmware
arduino-cli upload --fqbn esp32:esp32:esp32s3 --port /dev/ttyUSB0 --input-dir ./build
```

**4. OTA Package Creation**:
```bash
# Create TAR package for OTA
tar --format=ustar -cf firmware.tar -C build firmware.bin data/

# Sign package (production)
openssl dgst -sha256 -sign private_key.pem -out firmware.sig firmware.tar
```

## Debugging and Troubleshooting

### Serial Monitor Debugging

**Debug Output Levels**:
```cpp
// Enable verbose debugging
int NMEA2KVerbose = 1;           // NMEA2K message debugging
int ShowLearningDebugMessages = 1; // Learning mode debugging

// Console messages
queueConsoleMessage("DEBUG: Voltage = " + String(IBV, 2) + "V");
```

**Performance Monitoring**:
```cpp
// Loop timing
Serial.println("Loop time: " + String(LoopTime) + "µs");
Serial.println("Max loop time: " + String(MaxLoopTime) + "µs");

// Memory usage
Serial.println("Free heap: " + String(FreeHeap) + "KB");
Serial.println("Min heap: " + String(MinFreeHeap) + "KB");
Serial.println("Heap fragmentation: " + String(Heapfrag) + "%");
```

### Web Interface Debugging

**Console Messages**: Real-time system messages displayed in web interface
**Live Data Display**: All sensor readings with staleness indicators
**Performance Metrics**: Loop timing, memory usage, CPU load
**Configuration Echo**: Settings changes confirmed via console

### Common Issues and Solutions

#### WiFi Connection Problems

**Symptoms**: Cannot access web interface
**Diagnosis**:
```cpp
// Check WiFi status
Serial.println("WiFi Status: " + String(WiFi.status()));
Serial.println("IP Address: " + WiFi.localIP().toString());
Serial.println("Signal Strength: " + String(WiFi.RSSI()) + " dBm");
```

**Solutions**:
1. **GPIO35 to ground**: Force WiFi configuration mode
2. **GPIO34 to ground**: Force AP mode operation
3. **Check signal strength**: Move closer to router or use external antenna
4. **Factory reset**: GPIO15 to ground during boot

#### Sensor Reading Issues

**Symptoms**: Stale data indicators, erratic readings
**Diagnosis**:
```cpp
// Check I2C devices
Wire.begin();
Wire.beginTransmission(0x40);  // INA228 address
byte error = Wire.endTransmission();
if (error == 0) {
  Serial.println("INA228 found");
} else {
  Serial.println("INA228 not responding");
}
```

**Solutions**:
1. **Check wiring**: Verify I2C connections (SDA/SCL)
2. **Power supply**: Ensure stable 3.3V to sensors
3. **Address conflicts**: Verify unique I2C addresses
4. **Sensor replacement**: Use web interface to switch sources

#### Field Control Problems

**Symptoms**: No alternator output, voltage regulation issues
**Diagnosis**:
```cpp
// Check field status
Serial.println("Charging enabled: " + String(chargingEnabled));
Serial.println("Duty cycle: " + String(dutyCycle) + "%");
Serial.println("Field voltage: " + String(vvout, 2) + "V");
Serial.println("GPIO4 status: " + String(digitalRead(4)));
```

**Solutions**:
1. **Check enable signals**: Ignition, OnOff, BMS logic
2. **Verify MOSFET driver**: Measure voltage at field terminals
3. **Emergency field collapse**: Check for voltage spike protection
4. **Manual override**: Use ManualFieldToggle for testing

#### Memory Issues

**Symptoms**: Frequent restarts, console warnings
**Diagnosis**:
```cpp
// Memory analysis
Serial.println("Free heap: " + String(esp_get_free_heap_size()) + " bytes");
Serial.println("Largest free block: " + String(heap_caps_get_largest_free_block(MALLOC_CAP_8BIT)) + " bytes");
Serial.println("Task stack usage:");
// Task stack monitoring output
```

**Solutions**:
1. **Reduce console queue size**: Lower CONSOLE_QUEUE_SIZE
2. **Disable verbose debug**: Set debug flags to 0
3. **Firmware update**: May include memory optimizations
4. **Hardware replacement**: If consistent across multiple units

### Factory Testing Procedures

**1. Hardware Verification**:
- Power supply voltage (12V ±10%)
- I2C device detection (INA228, ADS1115)
- GPIO functionality (field enable, digital inputs)
- Temperature sensor detection (OneWire)

**2. Sensor Calibration**:
- Zero current readings with no load
- Voltage accuracy against reference meter
- Temperature sensor comparison
- RPM scaling factor adjustment

**3. Field Control Testing**:
- Manual duty cycle verification
- Automatic voltage regulation
- Safety shutdowns (overvoltage, overtemperature)
- Emergency field collapse

**4. Communication Testing**:
- WiFi connection in both modes
- Web interface functionality
- NMEA2K/VE.Direct data reception
- OTA update capability

**5. Endurance Testing**:
- 24-hour continuous operation
- Memory leak detection
- Thermal cycling
- Restart recovery testing

This comprehensive configuration and development system enables both field deployment and ongoing development of the alternator regulator platform.