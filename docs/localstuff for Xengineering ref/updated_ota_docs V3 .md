# XRegulator Secure OTA System - Complete Documentation V5

## Table of Contents
1. [System Architecture & Key Decisions](#system-architecture--key-decisions)
2. [ESP32 Firmware & Partition Configuration](#esp32-firmware--partition-configuration)
3. [Build and Deploy Process](#build-and-deploy-process)
4. [Streaming Tar Parser Implementation](#streaming-tar-parser-implementation)
5. [Server Configuration](#server-configuration)
6. [Security Implementation](#security-implementation)
7. [Targeted Version Update System](#targeted-version-update-system)
8. [Troubleshooting & Common Issues](#troubleshooting--common-issues)
9. [Future Improvements](#future-improvements)

---

## System Architecture & Key Decisions

### Core Design Philosophy
- **Streaming tar extraction**: Handle large packages without memory allocation issues
- **Dual content updates**: Firmware (.bin) + web files (HTML/CSS/JS) in single package
- **Factory recovery system**: GPIO41-triggered fallback to known-good firmware
- **Military-grade security**: RSA-4096 signatures with air-gapped private key
- **Production reliability**: Comprehensive error handling and validation
- **User-controlled updates**: Manual version selection with web interface
- **Targeted version system**: Support for upgrades, downgrades, and specific version selection

### Technology Stack
- **ESP32**: 16MB flash with custom partition scheme
- **Package Format**: Uncompressed tar archives (ustar format)
- **Security**: RSA-4096 + SHA-256 signatures
- **Server**: PHP on CloudPanel/nginx with Let's Encrypt SSL
- **Build System**: Arduino CLI with custom deployment script
- **Version Management**: JSON-based version catalog with REST API

### File Structure Overview
```
/Users/joeceo/Documents/Arduino/Xregulator/         # Local development
├── Xregulator.ino                                  # Main ESP32 firmware
├── build_and_deploy.sh                            # Automated build/deploy script 
├── data/                                           # Web files for OTA
│   ├── index.html                                 # Main dashboard 
│   ├── script.js                                  # JavaScript logic
│   ├── styles.css                                 # Styling
│   ├── uPlot.iife.min.js                         # Charting library
│   └── uPlot.min.css                             # Chart styling
├── build/                                         # Arduino compilation output
└── partitions.csv                                # ESP32 partition scheme

ota.xengineering.net (Server)                     # Production OTA server
├── api/firmware/
│   ├── check.php                                 # Update availability & targeted requests
│   ├── download.php                              # Package download
│   ├── signature.php                             # Signature download
│   └── versions.php                              # Version list API
├── config/
│   ├── ota_config.php                           # Latest version & public key
│   ├── security.php                             # Rate limiting & validation
│   └── versions.json                            # Version metadata
├── firmware/                                     # Signed tar packages
├── signatures/                                   # Base64 RSA signatures
└── tmp/                                          # Rate limiting data

/Users/joeceo/private_key.pem                     # Air-gapped signing key
```

---

## ESP32 Firmware & Partition Configuration

### Partition Scheme (16MB Flash)
```csv
# Name,     Type, SubType,  Offset,   Size,     Flags
nvs,        data, nvs,      0x9000,   0x5000,
otadata,    data, ota,      0xe000,   0x2000,
factory,    app,  factory,  0x10000,  0x400000,  # 4MB - Factory firmware (GPIO41 recovery)
ota_0,      app,  ota_0,    0x410000, 0x400000,  # 4MB - Production firmware (OTA updates)
factory_fs, data, littlefs,   0x810000, 0x200000,  # 2MB - Factory web files (never change)
prod_fs,    data, littlefs,   0xa10000, 0x200000,  # 2MB - Production web files (OTA updated)
userdata,   data, littlefs,   0xc10000, 0x3D0000,  # ~4MB - Data logs, config
coredump,   data, coredump, 0xFF0000, 0x10000,   # Debug data
```

**Partition Strategy**:
- **Factory partitions**: Permanent "golden image" for GPIO41 recovery
- **OTA partitions**: Where updates are installed
- **4MB firmware partitions**: Current usage ~1.9MB, provides 2.1MB growth room
- **Separate filesystems**: Factory vs production web files
- **LittleFS mounting**: Factory firmware reads from prod_fs (0xA10000), not factory_fs (0x810000)

### Arduino IDE Configuration
```bash
# Required settings in Arduino IDE:
Board: "ESP32S3 Dev Module"
Flash Size: "16MB (128Mb)"
Partition Scheme: "Custom" (uses partitions.csv)
```

### Key Firmware Features
See server code directly - implementation is straightforward and well-commented.

---

## Build and Deploy Process

### Automated Build Script with Version Notes
Location: `/Users/joeceo/Documents/Arduino/Xregulator/build_and_deploy.sh`

```bash
# Usage with version notes
chmod +x build_and_deploy.sh
./build_and_deploy.sh 0.2.3 "Fixed targeted version downgrade system"
```

**Script Features**:
1. **Cleans build artifacts** and Arduino caches
2. **Compiles firmware** using Arduino CLI with custom FQBN
3. **Creates tar package** containing firmware.bin + data/ folder contents
4. **Signs package** with RSA-4096 private key
5. **Uploads to server** via sshpass/scp
6. **Updates server configuration** to new version
7. **Updates versions.json** with version notes and metadata
8. **Validates description length** (200 character limit with override)
9. **Tests deployment** with curl

### Version Notes and Validation
```bash
# Examples of deployment with version notes:
./build_and_deploy.sh 0.2.3 "Fixed targeted version request system"
./build_and_deploy.sh 0.2.4 "Improved battery SOC accuracy, added error handling"
./build_and_deploy.sh 0.3.0 "Major update: new PID controller and web interface improvements"

# Long descriptions trigger validation:
./build_and_deploy.sh 0.2.5 "Very long description that exceeds 200 characters will trigger a warning and allow you to continue or cancel the deployment process"
```

### Manual Build Process
```bash
# Navigate to project
cd /Users/joeceo/Documents/Arduino/Xregulator

# Clean previous builds
rm -rf build/ temp_package/ firmware_*.tar firmware_*.sig.*
arduino-cli cache clean

# Compile with custom partition scheme
arduino-cli compile --fqbn "esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom" --output-dir ./build .

# Verify compilation
ls -la build/Xregulator.ino.bin
echo "Firmware size: $(wc -c < build/Xregulator.ino.bin) bytes"

# Create package
mkdir temp_package
cp build/Xregulator.ino.bin temp_package/firmware.bin
cp -r data/* temp_package/
tar --format=ustar -cf "firmware_0.2.3.tar" -C temp_package .

# Sign package (air-gapped machine)
openssl dgst -sha256 -sign /Users/joeceo/private_key.pem -out firmware_0.2.3.sig.binary firmware_0.2.3.tar
base64 -i firmware_0.2.3.sig.binary | tr -d '\n' > firmware_0.2.3.sig.base64
```

### Manual LittleFS Flashing Commands
```bash
# Flash to prod_fs partition (NOT factory_fs)
cd /Users/joeceo/Documents/Arduino/Xregulator
ls data/
du -sh data/
~/Tools/mklittlefs/mklittlefs -c data -s 0x200000 littlefs.bin
ls -lh littlefs.bin
esptool.py --chip esp32s3 --port /dev/cu.usbmodem2101 write_flash 0xA10000 littlefs.bin
```

**Note**: Factory firmware reads web files from prod_fs partition (0xA10000), not factory_fs (0x810000).

### Server Deployment
```bash
# Upload firmware package
sshpass -p "PASSWORD" scp firmware_0.2.3.tar root@host.xengineering.net:/home/ota/htdocs/ota.xengineering.net/firmware/

# Upload signature
sshpass -p "PASSWORD" ssh root@host.xengineering.net "echo '$(cat firmware_0.2.3.sig.base64)' > /home/ota/htdocs/ota.xengineering.net/signatures/firmware_0.2.3.sig"

# Update server version and versions.json
sshpass -p "PASSWORD" ssh root@host.xengineering.net "
  cd /home/ota/htdocs/ota.xengineering.net
  sed -i \"s/CURRENT_FIRMWARE_VERSION', '[^']*'/CURRENT_FIRMWARE_VERSION', '0.2.3'/\" config/ota_config.php
  rm -f tmp/rate_limits.json  # Clear rate limits for testing
"

# Test deployment
curl -s -H "Device-ID: TEST123" -H "Current-Version: 0.2.2" -H "Hardware-Version: ESP32-S3" https://ota.xengineering.net/api/firmware/check.php | python3 -m json.tool
```

---

## Streaming Tar Parser Implementation

### Key Technical Achievement
The streaming tar parser was the critical breakthrough that made this system work. It solves the fundamental problem of processing large firmware packages on memory-constrained ESP32 devices.

### Problem Solved
- **Memory limitation**: ESP32 couldn't load 1.8MB+ packages into RAM
- **Dual content**: Need to extract both firmware.bin and web files
- **Streaming requirement**: Process tar archive as it downloads
- **Padding complexity**: Tar files use 512-byte alignment with padding

### Implementation
See server firmware code directly - the streaming parser is well-documented and handles tar files by processing them in 1KB chunks as they download, routing firmware.bin to the OTA partition and web files to LittleFS, while properly managing 512-byte tar padding across chunk boundaries.

---

## Server Configuration

### PHP-based OTA Server
**Location**: `ota.xengineering.net` (CloudPanel/nginx)
**Path**: `/home/ota/htdocs/ota.xengineering.net/`

### API Endpoints

#### Update Check: `/api/firmware/check.php`

**Purpose**: Check for available updates and handle targeted version requests

**Standard Update Check (Latest Version)**:
```bash
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.2" \
     -H "Hardware-Version: ESP32-S3" \
     https://ota.xengineering.net/api/firmware/check.php
```

**Response (when newer version available)**:
```json
{
  "hasUpdate": true,
  "version": "0.2.4",
  "firmwareUrl": "https://ota.xengineering.net/api/firmware/download.php?version=0.2.4",
  "signatureUrl": "https://ota.xengineering.net/api/firmware/signature.php?version=0.2.4",
  "firmwareSize": 1932288,
  "changelog": "Update to version 0.2.4",
  "releaseDate": "2025-10-25T14:22:06-04:00"
}
```

**Response (when no update available)**:
```
HTTP 204 No Content
```

**Targeted Version Request (Upgrade/Downgrade)**:
```bash
# Request specific version via URL parameter
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.4" \
     -H "Hardware-Version: ESP32-S3" \
     "https://ota.xengineering.net/api/firmware/check.php?requestedVersion=0.2.1"
```

**Response (targeted version)**:
```json
{
  "hasUpdate": true,
  "version": "0.2.1",
  "firmwareUrl": "https://ota.xengineering.net/api/firmware/download.php?version=0.2.1",
  "signatureUrl": "https://ota.xengineering.net/api/firmware/signature.php?version=0.2.1",
  "firmwareSize": 1945600,
  "changelog": "Update to version 0.2.1",
  "releaseDate": "2025-10-25T14:08:53-04:00"
}
```

**Error Responses**:
```json
// HTTP 400 - Invalid parameters
{"error": "Invalid requested version format"}

// HTTP 404 - Version not found
{"error": "Requested firmware version not available"}

// HTTP 429 - Rate limited
{"error": "Rate limit exceeded. Please try again later."}
```

**Implementation Logic**:
1. If `requestedVersion` parameter present → serve that specific version
2. If no parameter → serve latest version (from `CURRENT_FIRMWARE_VERSION`)
3. For latest version checks, return HTTP 204 if device is already up-to-date
4. For targeted requests, return 404 if requested version doesn't exist
5. Always validate version format and device ID
6. Apply rate limiting (10 checks per hour)

#### Version List API: `/api/firmware/versions.php`

**Purpose**: Provide list of all available firmware versions for user selection

```bash
curl https://ota.xengineering.net/api/firmware/versions.php
```

**Expected Response**:
```json
{
  "versions": {
    "0.2.4": {
      "notes": "Latest stable release",
      "release_date": "2025-10-25T14:22:06-04:00",
      "file_size": 1932288
    },
    "0.2.3": {
      "notes": "Fixed targeted version system",
      "release_date": "2025-10-25T13:17:25-04:00",
      "file_size": 1932288
    },
    "0.2.1": {
      "notes": "Previous stable version",
      "release_date": "2025-09-30T20:07:01-04:00",
      "file_size": 1945600
    }
  }
}
```

#### Package Download: `/api/firmware/download.php`

**Purpose**: Serve firmware packages with security validation

**Features**:
- **Security**: Verifies signature before serving
- **Headers Required**: Device-ID, Current-Version
- **Rate limiting**: 5 downloads per device per day
- **Content-Type**: `application/octet-stream`
- **Streaming**: Efficient file serving with readfile()

**Usage**:
```bash
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.2" \
     "https://ota.xengineering.net/api/firmware/download.php?version=0.2.3" \
     --output firmware.tar
```

#### Signature Download: `/api/firmware/signature.php`

**Purpose**: Serve firmware signatures for client-side verification

**Features**:
- **Format**: Base64-encoded RSA-4096 signature
- **Length**: 684-685 characters
- **Content-Type**: `text/plain`

**Usage**:
```bash
curl -H "Device-ID: TEST123" \
     "https://ota.xengineering.net/api/firmware/signature.php?version=0.2.3"
```

### Configuration Files

#### Version Metadata: `/config/versions.json`
```json
{
  "versions": {
    "0.2.4": {
      "notes": "Latest release with improvements",
      "release_date": "2025-10-25T14:22:06-04:00",
      "file_size": 1932288
    },
    "0.2.3": {
      "notes": "Fixed targeted version system",
      "release_date": "2025-10-25T13:17:25-04:00",
      "file_size": 1932288
    }
  }
}
```

**Purpose**: 
- Lists all available firmware versions
- Provides metadata for web interface display
- Updated automatically by build_and_deploy.sh script

#### Version Configuration: `/config/ota_config.php`
```php
<?php
// Latest firmware version - used for standard update checks
define('CURRENT_FIRMWARE_VERSION', '0.2.4');

// Minimum supported version (optional)
define('MIN_SUPPORTED_VERSION', '1.0.0');

// File paths
define('FIRMWARE_DIR', __DIR__ . '/../firmware/');
define('SIGNATURES_DIR', __DIR__ . '/../signatures/');

// RSA-4096 Public Key for signature verification
define('OTA_PUBLIC_KEY', '-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAp2sRgMjD4wazKHo6Rk3g
[... full RSA-4096 public key ...]
-----END PUBLIC KEY-----');
?>
```

#### Security Configuration: `/config/security.php`
```php
<?php
// Rate limiting configuration
define('RATE_LIMIT_CHECKS_PER_HOUR', 10);
define('RATE_LIMIT_DOWNLOADS_PER_DAY', 5);

// Security validation functions
class OTASecurity {
    public static function validateDeviceId($deviceId);
    public static function validateVersion($version);
    public static function checkRateLimit($deviceId, $type);
    public static function getFirmwareInfo($version);
    public static function verifyFirmwareSignature($data, $signature);
    public static function compareVersions($v1, $v2);
}
?>
```

### Server Maintenance Commands
```bash
# SSH into server
ssh root@host.xengineering.net

# Check OTA server status
cd /home/ota/htdocs/ota.xengineering.net
ls -la firmware/        # Check firmware files
ls -la signatures/      # Check signature files
cat config/ota_config.php | grep CURRENT_FIRMWARE_VERSION

# Check version metadata
cat config/versions.json | python3 -m json.tool

# Clear rate limits for testing
rm -f tmp/rate_limits.json

# Check server logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Check disk space
df -h /home/ota/
du -sh firmware/        # Firmware storage usage

# Test targeted version request
curl -H "Device-ID: TEST123" -H "Current-Version: 0.2.4" \
  "https://ota.xengineering.net/api/firmware/check.php?requestedVersion=0.2.1" | python3 -m json.tool
```

---

## Security Implementation

### RSA-4096 Cryptographic Security
- **Algorithm**: RSA-4096 with SHA-256 hashing
- **Key storage**: Private key air-gapped at `/Users/joeceo/private_key.pem`
- **Signature size**: 512 bytes (684-685 characters base64)
- **Security level**: Military-grade (future-proof against quantum computing)

### Air-Gapped Signing Process
```bash
# On offline machine only:
cd /Users/joeceo/Documents/Arduino/Xregulator

# Sign firmware package
openssl dgst -sha256 -sign /Users/joeceo/private_key.pem -out firmware_0.2.3.sig.binary firmware_0.2.3.tar

# Convert to base64 for server upload
base64 -i firmware_0.2.3.sig.binary | tr -d '\n' > firmware_0.2.3.sig.base64

# Display signature for manual server upload
cat firmware_0.2.3.sig.base64
```

### SSL Certificate Validation
```cpp
// ESP32 firmware - Let's Encrypt R10 intermediate certificate
const char* server_root_ca =
  "-----BEGIN CERTIFICATE-----\n"
  "MIIFBTCCAu2gAwIBAgIQS6hSk/eaL6JzBkuoBI110DANBgkqhkiG9w0BAQsFADBP\n"
  // ... full certificate ...
  "-----END CERTIFICATE-----\n";

// Usage in ESP32 code
WiFiClientSecure client;
client.setCACert(server_root_ca);  // NEVER use client.setInsecure() in production!
```

### Signature Verification Process
See ESP32 firmware code - signature verification is implemented using mbedtls and occurs during the streaming download process, verifying the complete package hash against the RSA signature.

---

## Targeted Version Update System

### Overview
The system supports user-controlled firmware updates including upgrades, downgrades, and installation of any available version. This provides maximum flexibility for rollbacks, testing, and version management.

### Key Features
- **Targeted version requests**: Request any specific firmware version
- **Upgrade support**: Install newer versions
- **Downgrade support**: Rollback to previous versions
- **Version catalog**: Web interface displays all available versions
- **Dual update paths**: Different logic for factory vs OTA partition operation
- **Partition management**: Automatic switching between factory and ota_0 partitions
- **NVS state tracking**: Persistent update requests across reboots
- **GPIO41 override**: Emergency cancellation of stuck updates

### Web Interface Components

#### Software Update Tab
Location: `data/index.html` - Sub-tab under Settings

**Features**:
- Displays current firmware version and device ID
- Lists all available versions from server
- Allows selection of any version (upgrade or downgrade)
- Requires admin password authentication
- Provides update confirmation dialog
- Disabled in AP mode (requires internet connection)

#### Version Selection JavaScript
Location: `data/script.js`

**Key Functions**:
```javascript
// Load available versions from server
function loadAvailableVersions()

// Display versions with metadata
function displayAvailableVersions()

// Trigger update to specific version
function confirmUpdate(version)

// Decode and display current version from CSV data
function updateVersionInfo()
```

**Version Display Logic**:
```javascript
// Parse firmware version integer from CSV
const versionInt = parseInt(document.getElementById('firmwareVersionIntID').textContent);

// Decode to semantic version (major.minor.patch)
const major = Math.floor(versionInt / 10000);
const minor = Math.floor((versionInt % 10000) / 100);
const patch = versionInt % 100;
const versionStr = `${major}.${minor}.${patch}`;

// Example: 203 → 0.2.3, 10502 → 1.5.2
```

### ESP32 Implementation

#### HTTP Request Handler: `/get` Endpoint

**Purpose**: Handle version selection from web interface

**Parameters**:
- `password`: Admin authentication
- `UpdateToVersion`: Target firmware version (e.g., "0.2.1")

**Logic Flow**:
```cpp
if (request->hasParam("UpdateToVersion")) {
    String targetVersion = request->getParam("UpdateToVersion")->value();
    
    // Check which partition we're running on
    const esp_partition_t *running = esp_ota_get_running_partition();
    const esp_partition_t *factory = esp_partition_find_first(
        ESP_PARTITION_TYPE_APP, ESP_PARTITION_SUBTYPE_APP_FACTORY, NULL);
    
    if (running == factory) {
        // CASE 1: Running on factory partition
        // Perform update directly
        performOTAUpdateToVersion(targetVersion);
        
    } else {
        // CASE 2: Running on ota_0 partition
        // Store version in NVS and reboot to factory
        nvs_set_str(nvs_handle, "target_ver", targetVersion);
        nvs_set_u8(nvs_handle, "update_flag", 1);
        nvs_commit(nvs_handle);
        
        // Switch to factory partition
        esp_ota_set_boot_partition(factory);
        
        // Reboot
        ESP.restart();
    }
}
```

#### Targeted Version Request: `performOTAUpdateToVersion()`

**Purpose**: Request and download specific firmware version

**Implementation**:
```cpp
void performOTAUpdateToVersion(const char *targetVersion) {
    HTTPClient http;
    WiFiClientSecure client;
    client.setCACert(server_root_ca);
    
    // CRITICAL: Add requestedVersion as URL parameter
    String url = String(OTA_SERVER_URL) + 
                 "/api/firmware/check.php?requestedVersion=" + 
                 String(targetVersion);
    
    http.begin(client, url);
    http.addHeader("Device-ID", getDeviceId());
    http.addHeader("Current-Version", FIRMWARE_VERSION);
    http.addHeader("Hardware-Version", "ESP32-WROOM-32");
    
    int httpCode = http.GET();
    
    if (httpCode == 200) {
        String response = http.getString();
        UpdateInfo updateInfo = parseUpdateResponse(response);
        
        // Verify server returned the requested version
        if (updateInfo.version == String(targetVersion)) {
            // Proceed with download
            performOTAUpdate(updateInfo);
        } else {
            // Version mismatch - abort
            Serial.println("Version mismatch - requested " + 
                         String(targetVersion) + ", got " + 
                         updateInfo.version);
        }
    }
}
```

**Key Points**:
- Uses `?requestedVersion=X.X.X` URL parameter (NOT header)
- Verifies server response matches requested version
- Handles HTTP error codes (404, 429, etc.)
- Logs all steps for debugging

### Update Flow Scenarios

#### Scenario 1: Downgrade from OTA Partition
**User Goal**: Downgrade from 0.2.4 to 0.2.1 while running in ota_0

**Flow**:
```
1. Device running 0.2.4 in ota_0 partition
   └─> User selects "0.2.1" in web interface
   
2. ESP32 /get handler receives UpdateToVersion=0.2.1
   └─> Detects running in ota_0 (not factory)
   
3. Store target version in NVS
   └─> nvs_set_str("target_ver", "0.2.1")
   └─> nvs_set_u8("update_flag", 1)
   
4. Switch boot partition to factory
   └─> esp_ota_set_boot_partition(factory)
   
5. Reboot (may show backtrace - harmless)
   └─> ESP.restart()
   
6. Factory firmware boots up (0.2.4)
   └─> Detects update_flag in NVS
   └─> Reads target_ver: "0.2.1"
   
7. Request targeted version from server
   └─> GET /check.php?requestedVersion=0.2.1
   └─> Server responds with 0.2.1 metadata
   
8. Download firmware package
   └─> GET /download.php?version=0.2.1
   └─> Stream and extract to ota_0 partition
   
9. Download signature
   └─> GET /signature.php?version=0.2.1
   └─> Verify package integrity
   
10. Finalize OTA and reboot
    └─> esp_ota_set_boot_partition(ota_0)
    └─> ESP.restart()
    
11. Device boots into ota_0 running 0.2.1
    └─> Downgrade complete!
```

#### Scenario 2: Upgrade from Factory Partition
**User Goal**: Upgrade from 0.2.2 to 0.2.4 while running in factory

**Flow**:
```
1. Device running 0.2.2 in factory partition (GPIO41 triggered)
   └─> User selects "0.2.4" in web interface
   
2. ESP32 /get handler receives UpdateToVersion=0.2.4
   └─> Detects running in factory
   
3. Immediately call performOTAUpdateToVersion("0.2.4")
   └─> No NVS storage needed
   └─> No reboot required
   
4. Request targeted version from server
   └─> GET /check.php?requestedVersion=0.2.4
   └─> Server responds with 0.2.4 metadata
   
5. Download and install to ota_0
   └─> Stream firmware package
   └─> Verify signature
   └─> Write to ota_0 partition
   
6. Set ota_0 as boot partition and reboot
   └─> esp_ota_set_boot_partition(ota_0)
   └─> ESP.restart()
   
7. Device boots into ota_0 running 0.2.4
   └─> Upgrade complete!
```

#### Scenario 3: GPIO41 Emergency Override
**User Goal**: Cancel stuck update and return to factory mode

**Flow**:
```
1. Update requested but device stuck in boot loop
   └─> Update flag set in NVS
   
2. User holds GPIO41 low during power-on
   └─> Forces boot into factory partition
   
3. Factory firmware detects GPIO41
   └─> Clears update_flag in NVS
   └─> Clears target_ver in NVS
   
4. Device stays in factory mode
   └─> Update cancelled
   └─> User can try again or troubleshoot
```

#### Scenario 4: WiFi Failure During Update
**User Goal**: Recover from WiFi connection failure

**Flow**:
```
1. Update requested, device reboots to factory
   └─> Update flag set in NVS
   
2. Factory firmware boots but WiFi fails
   └─> MODE_AP instead of MODE_CLIENT
   
3. Update check never runs
   └─> Only runs in MODE_CLIENT mode
   └─> Device stays in AP mode
   
4. User can:
   a) Fix WiFi credentials and reboot
   b) Use GPIO41 to cancel update
   c) Wait for WiFi connection to restore
```

### Server-Side Implementation

#### Targeted Version Logic in check.php

```php
// Get target version from URL parameter
$requestedVersion = $_GET['requestedVersion'] ?? '';

if (!empty($requestedVersion)) {
    // TARGETED VERSION REQUEST
    
    // Validate version format
    if (!OTASecurity::validateVersion($requestedVersion)) {
        http_response_code(400);
        echo json_encode(['error' => 'Invalid requested version format']);
        exit;
    }
    
    // Use requested version as target
    $targetVersion = $requestedVersion;
    
    // Check if firmware exists
    $firmwareInfo = OTASecurity::getFirmwareInfo($targetVersion);
    if (!$firmwareInfo) {
        http_response_code(404);
        echo json_encode(['error' => 'Requested firmware version not available']);
        exit;
    }
    
    // Return requested version metadata
    $response = [
        'hasUpdate' => true,
        'version' => $targetVersion,
        'firmwareUrl' => $baseUrl . '/api/firmware/download.php?version=' . $targetVersion,
        'signatureUrl' => $baseUrl . '/api/firmware/signature.php?version=' . $targetVersion,
        'firmwareSize' => $firmwareInfo['size'],
        'changelog' => 'Update to version ' . $targetVersion,
        'releaseDate' => date('c')
    ];
    
} else {
    // STANDARD UPDATE CHECK (latest version)
    
    $targetVersion = CURRENT_FIRMWARE_VERSION;
    
    // Only return update if newer than current
    if (OTASecurity::compareVersions($targetVersion, $currentVersion) <= 0) {
        http_response_code(204); // No update available
        exit;
    }
    
    // Return latest version metadata
    // ... (same response structure)
}
```

**Key Points**:
- Checks for `requestedVersion` URL parameter
- If present: serves that specific version (no version comparison)
- If absent: serves latest version (only if newer than current)
- Returns 404 if requested version doesn't exist
- Always validates version format (semantic versioning)

### Version Encoding System

**Firmware Side** (C++):
```cpp
// Define version in firmware
const char* FIRMWARE_VERSION = "0.2.3";

// Encode to integer for CSV transmission
int major = 0, minor = 2, patch = 3;
firmwareVersionInt = major * 10000 + minor * 100 + patch;
// Result: 203

// Transmit via CSV to web interface
sprintf(csvBuffer, "%d", firmwareVersionInt);
```

**Web Interface Side** (JavaScript):
```javascript
// Receive version integer from CSV
const versionInt = 203;

// Decode to semantic version
const major = Math.floor(versionInt / 10000);      // 0
const minor = Math.floor((versionInt % 10000) / 100);  // 2
const patch = versionInt % 100;                    // 3

// Display as string
const versionStr = `${major}.${minor}.${patch}`;  // "0.2.3"
```

**Examples**:
```
Version String → Integer → Decoded String
"0.1.5"       → 105      → "0.1.5"
"0.2.3"       → 203      → "0.2.3"
"1.0.0"       → 10000    → "1.0.0"
"2.15.7"      → 21507    → "2.15.7"
```

### Testing Targeted Version System

#### Test 1: Verify Server Endpoint
```bash
# Request specific version
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.4" \
     "https://ota.xengineering.net/api/firmware/check.php?requestedVersion=0.2.1" \
     | python3 -m json.tool

# Expected: Returns 0.2.1 metadata (not 0.2.4)
```

#### Test 2: Verify Latest Version Endpoint
```bash
# No parameter - should return latest
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.2" \
     https://ota.xengineering.net/api/firmware/check.php \
     | python3 -m json.tool

# Expected: Returns 0.2.4 metadata (latest from CURRENT_FIRMWARE_VERSION)
```

#### Test 3: Verify Non-Existent Version
```bash
# Request version that doesn't exist
curl -H "Device-ID: TEST123" \
     -H "Current-Version: 0.2.2" \
     "https://ota.xengineering.net/api/firmware/check.php?requestedVersion=9.9.9" \
     | python3 -m json.tool

# Expected: HTTP 404 with error message
```

#### Test 4: End-to-End Downgrade
```
1. Ensure device is on version 0.2.4 in ota_0
2. Access web interface → Settings → Software Update
3. Click "Refresh Available Versions"
4. Select version 0.2.1 from list
5. Confirm update dialog
6. Monitor serial output:
   - Should see version mismatch detection (if any)
   - Should see factory boot
   - Should see targeted version request
   - Should see download progress
   - Should see signature verification
7. After reboot, verify version is 0.2.1
```

### Troubleshooting Targeted Updates

#### Version Mismatch Error
```
Error: "Version mismatch - requested 0.2.1, got 0.2.4"

Cause: Server returned wrong version
Fix: Check server check.php implementation
     Verify requestedVersion parameter is being read correctly
```

#### Version Not Found (404)
```
Error: "OTA: Version 0.2.1 not found on server"

Cause: Firmware file or signature missing
Fix: Check server filesystem:
     ls /home/ota/htdocs/ota.xengineering.net/firmware/firmware_0.2.1.tar
     ls /home/ota/htdocs/ota.xengineering.net/signatures/firmware_0.2.1.sig
```

#### Update Loops or Hangs
```
Error: Device keeps rebooting without completing update

Cause: Update flag stuck in NVS
Fix: Use GPIO41 override:
     1. Hold GPIO41 low during boot
     2. Factory firmware will clear NVS flags
     3. Device stays in factory mode
     4. Try update again or investigate issue
```

---

---

## Server Maintenance & Cleanup

### Complete OTA Version Removal

When you need to completely remove all firmware versions from the server (for testing, major updates, or starting fresh):

**Location**: `/home/ota/htdocs/ota.xengineering.net`
```bash
cd /home/ota/htdocs/ota.xengineering.net
set -e

# Backup config files
cp config/ota_config.php config/ota_config.php.$(date +%F_%H%M%S).bak
cp config/versions.json config/versions.json.$(date +%F_%H%M%S).bak

# Remove all firmware packages and signatures
rm -f firmware/firmware_*.tar signatures/firmware_*.sig tmp/rate_limits.json

# Clear the versions catalog (CRITICAL - devices cache this!)
echo '{"versions":{}}' > config/versions.json

# Reset version to 0.0.0
perl -0777 -i -pe "s/define\('CURRENT_FIRMWARE_VERSION',\s*'[^']*'\)/define('CURRENT_FIRMWARE_VERSION', '0.0.0')/" config/ota_config.php

# Verify cleanup
grep CURRENT_FIRMWARE_VERSION config/ota_config.php
cat config/versions.json
ls -al signatures/
ls -al firmware/

# Reload PHP-FPM to clear any opcode cache
systemctl list-units --type=service | awk '/php.*fpm/ {print $1}' | xargs -r -n1 systemctl reload
```

**Important**: Simply deleting firmware files and signatures is NOT enough. Devices cache the version list from `config/versions.json`, which is served by `/api/firmware/versions.php`. You must clear this file or devices will continue showing old versions in their UI.

**Verification**:
```bash
# Test that versions.json is empty
curl https://ota.xengineering.net/api/firmware/versions.php
# Should return: {"versions":{}}

# Test that check.php returns no update available
curl -X POST https://ota.xengineering.net/api/firmware/check.php \
  -H "Content-Type: application/json" \
  -d '{"current_version":"0.2.15","device_id":"test123"}'
# Should return: update_available: false
```

---

## Troubleshooting & Common Issues

### Build Issues

#### Arduino CLI Not Found
```bash
# Install Arduino CLI (macOS)
brew install arduino-cli

# Verify installation
arduino-cli version

# Install ESP32 board package
arduino-cli core install esp32:esp32
```

#### Compilation Errors
```bash
# Clear all caches
arduino-cli cache clean
rm -rf /tmp/arduino_*
rm -rf ~/Library/Arduino15/packages/esp32/hardware/esp32/cache

# Verify FQBN
arduino-cli compile --fqbn "esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom" --verify

# Check partition file
cat partitions.csv
```

### Runtime Issues

#### Memory Issues / Stack Overflow
```bash
# Monitor free heap during OTA
Serial.printf("Free heap: %d bytes\n", ESP.getFreeHeap());

# Reduce buffer sizes if needed (in ESP32 code)
const size_t CHUNK_SIZE = 512;  // Reduce from 1024 if memory issues
```

#### "Streaming extraction failed"
Ensure end-of-archive detection returns success:
```cpp
// In processDataChunk() - verify this logic exists:
if (allZeros) {
  Serial.println("✅ End of tar archive detected - extraction complete!");
  return true; // CRITICAL: Must return true, not false!
}
```

### Targeted Update Issues

#### Web Interface Shows Wrong Version
**Symptom**: Display shows 0.2.1 but device is actually running 0.2.4

**Debug Steps**:
```cpp
// Check serial output on boot:
Serial.printf("Firmware Version Int: %d\n", firmwareVersionInt);
// Expected: 204 for version 0.2.4

// Check CSV transmission:
Serial.printf("CSV payload position 87-89: %d\n", firmwareVersionInt);

// Check web interface JavaScript console (F12):
console.log("Version int from CSV:", versionInt);
console.log("Decoded version:", versionStr);
```

**Common Causes**:
1. Browser cache - Hard refresh (Ctrl+Shift+R)
2. CSV data not updating - Check CSV transmission code
3. Version encoding bug - Verify major*10000 + minor*100 + patch
4. Stale web files - Clear LittleFS and reflash

#### Downgrade Backtrace on Reboot
**Symptom**: Backtrace appears when switching from ota_0 to factory

**Example**:
```
Update to version 0.2.3 requested - rebooting to factory
Backtrace: 0x4037afeb:0x3fcadcd0 0x42100051:0x3fcadcf0 ...
Rebooting...
```

**Analysis**:
- Occurs during ESP.restart() after esp_ota_set_boot_partition()
- Caused by AsyncWebServer still having active connections
- Harmless - doesn't prevent reboot or corrupt NVS
- Update completes successfully despite backtrace

**Fix Options**:
1. Leave as-is (cosmetic issue only)
2. Implement graceful shutdown:
```cpp
// Instead of immediate restart:
request->send(200, "text/plain", "Rebooting to factory...");
shouldRestart = true;
restartTime = millis() + 5000;

// In loop():
if (shouldRestart && millis() > restartTime) {
    // Close web server
    server.end();
    events.end();
    delay(1000);
    
    // Now restart
    ESP.restart();
}
```

#### Rate Limiting (HTTP 429)
```bash
# Clear rate limits
ssh root@host.xengineering.net "rm -f /home/ota/htdocs/ota.xengineering.net/tmp/rate_limits.json"
```

#### Signature Verification Failed
```bash
# Check signature file
ssh root@host.xengineering.net "wc -c /home/ota/htdocs/ota.xengineering.net/signatures/firmware_0.2.3.sig"
# Should be 684-685 characters

# Verify base64 format (no line breaks)
ssh root@host.xengineering.net "head -c 50 /home/ota/htdocs/ota.xengineering.net/signatures/firmware_0.2.3.sig"
```

### Network Issues

#### SSL Certificate Problems
```bash
# Test SSL certificate
openssl s_client -showcerts -servername ota.xengineering.net -connect ota.xengineering.net:443

# Verify certificate in ESP32 code matches server
```

#### Connection Refused
```bash
# Test connectivity
ping ota.xengineering.net

# Test specific port
telnet ota.xengineering.net 443

# Check firewall
ssh root@host.xengineering.net "ufw status"
```

### Complete Update Flow Debug

```bash
# 1. Check web interface loads versions:
# Browser console should show: "Loading available versions from server..."

# 2. Check ESP32 receives update request:
# Serial monitor should show: "Direct update to version X.X.X from factory partition"

# 3. Check update parameters received:
# Look for parameter handler execution in serial output

# 4. Check targeted version request:
# Should see: "Requesting specific version from: ...?requestedVersion=X.X.X"

# 5. Check server response:
# Should see: "Server response: {"version":"X.X.X",...}"

# 6. Check download initiation:
# Should see OTA download progress messages

# 7. Monitor for completion:
# Look for "Streaming OTA update successful" messages
```

---

## Future Improvements

### Planned Enhancements
1. **Mandatory Updates**: Deploy different versions to different device groups (or individual customer ID's) for A/B testing and special  
4. **Update Notifications**: Push notifications when new versions available
7. **Graceful Web Server Shutdown**: Eliminate backtrace during partition switches

### Known Limitations

1. **Single OTA Partition**: Can't run A/B updates (would require ota_0 and ota_1)
2. **No Automatic Rollback**: Failed updates require manual GPIO41 recovery
3. **Rate Limiting**: May need adjustment for fleet deployments
4. **Web Interface Caching**: Browser may cache old JavaScript/CSS files
5. **Backtrace on Reboot**: Cosmetic issue during partition switching

---

## Appendix: Complete File Reference

### ESP32 Firmware Files
```
Xregulator.ino          - Main firmware with OTA logic
partitions.csv          - Partition table definition
data/index.html         - Web interface HTML
data/script.js          - Version selection JavaScript
data/styles.css         - Web interface styling
build_and_deploy.sh     - Automated deployment script
```

### Server Files
```
api/firmware/check.php      - Update availability (targeted requests)
api/firmware/download.php   - Firmware download
api/firmware/signature.php  - Signature download
api/firmware/versions.php   - Version list API
config/ota_config.php       - Version and key configuration
config/security.php         - Security validation functions
config/versions.json        - Version metadata
firmware/firmware_*.tar     - Firmware packages
signatures/firmware_*.sig   - RSA signatures
```

### Key Constants and Configuration
```cpp
// ESP32 Firmware
const char* FIRMWARE_VERSION = "0.2.3";
const char* OTA_SERVER_URL = "https://ota.xengineering.net";
const char* OTA_PUBLIC_KEY = "-----BEGIN PUBLIC KEY-----...";

// Server Configuration
CURRENT_FIRMWARE_VERSION = '0.2.4';
RATE_LIMIT_CHECKS_PER_HOUR = 10;
RATE_LIMIT_DOWNLOADS_PER_DAY = 5;
```

---

## Document Version History

- **V6** (2025-10-26): Added server maintenance section documenting complete cleanup process including versions.json
- **V5** (2025-10-25): Added targeted version system documentation, removed automatic update references, clarified ESP32-server communication via URL parameters 
- **V4** (2025-09-27): Added manual version selection system and versions.json
- **V3** (2025-09-15): Added streaming tar parser documentation
- **V2** (2025-08-01): Initial partition scheme documentation
- **V1** (2025-07-01): Basic OTA system documentation

---

**End of Documentation**
