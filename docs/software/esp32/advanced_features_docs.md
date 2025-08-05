# Advanced Features

## Overview

The regulator includes several advanced features that extend beyond basic alternator control: weather-based solar integration, comprehensive data persistence, secure over-the-air updates, and multiple recovery modes. These features enable sophisticated energy management and robust field deployment.

## Weather Integration and Solar Forecasting

### Purpose
Weather integration enables intelligent charging decisions based on solar panel output forecasts. When sufficient solar energy is predicted, the system can reduce or eliminate alternator charging to save fuel and engine runtime.

### Data Source and API Integration

The system uses the Open-Meteo weather service for solar irradiance forecasting:

```cpp
bool fetchWeatherData() {
  // Validate GPS coordinates from NMEA data
  if (LatitudeNMEA == 0.0 && LongitudeNMEA == 0.0) {
    weatherLastError = "No GPS coordinates available";
    weatherDataValid = 0;
    return false;
  }
  
  // Build API request URL
  String url = "https://api.open-meteo.com/v1/forecast?";
  url += "latitude=" + String(LatitudeNMEA, 6);
  url += "&longitude=" + String(LongitudeNMEA, 6);
  url += "&daily=shortwave_radiation_sum";
  url += "&timezone=auto";
  
  HTTPClient http;
  WiFiClientSecure client;
  client.setInsecure();  // Open-Meteo uses standard certificates
  
  http.begin(client, url);
  http.setTimeout(WeatherTimeoutMs);
  int httpResponseCode = http.GET();
  
  if (httpResponseCode == 200) {
    String payload = http.getString();
    return parseWeatherResponse(payload);
  }
  
  weatherLastError = "HTTP error: " + String(httpResponseCode);
  return false;
}
```

**API Benefits**:
- **Free service**: No API key required
- **Global coverage**: Worldwide weather data
- **Solar-specific**: Shortwave radiation data for PV calculations
- **Reliable**: European Weather Service data source

### Solar Power Calculation

Weather data is converted to predicted solar panel output:

```cpp
bool parseWeatherResponse(String payload) {
  DynamicJsonDocument doc(4096);
  DeserializationError error = deserializeJson(doc, payload);
  if (error) return false;
  
  // Extract daily shortwave radiation data (MJ/m²/day)
  JsonArray radiation = doc["daily"]["shortwave_radiation_sum"];
  if (radiation.size() < 3) return false;
  
  float mjToday = radiation[0];
  float mjTomorrow = radiation[1];
  float mjDay2 = radiation[2];
  
  // Convert to solar panel kWh output
  // Formula: kWh = (MJ/m²/day × 0.278) ÷ 1000 × PanelWatts × PerformanceRatio
  pKwHrToday = (mjToday * MJ_TO_KWH_CONVERSION / STC_IRRADIANCE) * 
               SolarWatts * performanceRatio;
  pKwHrTomorrow = (mjTomorrow * MJ_TO_KWH_CONVERSION / STC_IRRADIANCE) * 
                  SolarWatts * performanceRatio;
  pKwHr2days = (mjDay2 * MJ_TO_KWH_CONVERSION / STC_IRRADIANCE) * 
               SolarWatts * performanceRatio;
  
  // Store irradiance data for display (convert to Wh/m²/day)
  UVToday = mjToday * 278.0f;
  UVTomorrow = mjTomorrow * 278.0f;
  UVDay2 = mjDay2 * 278.0f;
  
  weatherLastUpdate = millis();
  weatherDataValid = 1;
  return true;
}
```

**Conversion Constants**:
- `MJ_TO_KWH_CONVERSION`: 0.278 (MJ/m² to kWh/m²)
- `STC_IRRADIANCE`: 1000.0 W/m² (Standard Test Conditions)
- `performanceRatio`: 0.75 typical (real-world losses)

### Weather Mode Decision Logic

The system analyzes multi-day forecasts to make charging decisions:

```cpp
void analyzeWeatherMode() {
  if (!weatherDataValid || !weatherModeEnabled) {
    currentWeatherMode = 0;  // Normal operation
    return;
  }
  
  // Count days above solar threshold
  int highSolarDays = 0;
  if (pKwHrToday >= UVThresholdHigh) highSolarDays++;
  if (pKwHrTomorrow >= UVThresholdHigh) highSolarDays++;
  if (pKwHr2days >= UVThresholdHigh) highSolarDays++;
  
  if (highSolarDays >= 2) {
    currentWeatherMode = 1;  // Disable alternator charging
    queueConsoleMessage("Weather Mode: High solar forecast - alternator disabled");
  } else {
    currentWeatherMode = 0;  // Normal operation
  }
}
```

**Decision Criteria**:
- **Multi-day analysis**: Requires 2+ days above threshold
- **Conservative approach**: Prevents single-day weather errors
- **User configurable**: UVThresholdHigh adjustable per installation

### Integration with Field Control

Weather mode affects alternator operation at the control level:

```cpp
// In AdjustField() or AdjustFieldLearnMode()
if (weatherModeEnabled == 1 && currentWeatherMode == 1) {
  uTargetAmps = -99;  // Force minimum field output
  queueConsoleMessage("Weather Mode: Alternator charging paused for solar");
}
```

**Control Integration**:
- **Seamless operation**: Works with both standard and learning modes
- **Override capability**: Manual override available via web interface
- **Automatic recovery**: Resumes charging when weather changes

### Configuration Parameters

| Parameter | Default | Range | Description |
|-----------|---------|--------|-------------|
| weatherModeEnabled | 0 | 0-1 | Enable weather-based control |
| UVThresholdHigh | 100 | 50-500 | Solar threshold (Wh/m²/day) |
| SolarWatts | 660 | 100-5000 | Solar panel capacity (watts) |
| performanceRatio | 0.75 | 0.5-1.0 | Real-world efficiency factor |
| WeatherUpdateInterval | 21600000 | 3600000-86400000 | Update interval (6 hours default) |
| WeatherTimeoutMs | 10000 | 5000-30000 | HTTP timeout (ms) |

## File System Management

### Dual File System Architecture

The regulator uses two complementary storage systems for different data types:

**LittleFS (Flash File System)**:
- Configuration files (human-readable text)
- Web interface files (HTML, CSS, JavaScript)
- Calibration data and user settings
- Factory recovery files

**NVS (Non-Volatile Storage)**:
- Runtime variables and statistics
- Binary data structures
- High-frequency updates
- Power-cycle safe storage

### LittleFS Implementation

```cpp
bool ensureLittleFS() {
  if (littleFSMounted) return true;
  
  Serial.println("Initializing LittleFS...");
  if (!LittleFS.begin(true, "/littlefs", 10, "spiffs")) {
    Serial.println("CRITICAL: LittleFS mount failed! Attempting format...");
    if (!LittleFS.begin(true)) {
      Serial.println("CRITICAL: LittleFS format failed - filesystem unavailable");
      littleFSMounted = false;
      return false;
    }
  }
  
  littleFSMounted = true;
  return true;
}
```

**Partition Configuration**:
- **Factory partition**: 2MB read-only recovery files
- **Production partition**: 2MB user-updatable files
- **Auto-formatting**: Creates filesystem if corrupted

### Configuration File Management

Settings are stored as individual text files for easy troubleshooting:

```cpp
void InitSystemSettings() {
  // Example: Load target current setting
  if (!LittleFS.exists("/TargetAmps.txt")) {
    writeFile(LittleFS, "/TargetAmps.txt", String(TargetAmps).c_str());
  } else {
    TargetAmps = readFile(LittleFS, "/TargetAmps.txt").toInt();
  }
  
  // Repeat for 100+ individual settings...
}

String readFile(fs::FS &fs, const char *path) {
  File file = fs.open(path, "r");
  if (!file || file.isDirectory()) {
    Serial.println("Failed to open file: " + String(path));
    return String();
  }
  
  String fileContent;
  while (file.available()) {
    fileContent += String((char)file.read());
  }
  file.close();
  return fileContent;
}

void writeFile(fs::FS &fs, const char *path, const char *message) {
  File file = fs.open(path, "w");
  if (!file) {
    Serial.println("Failed to open file for writing: " + String(path));
    return;
  }
  
  file.print(message);
  file.close();
}
```

**File Examples**:
```
/TargetAmps.txt → "40"
/BulkVoltage.txt → "13.9"
/TemperatureLimitF.txt → "150"
/SwitchingFrequency.txt → "15000"
```

### NVS Storage for Runtime Data

High-frequency and binary data uses NVS for efficiency:

```cpp
void saveNVSData() {
  nvs_handle_t nvs_handle;
  esp_err_t err = nvs_open("storage", NVS_READWRITE, &nvs_handle);
  if (err != ESP_OK) return;
  
  // Battery state (frequent updates)
  nvs_set_i32(nvs_handle, "SOC_percent", (int32_t)SOC_percent);
  nvs_set_i32(nvs_handle, "CoulombCount", (int32_t)CoulombCount_Ah_scaled);
  
  // Energy tracking (cumulative totals)
  nvs_set_u32(nvs_handle, "ChargedEnergy", (uint32_t)ChargedEnergy);
  nvs_set_u32(nvs_handle, "DischrgdEnergy", (uint32_t)DischargedEnergy);
  nvs_set_u32(nvs_handle, "AltChrgdEnergy", (uint32_t)AlternatorChargedEnergy);
  
  // Thermal stress accumulation (binary float data)
  nvs_set_blob(nvs_handle, "InsulDamage", &CumulativeInsulationDamage, sizeof(float));
  nvs_set_blob(nvs_handle, "GreaseDamage", &CumulativeGreaseDamage, sizeof(float));
  nvs_set_blob(nvs_handle, "BrushDamage", &CumulativeBrushDamage, sizeof(float));
  
  // Dynamic calibration factors
  nvs_set_blob(nvs_handle, "ShuntGain", &DynamicShuntGainFactor, sizeof(float));
  nvs_set_blob(nvs_handle, "AltZero", &DynamicAltCurrentZero, sizeof(float));
  
  nvs_commit(nvs_handle);
  nvs_close(nvs_handle);
}
```

**NVS Data Categories**:
- **Battery state**: SOC, coulomb count (updated every 2 seconds)
- **Energy totals**: Charged/discharged energy (persistent lifetime totals)
- **Thermal damage**: Cumulative damage factors (float precision required)
- **Calibration**: Dynamic correction factors (blob storage for precision)

## Factory Reset and Recovery Modes

### GPIO-Based Recovery System

Three GPIO pins provide different recovery modes for field deployment:

```cpp
void checkFactoryReset() {
  pinMode(15, INPUT_PULLUP);
  
  if (digitalRead(15) == LOW) {
    Serial.println("🏭 FACTORY RESET REQUESTED - GPIO15 pulled low");
    
    // Switch to factory partition (brick recovery)
    const esp_partition_t* factory = esp_partition_find_first(
      ESP_PARTITION_TYPE_APP, ESP_PARTITION_SUBTYPE_APP_FACTORY, NULL);
    
    if (factory != NULL) {
      esp_err_t err = esp_ota_set_boot_partition(factory);
      if (err == ESP_OK) {
        Serial.println("✅ Factory reset initiated - restarting in 3 seconds...");
        delay(3000);
        ESP.restart();
      }
    }
  }
}
```

**Recovery Pin Functions**:
- **GPIO15**: Factory partition boot (brick recovery from failed OTA)
- **GPIO34**: AP mode selection (client vs hotspot operation)
- **GPIO35**: WiFi configuration reset (force config portal)

### Software Factory Reset

Complete settings reset via web interface:

```cpp
server.on("/factoryReset", HTTP_POST, [](AsyncWebServerRequest *request) {
  if (!validatePassword(request->getParam("password", true)->value())) {
    request->send(403, "text/plain", "FAIL");
    return;
  }
  
  queueConsoleMessage("FACTORY RESET: Restoring all defaults...");
  
  // Delete all settings files
  const char *settingsFiles[] = {
    "/TemperatureLimitF.txt", "/BulkVoltage.txt", "/TargetAmps.txt",
    "/FloatVoltage.txt", "/dutyStep.txt", "/FieldAdjustmentInterval.txt",
    // ... 100+ settings files
  };
  
  for (int i = 0; i < sizeof(settingsFiles) / sizeof(settingsFiles[0]); i++) {
    if (LittleFS.exists(settingsFiles[i])) {
      LittleFS.remove(settingsFiles[i]);
    }
  }
  
  // Clear NVS data
  nvs_handle_t nvs_handle;
  esp_err_t err = nvs_open("storage", NVS_READWRITE, &nvs_handle);
  if (err == ESP_OK) {
    nvs_erase_all(nvs_handle);
    nvs_commit(nvs_handle);
    nvs_close(nvs_handle);
  }
  
  // Reinitialize with defaults
  InitSystemSettings();
  request->send(200, "text/plain", "OK");
});
```

**Reset Scope**:
- **All configuration files**: Returns to firmware defaults
- **NVS data**: Clears runtime statistics and calibration
- **Preserves firmware**: Does not affect main application
- **Preserves WiFi credentials**: Network settings optional

### Automatic Recovery Features

```cpp
void checkAndRestart() {
  static unsigned long lastRestartTime = 0;
  const unsigned long RESTART_INTERVAL = 7200000;  // 2 hours
  
  unsigned long currentMillis = millis();
  
  // Handle millis() rollover (49.7 days)
  if (currentMillis < lastRestartTime) {
    lastRestartTime = 0;
  }
  
  if (currentMillis - lastRestartTime >= RESTART_INTERVAL) {
    events.send("Performing scheduled restart for system maintenance", "console");
    delay(2500);
    ESP.restart();
  }
}
```

**Automatic Recovery**:
- **Scheduled restart**: Every 2 hours for memory cleanup
- **Watchdog reset**: 15-second hang detection
- **Memory protection**: Restart if heap falls below 20KB
- **WiFi recovery**: Automatic reconnection with exponential backoff

## Over-The-Air (OTA) Update System

### Secure OTA Architecture

The OTA system implements cryptographic verification and streaming installation:

```cpp
// Server configuration
const char *OTA_SERVER_URL = "https://ota.xengineering.net";
const char *FIRMWARE_VERSION = "1.9.0";  // Semantic versioning required

// RSA public key for signature verification (4096-bit)
const char *OTA_PUBLIC_KEY = 
  "-----BEGIN PUBLIC KEY-----\n"
  "MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAp2sRgMjD4wazKHo6Rk3g\n"
  // ... (truncated for brevity)
  "-----END PUBLIC KEY-----\n";
```

### Update Check Process

```cpp
void checkForOTAUpdate() {
  HTTPClient http;
  WiFiClientSecure client;
  client.setCACert(server_root_ca);  // Let's Encrypt R10 certificate
  
  String url = String(OTA_SERVER_URL) + "/api/firmware/check.php";
  http.begin(client, url);
  
  // Device identification headers
  http.addHeader("Device-ID", getDeviceId());
  http.addHeader("Current-Version", FIRMWARE_VERSION);
  http.addHeader("Hardware-Version", "ESP32-WROOM-32");
  
  int httpCode = http.GET();
  
  if (httpCode == 200) {
    String response = http.getString();
    UpdateInfo updateInfo = parseUpdateResponse(response);
    
    if (updateInfo.hasUpdate) {
      Serial.println("🚀 UPDATE AVAILABLE: " + updateInfo.version);
      performOTAUpdate(updateInfo);
    }
  } else if (httpCode == 204) {
    Serial.println("✅ No updates available");
  }
  
  http.end();
}

String getDeviceId() {
  uint64_t chipid = ESP.getEfuseMac();
  return String((uint32_t)(chipid >> 32), HEX) + String((uint32_t)chipid, HEX);
}
```

### Streaming TAR Extraction

Updates are delivered as signed TAR packages containing firmware and web files:

```cpp
void performStreamingOTAUpdate(const UpdateInfo& updateInfo, const String& signatureBase64) {
  StreamingExtractor extractor;
  initStreamingExtractor(&extractor);
  
  HTTPClient http;
  WiFiClientSecure client;
  client.setCACert(server_root_ca);
  
  http.begin(client, updateInfo.firmwareUrl);
  int httpCode = http.GET();
  if (httpCode != 200) return;
  
  WiFiClient* stream = http.getStreamPtr();
  const size_t CHUNK_SIZE = 1024;
  uint8_t inputBuffer[CHUNK_SIZE];
  
  int totalDownloaded = 0;
  int contentLength = http.getSize();
  
  while (totalDownloaded < contentLength) {
    int actualRead = stream->readBytes(inputBuffer, 
                                      min(CHUNK_SIZE, contentLength - totalDownloaded));
    
    if (actualRead > 0) {
      totalDownloaded += actualRead;
      
      // Update hash for signature verification
      mbedtls_md_update(&extractor.hashCtx, inputBuffer, actualRead);
      
      // Process TAR data directly (no decompression)
      if (!processDataChunk(&extractor, inputBuffer, actualRead)) {
        break;
      }
    }
  }
  
  // Verify package signature
  if (verifyPackageSignature(packageData, packageSize, signatureBase64)) {
    // Finalize OTA and restart
    esp_ota_end(extractor.otaHandle);
    esp_ota_set_boot_partition(extractor.otaPartition);
    ESP.restart();
  }
}
```

**Security Features**:
- **RSA-4096 signature verification**: Cryptographic package validation
- **HTTPS with certificate pinning**: Secure transport
- **Version validation**: Semantic versioning prevents downgrades
- **Atomic updates**: Complete package verification before installation

### TAR Package Processing

```cpp
bool processDataChunk(StreamingExtractor* extractor, uint8_t* data, size_t dataSize) {
  size_t processed = 0;
  
  while (processed < dataSize) {
    if (extractor->inTarHeader) {
      // Read 512-byte TAR header
      size_t headerRemaining = 512 - extractor->tarHeaderPos;
      size_t toCopy = min(headerRemaining, dataSize - processed);
      
      memcpy(extractor->tarHeader + extractor->tarHeaderPos, data + processed, toCopy);
      extractor->tarHeaderPos += toCopy;
      processed += toCopy;
      
      if (extractor->tarHeaderPos >= 512) {
        // Complete header - check for end of archive
        bool allZeros = true;
        for (int i = 0; i < 512; i++) {
          if (extractor->tarHeader[i] != 0) {
            allZeros = false;
            break;
          }
        }
        
        if (allZeros) {
          return true;  // End of archive
        }
        
        parseTarHeader(extractor);
        extractor->inTarHeader = false;
      }
    } else {
      // Process file data
      size_t fileRemaining = extractor->currentFileSize - extractor->currentFilePos;
      size_t toWrite = min(fileRemaining, dataSize - processed);
      
      if (extractor->isCurrentFileFirmware && extractor->otaStarted) {
        // Write to firmware partition
        esp_ota_write(extractor->otaHandle, data + processed, toWrite);
      } else if (extractor->currentWebFile) {
        // Write to LittleFS web files
        extractor->currentWebFile.write(data + processed, toWrite);
      }
      
      processed += toWrite;
      extractor->currentFilePos += toWrite;
      
      // Handle file completion and padding
      if (extractor->currentFilePos >= extractor->currentFileSize) {
        completeCurrentFile(extractor);
      }
    }
  }
  
  return true;
}
```

**TAR Package Structure**:
```
firmware.tar
├── firmware.bin     → OTA partition
├── index.html      → LittleFS /index.html
├── app.js          → LittleFS /app.js
├── style.css       → LittleFS /style.css
└── (other web files)
```

## Special Operating Modes

### Force Float Mode

Precision float charging mode targets zero net battery current:

```cpp
if (ForceFloat == 1) {
  uTargetAmps = 0;              // Target zero net battery current
  targetCurrent = Bcur;         // Use battery current, not alternator current
  queueConsoleMessage("Force Float mode enabled - targeting zero battery current");
}
```

**Benefits**:
- **Precise float voltage**: Compensates for vessel loads
- **Battery protection**: Prevents overcharging during long float periods
- **Load awareness**: Automatically adjusts for changing electrical loads

### Limp Home Mode

Emergency operation with minimal functionality:

```cpp
if (LimpHome == 1) {
  // Disable advanced features
  LearningMode = 0;
  weatherModeEnabled = 0;
  AutoShuntGainCorrection = 0;
  
  // Use safe, conservative settings
  TargetAmps = min(TargetAmps, 20);  // Limit to 20A maximum
  TemperatureLimitF = min(TemperatureLimitF, 140);  // Lower temp limit
  
  queueConsoleMessage("LIMP HOME: Operating with reduced functionality");
}
```

### Manual Override Mode

Complete manual control bypassing all automatic systems:

```cpp
if (ManualFieldToggle == 1) {
  dutyCycle = ManualDutyTarget;     // Direct duty cycle control
  dutyCycle = constrain(dutyCycle, 0, 100);
  
  // Bypass all automatic calculations
  uTargetAmps = 0;
  targetCurrent = 0;
  
  queueConsoleMessage("Manual override active - duty cycle: " + String(dutyCycle) + "%");
}
```

### BMS Integration Mode

Integration with Battery Management Systems for LiFePO4 protection:

```cpp
if (bmsLogic == 1) {
  bmsSignalActive = !digitalRead(36);  // Read BMS signal (inverted for optocoupler)
  
  if (bmsLogicLevelOff == 0) {
    // BMS gives LOW signal when charging NOT desired
    chargingEnabled = chargingEnabled && bmsSignalActive;
  } else {
    // BMS gives HIGH signal when charging NOT desired  
    chargingEnabled = chargingEnabled && !bmsSignalActive;
  }
  
  if (!chargingEnabled) {
    queueConsoleMessage("BMS requesting charge stop - alternator disabled");
  }
}
```

**BMS Integration Features**:
- **Configurable polarity**: Active high or low signal
- **Override capability**: BMS can completely disable charging
- **Status indication**: Web interface shows BMS state
- **Safety priority**: BMS signal overrides all other control logic

## System Maintenance Features

### Scheduled Maintenance

```cpp
// Automatic restart every 2 hours for memory management
void checkAndRestart() {
  static unsigned long lastRestartTime = 0;
  const unsigned long RESTART_INTERVAL = 7200000;  // 2 hours
  
  if (millis() - lastRestartTime >= RESTART_INTERVAL) {
    events.send("Performing scheduled restart for system maintenance", "console");
    delay(2500);
    ESP.restart();
  }
}
```

### Data Persistence Management

```cpp
// Periodic NVS data saves (every 10 seconds)
if (currentTime - lastDataSaveTime >= DataSaveInterval) {
  saveNVSData();
  lastDataSaveTime = currentTime;
}

// Session statistics tracking
void RestoreLastSessionValues() {
  // Load previous session statistics for comparison
  LastSessionDuration = readNVSValue("SessionDur");
  LastSessionMaxLoopTime = readNVSValue("MaxLoop"); 
  lastSessionMinHeap = readNVSValue("MinHeap");
}
```

### Reset Reason Tracking

```cpp
void captureResetReason() {
  // Load previous reset reason
  ancientResetReason = readFile(LittleFS, "/LastResetReason.txt").toInt();
  
  // Capture current reset reason
  int rawReason = (int)esp_reset_reason();
  switch (rawReason) {
    case ESP_RST_SW: LastResetReason = 1; break;          // Software restart
    case ESP_RST_DEEPSLEEP: LastResetReason = 2; break;   // Deep sleep wakeup
    case ESP_RST_EXT: LastResetReason = 3; break;         // External reset
    case ESP_RST_TASK_WDT: LastResetReason = 4; break;    // Task watchdog
    case ESP_RST_PANIC: LastResetReason = 5; break;       // Software panic
    case ESP_RST_BROWNOUT: LastResetReason = 6; break;    // Brownout reset
    default: LastResetReason = 8; break;                  // Unknown
  }
  
  // Save for next session
  writeFile(LittleFS, "/LastResetReason.txt", String(LastResetReason).c_str());
  totalPowerCycles++;
}
```

This advanced features system provides sophisticated energy management capabilities while maintaining robust operation and easy field deployment through comprehensive recovery and maintenance systems.