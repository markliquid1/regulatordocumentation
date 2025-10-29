# XRegulator Complete Partition Scheme & OTA System Documentation

## Table of Contents
1. [Practical Use Primer](#practical-use-primer)
2. [System Architecture & Design Philosophy](#system-architecture--design-philosophy)
3. [ESP32-S3 Partition Configuration](#esp32-s3-partition-configuration)
4. [Boot Logic & Recovery System](#boot-logic--recovery-system)
5. [Dual Filesystem Management](#dual-filesystem-management)
6. [OTA Update Architecture](#ota-update-architecture)
7. [Streaming Tar Parser Implementation](#streaming-tar-parser-implementation)
8. [Security Implementation](#security-implementation)
9. [Build and Deploy Process](#build-and-deploy-process)
10. [Server Configuration](#server-configuration)
11. [Key File Paths & Commands Reference](#key-file-paths--commands-reference)

---

## Practical Use Primer

### How to Deploy a New OTA Update

**Step 1: Update Firmware Version in Code**
```cpp
// In Xregulator.ino, change this line:
const char *FIRMWARE_VERSION = "2.5.0";  // Increment from current version
```
- Must follow semantic versioning: x.y.z (numbers and dots only)
- New version must be numerically higher than current deployed version
- Examples: "1.0.0", "2.1.3", "10.5.22" ✅
- Invalid: "1.1.1Retry", "v2.0.0", "2.1.0-beta" ❌
- Save the file after making this change

**Step 2: Run Build Script**
```bash
cd /Users/joeceo/Documents/Arduino/Xregulator
./build_and_deploy.sh 0.0.4  # Use same version as in code
```

The script will:
- Compile firmware with ESP32-S3 and PSRAM support
- Create tar package with firmware + web files
- Sign with RSA-4096 private key
- Upload to server and update configuration
- Test deployment automatically

**Step 3: Verify Deployment**
The script tests automatically, but you can also manually verify:
```bash
curl -s -H "Device-ID: TEST123" -H "Current-Version: 2.4.0" -H "Hardware-Version: ESP32-S3" \
https://ota.xengineering.net/api/firmware/check.php | python3 -m json.tool
```

**Step 4: Device Updates**
Devices will discover the new version when users manually check for updates through the web interface, or if the server forces automatic installations (emergency use only)


Current active version check:
```bash
ssh root@host.xengineering.net
grep CURRENT_FIRMWARE_VERSION /home/ota/htdocs/ota.xengineering.net/config/ota_config.php
```

## System Architecture & Design Philosophy

### Core Design Principles
- **Factory recovery system**: GPIO41-triggered fallback to permanent "golden image" firmware
- **Dual content updates**: Firmware (.bin) + web files (HTML/CSS/JS) in single package
- **Streaming tar extraction**: Handle large packages efficiently with minimal memory usage
- **Cryptographic security**: RSA-4096 signatures with air-gapped private key (never rotated to prevent bricking)
- **User-controlled updates**: Manual approval required for all updates
- **Partition pairing**: factory+factory_fs vs ota_0+prod_fs for atomic updates

### Important Limitations
- **Cannot rollback to "last working update"** - only to factory
- **Single OTA partition** means each update overwrites the previous version
- **Not a dual OTA system** - requires factory partition + single OTA partition
- **No key rotation** - accepting security risk to prevent field device bricking

### Technology Stack
- **ESP32-S3**: 16MB flash with 8MB OPI PSRAM
- **Package Format**: Uncompressed tar archives (ustar format)
- **Security**: RSA-4096 + SHA-256 signatures (permanent key)
- **Server**: PHP on CloudPanel/nginx with Let's Encrypt SSL
- **Build System**: Arduino CLI with OPI PSRAM support

### File Structure Overview
```
/Users/joeceo/Documents/Arduino/Xregulator/         # Local development
├── Xregulator.ino                                  # Main ESP32 firmware
├── 2_importantFunctions.txt                        # Core support functions
├── 3_nonImportantFunctions.txt                     # Standard/utility functions
├── build_and_deploy.sh                             # Automated build/deploy script
├── partitions.csv                                  # ESP32 partition scheme
├── data/                                           # Web files for OTA
│   ├── index.html                                  # Main dashboard
│   ├── script.js                                   # JavaScript logic
│   ├── styles.css                                  # Styling
│   ├── uPlot.iife.min.js                          # Charting library
│   └── uPlot.min.css                              # Chart styling
└── build/                                         # Arduino compilation output

ota.xengineering.net (Server)                      # Production OTA server
├── api/firmware/
│   ├── check.php                                  # Update availability
│   ├── download.php                               # Package download
│   └── signature.php                              # Signature download
├── config/
│   ├── ota_config.php                            # Version & public key
│   └── security.php                              # Rate limiting
├── firmware/                                     # Signed tar packages
├── signatures/                                   # Base64 RSA signatures
└── tmp/                                          # Rate limiting data

/Users/joeceo/private_key.pem                     # Air-gapped signing key (NEVER ROTATE)
```

---

## ESP32-S3 Partition Configuration

### Custom Partition Scheme (16MB Flash)
**File**: `partitions.csv` (must be in sketch folder)

```csv
# Name,     Type, SubType,  Offset,   Size,     Flags
nvs,        data, nvs,      0x9000,   0x5000,
otadata,    data, ota,      0xe000,   0x2000,
factory,    app,  factory,  0x10000,  0x400000,
ota_0,      app,  ota_0,    0x410000, 0x400000,
factory_fs, data, littlefs, 0x810000, 0x200000,
prod_fs,    data, littlefs, 0xa10000, 0x200000,
userdata,   data, littlefs, 0xc10000, 0x3D0000,
coredump,   data, coredump, 0xFF0000, 0x10000,
```

### Arduino IDE/CLI Configuration
```bash
# Arduino CLI build configuration:
FQBN="esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom,PSRAM=opi"

# Arduino IDE settings:
Board: "ESP32S3 Dev Module"
Flash Size: "16MB (128Mb)"
Partition Scheme: "Custom" (uses partitions.csv)
PSRAM: "OPI PSRAM"
```

### Partition Strategy
- **Factory partitions**: Permanent "golden image" for GPIO41 recovery
- **OTA partitions**: Where updates are installed
- **4MB firmware partitions**: Current usage ~1.4MB, provides 2.6MB growth room
- **Separate filesystems**: Factory vs production web files

---

## Boot Logic & Recovery System

### Boot Preference Logic
The system always prefers ota_0 unless GPIO41 is low or ota_0 is invalid:

```cpp
void ensurePreferredBootPartition() {
  // Check GPIO41 for manual factory reset
  pinMode(41, INPUT_PULLUP);
  bool forceFactory = (digitalRead(41) == LOW);
  
  if (forceFactory) {
    esp_ota_set_boot_partition(factory_partition);
    return;
  }
  // Normal preference: ota_0 first, factory fallback
}
```

### GPIO Control System
- **GPIO41**: Factory Reset - Force factory partition when pulled LOW on boot
- **GPIO45**: WiFi Configuration Mode - Enter configuration interface
- **GPIO46**: WiFi Mode Selection - Choose AP vs Client mode

---

## Dual Filesystem Management

### Filesystem Architecture
- **`LittleFS`**: Always mounted to `userdata` partition for settings/configuration
- **`webFS`**: Dynamically mounted to either `prod_fs` or `factory_fs` for web interface

### Web Interface Filesystem with Intelligent Fallback
The system tries `prod_fs` first, falls back to `factory_fs` if corrupted:

```cpp
bool ensureWebFS() {
  // Try prod_fs first (for OTA updates)
  if (webFS.begin(true, "/web", 10, "prod_fs")) {
    if (validateWebFilesystem()) {
      usingFactoryWebFiles = false;
      return true;
    } else {
      // Force factory rollback on corruption
      esp_ota_mark_app_invalid_rollback_and_reboot();
    }
  }
  // Fallback to factory_fs
  return webFS.begin(true, "/web", 10, "factory_fs");
}
```

### Web File Validation System
Files must contain validation markers:
```cpp
bool validateWebFile(const char *filename) {
  // Check for XREG_START and XREG_END markers
  return start.indexOf("XREG_START") >= 0 && end.indexOf("XREG_END") >= 0;
}
```

---

## OTA Update Architecture

### User-Controlled Updates
Updates require manual user approval. **Future implementation notes**:
- Modify 5-minute check to set `updateAvailable = true` flag only
- Add UI notification showing "Update Available" with version info
- Add "Download Update" button for user-triggered installation
- Keep emergency GPIO-triggered force update capability

### Key Firmware Configuration
```cpp
const char *FIRMWARE_VERSION = "1.9.0";  // Must follow semantic versioning (x.y.z)
```

### OTA Server Configuration
```cpp
const char *OTA_SERVER_URL = "https://ota.xengineering.net";
const char *server_root_ca = "-----BEGIN CERTIFICATE-----\n..."; // Let's Encrypt R10
```

### Partition Switching Logic
The system never overwrites the currently running firmware:
```cpp
if (running_partition == ota0_partition) {
  // Switch to factory first, then restart
  esp_ota_set_boot_partition(factory_partition);
  ESP.restart();
}
// Now safe to update ota_0
```

---

## Streaming Tar Parser Implementation

### The Proven Solution
The streaming tar parser efficiently processes large firmware packages without loading them entirely into memory. While ESP32-S3 with 8MB PSRAM could potentially load 1.8MB+ packages entirely, the streaming approach provides these benefits:
- **Lower memory usage**: <2KB vs 1.8MB+ for full package loading
- **Protection against fragmentation**: No large memory allocations
- **Immediate processing**: Files processed as they're received
- **Proven reliability**: Battle-tested solution

### Core Parser Structure
```cpp
struct StreamingExtractor {
  bool inTarHeader;
  uint8_t tarHeader[512];
  size_t currentFileSize;
  bool inPadding;
  size_t paddingRemaining;
  // OTA and hash verification state...
};
```

### File Routing Logic
```cpp
if (filename.equals("firmware.bin")) {
  // Route to OTA partition
  esp_ota_begin(otaPartition, fileSize, &otaHandle);
} else if (filename.indexOf('.') > 0) {
  // Route to prod_fs for web files
  currentWebFile = webFS.open(filePath, "w");
}
```

### Memory Usage
- **Streaming buffer**: 1KB chunks
- **Tar header buffer**: 512 bytes (fixed tar header size)
- **Total RAM usage**: <2KB for entire OTA process

---

## Security Implementation

### RSA-4096 Cryptographic Security
- **Algorithm**: RSA-4096 with SHA-256 hashing
- **Key storage**: Private key air-gapped at `/Users/joeceo/private_key.pem`
- **Signature size**: 512 bytes (684-685 characters base64)
- **Security level**: Military-grade, sufficient for system lifetime
- **Key rotation**: **NEVER PERFORMED** - accepting security risk to prevent device bricking

### RSA Public Key (Embedded in Firmware)
```cpp
const char *OTA_PUBLIC_KEY = "-----BEGIN PUBLIC KEY-----\n...";
```

### Critical Security Decision: No Key Rotation
This system does not rotate cryptographic keys. While this creates a theoretical long-term security risk, key rotation poses an unacceptable risk of bricking devices in the field. Since this is not a life-or-death system, we accept the security risk rather than risk rendering devices unrecoverable. The RSA-4096 key provides sufficient security for the system's operational lifetime.

### Air-Gapped Signing Process
```bash
# On offline machine only:
openssl dgst -sha256 -sign /Users/joeceo/private_key.pem -out firmware_X.X.X.sig.binary firmware_X.X.X.tar
base64 -i firmware_X.X.X.sig.binary | tr -d '\n' > firmware_X.X.X.sig.base64
```

---

## Build and Deploy Process

### Current Automated Build Script
**Location**: `/Users/joeceo/Documents/Arduino/Xregulator/build_and_deploy.sh`

**Key Configuration**:
```bash
FQBN="esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom,PSRAM=opi"
```

### Usage
```bash
chmod +x build_and_deploy.sh
./build_and_deploy.sh 2.5.0
```

### Build Process
1. Clean previous builds and caches
2. Compile with Arduino CLI including OPI PSRAM support
3. Create tar package (firmware.bin + data/ contents)
4. Sign package with RSA-4096
5. Upload to server
6. Update server version configuration
7. Test deployment

### Package Format
- **firmware.bin**: Compiled ESP32 firmware
- **Web files**: HTML/CSS/JS from data/ folder
- **Format**: Uncompressed ustar tar for streaming compatibility

---

## Server Configuration

### PHP-based OTA Server
**Location**: `ota.xengineering.net` (CloudPanel/nginx)

### API Endpoints
- **`/api/firmware/check.php`**: Update availability check
- **`/api/firmware/download.php`**: Package download with security validation
- **`/api/firmware/signature.php`**: RSA signature download

### Configuration Files
```php
// config/ota_config.php
define('CURRENT_FIRMWARE_VERSION', '2.5.0');
define('OTA_PUBLIC_KEY', '-----BEGIN PUBLIC KEY-----...');
```

### Security Features
- **Rate limiting**: 10 checks/hour, 5 downloads/day per device
- **Input validation**: Device ID and version verification
- **SSL encryption**: Let's Encrypt certificates
- **Signature verification**: Server validates before serving

### Server Maintenance Commands
```bash
# SSH into server
ssh root@host.xengineering.net
SEE BUILD AND DEPLOY file for password

# Check OTA server status
cd /home/ota/htdocs/ota.xengineering.net
ls -la firmware/        # Check firmware files
ls -la signatures/      # Check signature files
cat config/ota_config.php | grep CURRENT_FIRMWARE_VERSION

# Clear rate limits for testing
rm -f tmp/rate_limits.json

# Check server logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Check disk space
df -h /home/ota/
du -sh firmware/        # Firmware storage usage

# Clean old firmware files (if needed)
find firmware/ -name "firmware_*.tar" -mtime +30 -delete
find signatures/ -name "firmware_*.sig" -mtime +30 -delete
```

### Discover Server File Structure (for new deployments)
```bash
# SSH into server to find current file paths
ssh root@host.xengineering.net

# Find firmware storage location
find /home/ota -name "firmware" -type d
# Expected: /home/ota/htdocs/ota.xengineering.net/firmware/

# Find signature storage location  
find /home/ota -name "signatures" -type d
# Expected: /home/ota/htdocs/ota.xengineering.net/signatures/

# List current firmware files
ls -la /home/ota/htdocs/ota.xengineering.net/firmware/

# List current signature files
ls -la /home/ota/htdocs/ota.xengineering.net/signatures/

# Check current server version
grep "CURRENT_FIRMWARE_VERSION" /home/ota/htdocs/ota.xengineering.net/config/ota_config.php
```

---

## Key File Paths & Commands Reference

### Local Development Paths
```
/Users/joeceo/Documents/Arduino/Xregulator/     # Project root
├── partitions.csv                              # Custom partition scheme
├── build_and_deploy.sh                         # Build script with PSRAM support
├── data/                                       # Web files for OTA
└── build/                                      # Arduino compilation output

/Users/joeceo/private_key.pem                   # Air-gapped signing key (PERMANENT)
```

### Server Paths
```
/home/ota/htdocs/ota.xengineering.net/         # Web root
├── config/ota_config.php                      # Version configuration
├── api/firmware/                               # API endpoints
├── firmware/                                   # Package storage
└── signatures/                                 # Signature storage
```

### Essential Commands
```bash
# Build with PSRAM support
arduino-cli compile --fqbn "esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=custom,PSRAM=opi" --output-dir ./build .

# Create package
tar --format=ustar -cf "firmware_X.X.X.tar" -C temp_package .

# Sign package (air-gapped)
openssl dgst -sha256 -sign /Users/joeceo/private_key.pem -out firmware_X.X.X.sig.binary firmware_X.X.X.tar

# Test server
curl -s -H "Device-ID: TEST123" -H "Current-Version: X.X.X" https://ota.xengineering.net/api/firmware/check.php
```

### System Status & Verified Metrics

#### Current System Capabilities
- **Complete streaming tar parser**: Handles large packages efficiently
- **RSA-4096 signature verification**: Military-grade security with permanent key
- **Dual content updates**: Firmware + web files in single package
- **Factory recovery system**: GPIO41 fallback to golden image
- **ESP32-S3 with PSRAM**: 16MB flash + 8MB external PSRAM
- **User-controlled updates**: Manual approval required

#### Current Package Metrics
- **Package size**: ~1.8MB (tested successfully)
- **Firmware size**: ~1.4MB (room for 2.6MB growth within 4MB partition)
- **Web files**: ~400KB (HTML/CSS/JS dashboard)
- **Signature**: 684-685 characters base64
- **Memory usage**: <2KB RAM for streaming vs 1.8MB+ for full loading

#### Server Configuration
- **Domain**: ota.xengineering.net
- **SSL**: Let's Encrypt (auto-renewed)
- **Rate limits**: 10 checks/hour, 5 downloads/day per device
- **Platform**: CloudPanel/nginx/PHP 8.4
- **Security**: Input validation, rate limiting, signature verification

This system provides cryptographically secure OTA updates with streaming capabilities optimized for ESP32-S3 hardware. The proven streaming tar parser enables dual-content updates (firmware + web files) while maintaining security through permanent RSA-4096 signatures and user-controlled update approval.