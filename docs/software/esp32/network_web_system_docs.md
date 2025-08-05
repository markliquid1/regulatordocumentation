# Network Interface & Web System

## Overview

The network system provides WiFi connectivity in multiple operational modes with a comprehensive web interface for real-time monitoring and configuration. The system implements GPIO-based mode selection, automatic reconnection, security features, and over-the-air (OTA) update capabilities.

## WiFi System Architecture

### Operational Modes

The system operates in three distinct modes based on GPIO pin states and configuration:

```cpp
enum OperationalMode {
  CONFIG_AP_MODE,       // First boot configuration
  OPERATIONAL_AP_MODE,  // Permanent AP with full functionality
  CLIENT_MODE           // Connected to ship's WiFi
};

OperationalMode getCurrentMode() {
  if (currentWiFiMode == AWIFI_MODE_CLIENT) {
    return CLIENT_MODE;
  } else if (permanentAPMode == 1) {
    return OPERATIONAL_AP_MODE;
  } else {
    return CONFIG_AP_MODE;
  }
}
```

### GPIO Mode Selection

WiFi mode is determined by GPIO pin states during startup:

```cpp
void setupWiFi() {
  // GPIO35 = Configuration Mode Override (highest priority)
  pinMode(35, INPUT_PULLUP);
  bool forceConfigMode = (digitalRead(35) == LOW);
  
  if (forceConfigMode) {
    Serial.println("=== GPIO35 LOW: FORCED CONFIGURATION MODE ===");
    setupAccessPoint();
    setupWiFiConfigServer();
    currentWiFiMode = AWIFI_MODE_AP;
    return;
  }
  
  // Check for existing configuration
  bool hasClientConfig = LittleFS.exists(WIFI_SSID_FILE);
  String saved_ssid = "";
  String saved_password = "";
  
  if (hasClientConfig) {
    saved_ssid = readFile(LittleFS, WIFI_SSID_FILE);
    saved_password = readFile(LittleFS, WIFI_PASS_FILE);
    saved_ssid.trim();
    saved_password.trim();
  }
  
  // GPIO34 = Operational Mode Selection (AP vs Client)
  pinMode(34, INPUT_PULLUP);
  bool requestAPMode = (digitalRead(34) == LOW);
  
  if (requestAPMode) {
    Serial.println("=== GPIO34 LOW: OPERATIONAL AP MODE ===");
    setupAccessPoint();
    currentWiFiMode = AWIFI_MODE_AP;
    setupServer();  // Full alternator interface
    return;
  }
  
  // GPIO34 HIGH = Client Mode Requested
  if (connectToWiFi(saved_ssid.c_str(), saved_password.c_str(), 10000)) {
    currentWiFiMode = AWIFI_MODE_CLIENT;
    setupServer();
  } else {
    currentWiFiMode = AWIFI_MODE_CLIENT;  // Keep trying
  }
}
```

**GPIO Configuration**:
- **GPIO35 LOW**: Force configuration mode (password recovery)
- **GPIO34 LOW**: Operational AP mode (permanent hotspot)
- **GPIO34 HIGH**: Client mode (connect to ship's WiFi)
- **No saved config**: Automatic configuration mode

### Access Point Setup

```cpp
void setupAccessPoint() {
  WiFi.mode(WIFI_AP);
  
  bool apStarted = WiFi.softAP(esp32_ap_ssid.c_str(), esp32_ap_password.c_str());
  
  if (apStarted) {
    Serial.print("AP SSID: ");
    Serial.println(esp32_ap_ssid);
    Serial.print("AP IP address: ");
    Serial.println(WiFi.softAPIP());  // Typically 192.168.4.1
    
    // Start DNS server for captive portal
    dnsServer.start(DNS_PORT, "*", WiFi.softAPIP());
  }
}
```

**Default AP Configuration**:
- **SSID**: "ALTERNATOR_WIFI" (user configurable)
- **Password**: "alternator123" (user configurable)
- **IP Address**: 192.168.4.1
- **DHCP Range**: 192.168.4.2-192.168.4.100

### Client Mode Connection

```cpp
bool connectToWiFi(const char *ssid, const char *password, unsigned long timeout) {
  WiFi.disconnect(true);
  delay(100);
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);
  
  if (strlen(password) > 0) {
    WiFi.begin(ssid, password);
  } else {
    WiFi.begin(ssid);  // Open network
  }
  
  unsigned long startTime = millis();
  int attempts = 0;
  const int maxAttempts = timeout / 500;
  
  while (WiFi.status() != WL_CONNECTED && attempts < maxAttempts) {
    delay(500);
    esp_task_wdt_reset();  // Feed watchdog during connection
    attempts++;
    
    if (attempts % 4 == 0) {  // Progress every 2 seconds
      Serial.printf("WiFi Status: %d, attempt %d/%d\n", WiFi.status(), attempts, maxAttempts);
    }
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("IP address: %s\n", WiFi.localIP().toString().c_str());
    Serial.printf("Signal strength: %d dBm\n", WiFi.RSSI());
    
    // mDNS setup for easy access
    if (MDNS.begin("alternator")) {
      MDNS.addService("http", "tcp", 80);
    }
    return true;
  }
  
  return false;
}
```

## Intelligent Reconnection System

### Exponential Backoff Algorithm

```cpp
struct WiFiReconnection {
  unsigned long lastAttempt = 0;
  int attemptCount = 0;
  unsigned long currentInterval = 2000;      // Start at 2 seconds
  const unsigned long minInterval = 2000;    // 2 seconds minimum
  const unsigned long maxInterval = 300000;  // 5 minutes maximum
  const int maxAttempts = 20;                // Give up after 20 attempts
  bool giveUpMode = false;
  int lastSignalStrength = -999;             // Track signal strength
  const int minSignalThreshold = -80;        // Don't retry aggressively if weak
} wifiRecon;

void checkWiFiConnection() {
  if (currentWiFiMode != AWIFI_MODE_CLIENT) return;
  
  if (WiFi.status() == WL_CONNECTED) {
    wifiRecon.lastSignalStrength = WiFi.RSSI();
    if (wifiRecon.attemptCount > 0) {
      queueConsoleMessage("WiFi reconnected after " + String(wifiRecon.attemptCount) + " attempts");
    }
    // Reset reconnection state on success
    wifiRecon.attemptCount = 0;
    wifiRecon.currentInterval = wifiRecon.minInterval;
    wifiRecon.giveUpMode = false;
    return;
  }
  
  // Check give-up mode
  if (wifiRecon.giveUpMode) {
    if (millis() - wifiRecon.lastAttempt < 300000) return;  // 5 minutes
    wifiRecon.giveUpMode = false;
    wifiRecon.attemptCount = 0;
    wifiRecon.currentInterval = wifiRecon.minInterval;
  }
  
  // Signal strength awareness
  if (wifiRecon.lastSignalStrength < wifiRecon.minSignalThreshold) {
    if (millis() - wifiRecon.lastAttempt < 60000) return;  // 1 minute for weak signal
  } else {
    if (millis() - wifiRecon.lastAttempt < wifiRecon.currentInterval) return;
  }
  
  wifiRecon.lastAttempt = millis();
  wifiRecon.attemptCount++;
  
  // Load credentials and attempt connection
  String saved_ssid = readFile(LittleFS, WIFI_SSID_FILE);
  String saved_password = readFile(LittleFS, WIFI_PASS_FILE);
  saved_ssid.trim();
  saved_password.trim();
  
  bool connected = connectToWiFi(saved_ssid.c_str(), saved_password.c_str(), 10000);
  
  if (!connected) {
    if (wifiRecon.attemptCount >= wifiRecon.maxAttempts) {
      queueConsoleMessage("WiFi: Max reconnection attempts reached, will retry in 5 minutes");
      wifiRecon.giveUpMode = true;
      return;
    }
    
    // Exponential backoff with caps
    if (wifiRecon.currentInterval < 32000) {
      wifiRecon.currentInterval *= 2;
    } else if (wifiRecon.currentInterval < 60000) {
      wifiRecon.currentInterval = 60000;
    } else {
      wifiRecon.currentInterval = wifiRecon.maxInterval;
    }
  }
}
```

**Reconnection Strategy**:
- **Progressive delays**: 2s, 4s, 8s, 16s, 32s, 60s, 120s, 300s (max)
- **Signal awareness**: Longer delays for weak signal conditions
- **Give-up mode**: 5-minute pause after 20 failed attempts
- **Automatic recovery**: Fresh attempt burst after give-up period

### Disconnect Detection

```cpp
// Simple ping method for disconnect detection
static unsigned long lastPingTime = 0;
static unsigned long lastDisconnectCheck = 0;
static bool alreadyCountedDisconnect = false;

if (millis() - lastDisconnectCheck > 10000) {  // Check every 10 seconds
  lastDisconnectCheck = millis();
  if (millis() - lastPingTime > 25000 && lastPingTime > 0) {
    // WiFi is disconnected
    if (!alreadyCountedDisconnect) {
      wifiDisconnectCount++;
      wifiReconnectsTotal++;
      queueConsoleMessage("WiFi disconnect #" + String(wifiDisconnectCount));
      alreadyCountedDisconnect = true;
    }
  } else if (lastPingTime > 0 && (millis() - lastPingTime < 25000)) {
    alreadyCountedDisconnect = false;  // Reset for next disconnect
  }
}

// Ping handler updates lastPingTime
server.on("/ping", HTTP_GET, [](AsyncWebServerRequest *request) {
  lastPingTime = millis();
  request->send(200, "text/plain", "ok");
});
```

## Web Server Implementation

### Server Setup

```cpp
AsyncWebServer server(80);
AsyncEventSource events("/events");

void setupServer() {
  // Main interface
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    request->send(LittleFS, "/index.html", "text/html");
  });
  
  // Settings handler (master endpoint for all configuration)
  server.on("/get", HTTP_GET, [](AsyncWebServerRequest *request) {
    if (!request->hasParam("password") || 
        strcmp(request->getParam("password")->value().c_str(), requiredPassword) != 0) {
      request->send(403, "text/plain", "Forbidden");
      return;
    }
    
    // Process 80+ different settings parameters
    bool foundParameter = false;
    String inputMessage;
    
    if (request->hasParam("TemperatureLimitF")) {
      foundParameter = true;
      inputMessage = request->getParam("TemperatureLimitF")->value();
      writeFile(LittleFS, "/TemperatureLimitF.txt", inputMessage.c_str());
      TemperatureLimitF = inputMessage.toInt();
    } 
    // ... 80+ more parameter handlers
    
    request->send(200, "text/plain", inputMessage);
  });
  
  // Password management
  server.on("/setPassword", HTTP_POST, [](AsyncWebServerRequest *request) {
    // Validate current password and set new one
  });
  
  server.on("/checkPassword", HTTP_POST, [](AsyncWebServerRequest *request) {
    // Validate password for access control
  });
  
  // File serving with proper MIME types
  server.onNotFound([](AsyncWebServerRequest *request) {
    String path = request->url();
    if (LittleFS.exists(path)) {
      String contentType = getContentType(path);
      request->send(LittleFS, path, contentType);
    } else {
      request->redirect("http://" + WiFi.softAPIP().toString());
    }
  });
  
  // Event source for real-time updates
  events.onConnect([](AsyncEventSourceClient *client) {
    client->send("hello!", NULL, millis(), 10000);
  });
  
  server.addHandler(&events);
  server.begin();
}
```

### Settings Management System

The system implements a massive settings handler supporting 80+ parameters:

```cpp
// Example parameter handling in /get endpoint
if (request->hasParam("BulkVoltage")) {
  foundParameter = true;
  inputMessage = request->getParam("BulkVoltage")->value();
  writeFile(LittleFS, "/BulkVoltage.txt", inputMessage.c_str());
  BulkVoltage = inputMessage.toFloat();
} else if (request->hasParam("TargetAmps")) {
  foundParameter = true;
  inputMessage = request->getParam("TargetAmps")->value();
  writeFile(LittleFS, "/TargetAmps.txt", inputMessage.c_str());
  TargetAmps = inputMessage.toInt();
}
// ... continues for all parameters
```

**Settings Architecture**:
- **Single endpoint**: `/get` handles all parameter updates
- **Password protection**: All changes require authentication
- **Immediate persistence**: Settings saved to LittleFS files immediately
- **Live updates**: Changes take effect without restart
- **Parameter validation**: Type conversion and bounds checking

## Real-Time Data Streaming

### SendWifiData() Architecture

The core data transmission function implements a priority-based streaming system:

```cpp
void SendWifiData() {
  static unsigned long prev_millis5 = 0;            // CSVData timing
  static unsigned long lastConsoleMessageTime = 0;  // Console timing
  static unsigned long lastpayload2send = 0;        // CSVData2 timing
  static unsigned long lastpayload3send = 0;        // CSVData3 timing
  static unsigned long lastTimestampSend = 0;       // TimestampData timing
  static unsigned long lastEventSourceSend = 0;     // Global cooldown
  
  const unsigned long EVENTSOURCE_COOLDOWN = 8;     // 8ms between sends
  const unsigned long CONSOLE_MESSAGE_INTERVAL = 1000;
  
  unsigned long now = millis();
  
  // Global cooldown check
  bool canSendNow = (now - lastEventSourceSend >= EVENTSOURCE_COOLDOWN);
  if (!canSendNow) return;
  
  bool sentSomething = false;
  
  // PRIORITY 1: CSVData (real-time sensor data)
  if (!sentSomething && now - prev_millis5 >= webgaugesinterval && events.count() > 0) {
    static char payload1[420];
    snprintf(payload1, sizeof(payload1),
             "%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d",
             SafeInt(AlternatorTemperatureF),  // 0
             SafeInt(dutyCycle),               // 1
             SafeInt(BatteryV, 100),           // 2
             // ... 35 total values
    );
    events.send(payload1, "CSVData");
    prev_millis5 = now;
    lastEventSourceSend = now;
    sentSomething = true;
  }
  
  // PRIORITY 2: Console messages
  if (!sentSomething && now - lastConsoleMessageTime >= CONSOLE_MESSAGE_INTERVAL && consoleCount > 0) {
    int messagesToSend = min(2, consoleCount);
    for (int i = 0; i < messagesToSend; i++) {
      if (consoleCount > 0) {
        events.send(consoleQueue[consoleTail].message, "console");
        consoleTail = (consoleTail + 1) % CONSOLE_QUEUE_SIZE;
        consoleCount--;
      }
    }
    lastConsoleMessageTime = now;
    lastEventSourceSend = now;
    sentSomething = true;
  }
  
  // PRIORITY 3: CSVData2 (status data - every 2 seconds)
  if (!sentSomething && now - lastpayload2send >= 2000) {
    static char payload2[700];
    // 85 status values including SOC, energy, alarms, etc.
    events.send(payload2, "CSVData2");
    lastpayload2send = now;
    lastEventSourceSend = now;
    sentSomething = true;
  }
  
  // PRIORITY 4: CSVData3 (settings data - every 2 seconds)  
  if (!sentSomething && now - lastpayload3send >= 2000) {
    static char payload3[1000];
    // 149 configuration values including all settings
    events.send(payload3, "CSVData3");
    lastpayload3send = now;
    lastEventSourceSend = now;
    sentSomething = true;
  }
  
  // PRIORITY 5: TimestampData (staleness data - every 3 seconds)
  if (!sentSomething && now - lastTimestampSend >= 3000) {
    static char timestampPayload[400];
    // Data freshness timestamps for all 17 sensor channels
    events.send(timestampPayload, "TimestampData");
    lastTimestampSend = now;
    lastEventSourceSend = now;
    sentSomething = true;
  }
}
```

### Data Payload Structure

#### CSVData (Priority 1 - Real-time)
**Update Rate**: User configurable (50ms default)
**Size**: 35 values
**Content**: Core sensor readings
- Temperature, duty cycle, voltages, currents
- RPM, timing data, WiFi strength
- Plot control parameters

#### CSVData2 (Priority 3 - Status)
**Update Rate**: 2 seconds
**Size**: 85 values  
**Content**: Status and monitoring data
- SOC, energy totals, runtime statistics
- Temperature maximums, alarm states
- Weather data, system health metrics

#### CSVData3 (Priority 4 - Settings)
**Update Rate**: 2 seconds
**Size**: 149 values
**Content**: All configuration parameters
- Basic settings (voltage, current targets)
- Advanced settings (PID, learning parameters)
- RPM tables, learning diagnostics

#### TimestampData (Priority 5 - Staleness)
**Update Rate**: 3 seconds
**Size**: 17 values
**Content**: Data freshness indicators
- Time since last update for each sensor
- Stale data detection for web interface
- Sensor failure indication

### Data Scaling and Encoding

The system uses consistent scaling factors for transmission:

```cpp
int SafeInt(float f, int scale = 1) {
  return isnan(f) || isinf(f) ? -1 : (int)(f * scale);
}

// Common scaling patterns:
SafeInt(BatteryV, 100),           // Voltage × 100 (13.75V → 1375)
SafeInt(MeasuredAmps, 100),       // Current × 100 (42.5A → 4250)  
SafeInt(iiout, 10),               // Field current × 10 (2.35A → 235)
SafeInt(DynamicShuntGainFactor, 1000),  // Gain × 1000 (1.025 → 1025)
SafeInt(rpmCurrentTable[0], 100), // Learning table × 100 (45.5A → 4550)
```

**Scaling Benefits**:
- **Integer transmission**: Avoids floating point parsing in JavaScript
- **Precision preservation**: Maintains 2-3 decimal places
- **Error indication**: -1 value indicates invalid/NaN data
- **Bandwidth efficiency**: Smaller payload size than JSON

## Security System

### Password Management

```cpp
// Password storage and validation
char requiredPassword[32] = "admin";  // Default password
char storedPasswordHash[65] = {0};    // SHA-256 hash storage

void loadPasswordHash() {
  // Load plaintext password for auth
  if (LittleFS.exists("/password.txt")) {
    File plainFile = LittleFS.open("/password.txt", "r");
    String pwdStr = plainFile.readStringUntil('\n');
    pwdStr.trim();
    strncpy(requiredPassword, pwdStr.c_str(), sizeof(requiredPassword) - 1);
    plainFile.close();
  }
  
  // Load hash for future use
  if (LittleFS.exists("/password.hash")) {
    File file = LittleFS.open("/password.hash", "r");
    size_t len = file.readBytesUntil('\n', storedPasswordHash, sizeof(storedPasswordHash) - 1);
    storedPasswordHash[len] = '\0';
    file.close();
  } else {
    // Create default hash
    strncpy(requiredPassword, "admin", sizeof(requiredPassword) - 1);
    sha256("admin", storedPasswordHash);
  }
}

bool validatePassword(const char *password) {
  if (!password) return false;
  char hash[65] = {0};
  sha256(password, hash);
  return (strcmp(hash, storedPasswordHash) == 0);
}
```

### SHA-256 Implementation

```cpp
void sha256(const char *input, char *outputBuffer) {
  byte shaResult[32];
  mbedtls_md_context_t ctx;
  const mbedtls_md_info_t *info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  
  mbedtls_md_init(&ctx);
  mbedtls_md_setup(&ctx, info, 0);
  mbedtls_md_starts(&ctx);
  mbedtls_md_update(&ctx, (const unsigned char *)input, strlen(input));
  mbedtls_md_finish(&ctx, shaResult);
  mbedtls_md_free(&ctx);
  
  for (int i = 0; i < 32; ++i) {
    sprintf(outputBuffer + (i * 2), "%02x", shaResult[i]);
  }
}
```

### Access Control

All configuration changes require password validation:

```cpp
server.on("/get", HTTP_GET, [](AsyncWebServerRequest *request) {
  if (!request->hasParam("password") || 
      strcmp(request->getParam("password")->value().c_str(), requiredPassword) != 0) {
    request->send(403, "text/plain", "Forbidden");
    return;
  }
  // Process setting changes...
});
```

## WiFi Configuration System

### Configuration Interface

```cpp
void setupWiFiConfigServer() {
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    request->send_P(200, "text/html", WIFI_CONFIG_HTML);
  });
  
  server.on("/wifi", HTTP_POST, [](AsyncWebServerRequest *request) {
    String ssid = "";
    String password = "";
    String ap_password = "";
    String hotspot_ssid = "";
    
    // Extract parameters
    if (request->hasParam("ssid", true)) {
      ssid = request->getParam("ssid", true)->value();
      ssid.trim();
    }
    if (request->hasParam("password", true)) {
      password = request->getParam("password", true)->value();
      password.trim();
    }
    if (request->hasParam("ap_password", true)) {
      ap_password = request->getParam("ap_password", true)->value();
      ap_password.trim();
    }
    if (request->hasParam("hotspot_ssid", true)) {
      hotspot_ssid = request->getParam("hotspot_ssid", true)->value();
      hotspot_ssid.trim();
    }
    
    // Apply defaults
    if (ap_password.length() == 0) {
      ap_password = "alternator123";
    } else if (ap_password.length() < 8) {
      ap_password = "alternator123";  // Security requirement
    }
    
    // Save configuration
    writeFile(LittleFS, AP_PASSWORD_FILE, ap_password.c_str());
    esp32_ap_password = ap_password;
    
    if (hotspot_ssid.length() > 0) {
      writeFile(LittleFS, AP_SSID_FILE, hotspot_ssid.c_str());
      esp32_ap_ssid = hotspot_ssid;
    }
    
    writeFile(LittleFS, "/ssid.txt", ssid.c_str());
    writeFile(LittleFS, "/pass.txt", password.c_str());
    
    request->send(200, "text/plain", "Configuration saved! Device will restart in 3 seconds.");
    delay(3000);
    ESP.restart();
  });
  
  server.begin();
}
```

### Captive Portal Implementation

```cpp
void setupAccessPoint() {
  WiFi.mode(WIFI_AP);
  WiFi.softAP(esp32_ap_ssid.c_str(), esp32_ap_password.c_str());
  
  // DNS server redirects all requests to AP IP
  dnsServer.start(DNS_PORT, "*", WiFi.softAPIP());
}

void dnsHandleRequest() {
  if (currentWiFiMode == AWIFI_MODE_AP) {
    dnsServer.processNextRequest();
  }
}

// 404 handler redirects to captive portal
server.onNotFound([](AsyncWebServerRequest *request) {
  if (LittleFS.exists(request->url())) {
    // Serve file if it exists
    request->send(LittleFS, request->url(), getContentType(request->url()));
  } else {
    // Redirect to captive portal
    request->redirect("http://" + WiFi.softAPIP().toString());
  }
});
```

## Over-The-Air (OTA) Updates

### OTA System Architecture

```cpp
const char *OTA_SERVER_URL = "https://ota.xengineering.net";
const char *FIRMWARE_VERSION = "1.9.0";

// Server certificate for HTTPS validation
const char *server_root_ca = "-----BEGIN CERTIFICATE-----\n"
  "MIIFBTCCAu2gAwIBAgIQS6hSk/eaL6JzBkuoBI110DANBgkqhkiG9w0BAQsFADBP\n"
  // ... certificate content
  "-----END CERTIFICATE-----\n";

// RSA public key for firmware signature verification
const char *OTA_PUBLIC_KEY = "-----BEGIN PUBLIC KEY-----\n"
  "MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAp2sRgMjD4wazKHo6Rk3g\n"
  // ... public key content
  "-----END PUBLIC KEY-----\n";
```

### Update Process

```cpp
void checkForOTAUpdate() {
  HTTPClient http;
  WiFiClientSecure client;
  client.setCACert(server_root_ca);
  
  String url = String(OTA_SERVER_URL) + "/api/firmware/check.php";
  http.begin(client, url);
  
  // Add device identification headers
  http.addHeader("Device-ID", getDeviceId());
  http.addHeader("Current-Version", FIRMWARE_VERSION);
  http.addHeader("Hardware-Version", "ESP32-WROOM-32");
  
  int httpCode = http.GET();
  
  if (httpCode == 200) {
    String response = http.getString();
    UpdateInfo updateInfo = parseUpdateResponse(response);
    
    if (updateInfo.hasUpdate) {
      performOTAUpdate(updateInfo);
    }
  }
  
  http.end();
}
```

### Streaming TAR Extraction

The OTA system implements streaming extraction of TAR packages:

```cpp
struct StreamingExtractor {
  bool inTarHeader;
  uint8_t tarHeader[512];
  size_t tarHeaderPos;
  String currentFileName;
  size_t currentFileSize;
  bool isCurrentFileFirmware;
  esp_ota_handle_t otaHandle;
  const esp_partition_t* otaPartition;
  bool otaStarted;
  mbedtls_md_context_t hashCtx;  // For signature verification
};

bool processDataChunk(StreamingExtractor* extractor, uint8_t* data, size_t dataSize) {
  size_t processed = 0;
  
  while (processed < dataSize) {
    if (extractor->inTarHeader) {
      // Read TAR header
      size_t headerRemaining = 512 - extractor->tarHeaderPos;
      size_t toCopy = min(headerRemaining, dataSize - processed);
      
      memcpy(extractor->tarHeader + extractor->tarHeaderPos, data + processed, toCopy);
      extractor->tarHeaderPos += toCopy;
      processed += toCopy;
      
      if (extractor->tarHeaderPos >= 512) {
        if (!parseTarHeader(extractor)) return false;
        extractor->inTarHeader = false;
      }
    } else {
      // Read file data
      size_t fileRemaining = extractor->currentFileSize - extractor->currentFilePos;
      size_t toWrite = min(fileRemaining, dataSize - processed);
      
      if (extractor->isCurrentFileFirmware && extractor->otaStarted) {
        esp_err_t err = esp_ota_write(extractor->otaHandle, data + processed, toWrite);
        if (err != ESP_OK) return false;
      }
      
      extractor->currentFilePos += toWrite;
      processed += toWrite;
    }
  }
  
  return true;
}
```

### Signature Verification

```cpp
bool verifyPackageSignature(uint8_t* packageData, size_t packageSize, const String& signatureBase64) {
  mbedtls_pk_context pk;
  uint8_t signature[520];
  size_t sigLength;
  
  // Decode signature
  if (!base64Decode(signatureBase64, signature, sizeof(signature), &sigLength)) {
    return false;
  }
  
  // Hash package
  uint8_t hash[32];
  mbedtls_md_context_t ctx;
  mbedtls_md_init(&ctx);
  const mbedtls_md_info_t* info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  
  mbedtls_md_setup(&ctx, info, 0);
  mbedtls_md_starts(&ctx);
  mbedtls_md_update(&ctx, packageData, packageSize);
  mbedtls_md_finish(&ctx, hash);
  mbedtls_md_free(&ctx);
  
  // Verify signature with RSA public key
  mbedtls_pk_init(&pk);
  int ret = mbedtls_pk_parse_public_key(&pk, (const unsigned char*)OTA_PUBLIC_KEY, strlen(OTA_PUBLIC_KEY) + 1);
  if (ret != 0) {
    mbedtls_pk_free(&pk);
    return false;
  }
  
  ret = mbedtls_pk_verify(&pk, MBEDTLS_MD_SHA256, hash, 32, signature, sigLength);
  mbedtls_pk_free(&pk);
  
  return (ret == 0);
}
```

## Console Message System

### Message Queue Implementation

```cpp
#define CONSOLE_QUEUE_SIZE 10
struct ConsoleMessage {
  char message[128];
  unsigned long timestamp;
};

ConsoleMessage consoleQueue[CONSOLE_QUEUE_SIZE];
volatile int consoleHead = 0;
volatile int consoleTail = 0;
volatile int consoleCount = 0;

void queueConsoleMessage(String message) {
  // Prevent stack overflow
  if (message.length() > 120) {
    message = message.substring(0, 120) + "...";
  }
  
  // Thread-safe circular buffer
  int nextHead = (consoleHead + 1) % CONSOLE_QUEUE_SIZE;
  int localTail = consoleTail;
  
  if (nextHead != localTail) {  // Not full
    strncpy(consoleQueue[consoleHead].message, message.c_str(), 127);
    consoleQueue[consoleHead].message[127] = '\0';
    consoleQueue[consoleHead].timestamp = millis();
    consoleHead = nextHead;
    
    int localCount = consoleCount;
    if (localCount < CONSOLE_QUEUE_SIZE) {
      consoleCount = localCount + 1;
    }
  }
  // If full, oldest message is overwritten
}
```

**Console Features**:
- **Fixed-size buffer**: Prevents memory exhaustion
- **Thread-safe**: Atomic operations for interrupt safety
- **Overflow handling**: Oldest messages discarded when full
- **Timestamp tracking**: Message age for debugging
- **Length limiting**: Prevents buffer overruns

## Performance Characteristics

### Network Bandwidth Usage

| Data Stream | Update Rate | Payload Size | Bandwidth |
|-------------|-------------|--------------|-----------|
| CSVData | 50ms (configurable) | 420 bytes | ~8.4 KB/s |
| Console | 1 second | ~50 bytes/msg | ~0.1 KB/s |
| CSVData2 | 2 seconds | 700 bytes | ~0.35 KB/s |
| CSVData3 | 2 seconds | 1000 bytes | ~0.5 KB/s |
| TimestampData | 3 seconds | 400 bytes | ~0.13 KB/s |
| **Total** | **Combined** | **Variable** | **~9.5 KB/s** |

### Memory Usage

| Component | RAM Usage | Flash Usage |
|-----------|-----------|-------------|
| WiFi stack | ~15 KB | ~200 KB |
| Web server | ~8 KB | ~50 KB |
| TLS/crypto | ~20 KB | ~150 KB |
| Console queue | 1.3 KB | - |
| Data buffers | 2.5 KB | - |
| **Total** | **~47 KB** | **~400 KB** |

### Client Connection Limits

- **Concurrent connections**: 5-10 tested
- **EventSource streams**: Limited by ESP32 memory
- **HTTP requests**: No artificial limits
- **WebSocket**: Not implemented (EventSource preferred)

## Configuration Parameters

### Network Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| esp32_ap_ssid | "ALTERNATOR_WIFI" | Access point SSID |
| esp32_ap_password | "alternator123" | Access point password |
| webgaugesinterval | 50 | Real-time data update rate (ms) |
| plotTimeWindow | 120 | Plot time window (seconds) |
| EVENTSOURCE_COOLDOWN | 8 | Minimum time between sends (ms) |

### WiFi Reconnection

| Parameter | Default | Description |
|-----------|---------|-------------|
| WIFI_TIMEOUT | 20000 | Connection timeout (ms) |
| minInterval | 2000 | Minimum retry interval (ms) |
| maxInterval | 300000 | Maximum retry interval (ms) |
| maxAttempts | 20 | Attempts before give-up mode |
| minSignalThreshold | -80 | Signal strength threshold (dBm) |

### Security Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| requiredPassword | "admin" | Default admin password |
| Password length | 8+ chars | Minimum for AP password |
| Hash algorithm | SHA-256 | Password hashing method |
| Certificate validation | Enabled | HTTPS certificate checking |

## File System Integration

### LittleFS Web Files

```cpp
server.onNotFound([](AsyncWebServerRequest *request) {
  String path = request->url();
  
  if (LittleFS.exists(path)) {
    String contentType = "text/html";
    if (path.endsWith(".css")) contentType = "text/css";
    else if (path.endsWith(".js")) contentType = "application/javascript";
    else if (path.endsWith(".json")) contentType = "application/json";
    else if (path.endsWith(".png")) contentType = "image/png";
    
    request->send(LittleFS, path, contentType);
  } else {
    request->redirect("http://" + WiFi.softAPIP().toString());
  }
});
```

### Settings Persistence

All network configuration is stored in LittleFS:

```
/ssid.txt          - Client mode WiFi SSID
/pass.txt          - Client mode WiFi password  
/apssid.txt        - AP mode SSID override
/appass.txt        - AP mode password
/password.txt      - Web interface password (plaintext)
/password.hash     - Web interface password (SHA-256)
/wifimode.txt      - Permanent AP mode flag
```

## Troubleshooting Guide

### Common Network Issues

#### No WiFi Connection
```
Check:
1. GPIO pin states during boot
2. Saved credentials in /ssid.txt and /pass.txt
3. Signal strength and router compatibility
4. DHCP pool availability on router
5. MAC address filtering on router
```

#### Web Interface Not Loading
```
Check:
1. Correct IP address (check serial output)
2. mDNS resolution (try alternator.local)
3. Browser cache and cookies
4. Firewall blocking connections
5. LittleFS filesystem integrity
```

#### Real-time Data Not Updating
```
Check:
1. EventSource connection established
2. webgaugesinterval setting
3. JavaScript console errors
4. Network latency/packet loss
5. Multiple browser tabs competing
```

#### OTA Update Failures
```
Check:
1. Internet connectivity
2. Server certificate validation
3. Available flash space
4. Stable power supply during update
5. Firmware signature verification
```

### Diagnostic Tools

#### Network Status Monitoring
- WiFi signal strength display
- Connection attempt counting
- Reconnection statistics
- Ping response tracking

#### Performance Monitoring
- Loop time tracking
- Memory usage monitoring
- EventSource client counting
- Console message queue status

#### Security Monitoring
- Failed password attempts
- Certificate validation results
- Signature verification status
- Access control violations

The network system provides robust WiFi connectivity with intelligent reconnection, comprehensive web interface, and secure OTA update capabilities, supporting both standalone AP operation and integration with existing ship networks.