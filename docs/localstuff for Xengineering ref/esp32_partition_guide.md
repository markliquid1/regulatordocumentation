# ESP32-S3 Custom 5MB OTA Partition Setup Guide for Arduino IDE 2.x

## Prerequisites
- Arduino IDE 2.x installed
- ESP32-S3 board with 16MB flash

## Step 1: Install ESP32 Board Package
1. Open Arduino IDE 2.x
2. Go to **File → Preferences**
3. In "Additional board manager URLs" add:
   ```
   https://espressif.github.io/arduino-esp32/package_esp32_index.json
   ```
4. Go to **Tools → Board → Boards Manager**
5. Search for "ESP32" and install **"esp32 by Espressif Systems"**
6. Select your ESP32-S3 board: **Tools → Board → ESP32 Arduino → Your ESP32-S3 variant**

## Step 2: Clear Arduino Cache
**Important:** Clear Arduino cache to avoid conflicts:
```bash
rm -rf ~/Library/Caches/arduino
```

## Step 3: Create Custom Partition Table
1. **Open Terminal and navigate to your sketch folder:**
   ```bash
   cd /Users/joeceo/Documents/Arduino/Xregulator
   ```

2. **Create partitions.csv file using Terminal cat command:**
   ```bash
   cat > partitions.csv << 'EOF'
   # Name, Type, SubType, Offset, Size, Flags
   nvs, data, nvs, 0x9000, 0x5000,
   otadata, data, ota, 0xe000, 0x2000,
   app0, app, ota_0, 0x10000, 0x500000,
   app1, app, ota_1, 0x510000, 0x500000,
   spiffs, data, spiffs, 0xa10000, 0x5D0000,
   coredump, data, coredump, 0xFF0000, 0x40000,
   EOF
   ```

3. **Verify the file was created:**
   ```bash
   cat /Users/joeceo/Documents/Arduino/Xregulator/partitions.csv
   ```

## Step 4: Configure Arduino IDE for Custom Partitions
1. **Tools → Programmer → esptool** (select esptool as programmer)
2. **Important:** Any partition scheme selected in Tools → Partition Scheme will be ignored when partitions.csv exists in sketch folder

## Step 5: Clear Cache and Upload
1. **Clear Arduino cache again before upload:**
   ```bash
   rm -rf ~/Library/Caches/arduino
   ```

2. **Upload your sketch using the regular Upload button**

## Step 6: Verify Success
**Upload this verification sketch:**
```cpp
#include "esp_partition.h"

void setup() {
  Serial.begin(115200);
  delay(2000);
  Serial.println("=== Partition Table ===");
  
  esp_partition_iterator_t it = esp_partition_find(ESP_PARTITION_TYPE_APP, ESP_PARTITION_SUBTYPE_ANY, NULL);
  while (it != NULL) {
    const esp_partition_t* part = esp_partition_get(it);
    Serial.printf("APP: %s at 0x%x, size: %d MB\n", 
                  part->label, part->address, part->size/1024/1024);
    it = esp_partition_next(it);
  }
  esp_partition_iterator_release(it);
}

void loop() {}
```

**Expected output:**
```
APP: app0 at 0x10000, size: 5 MB
APP: app1 at 0x510000, size: 5 MB
```

## Key Points
- ✅ **partitions.csv in sketch folder automatically overrides all IDE partition settings**
- ✅ **Clear Arduino cache frequently to avoid stale cached partition tables**
- ✅ **Always use Terminal cat command to create partitions.csv**

## Important Note About Compilation Messages
**Why compilation messages caused confusion:** Arduino IDE's compilation output shows flash erase ranges that may be smaller than your full partition size (e.g., erasing 1.38MB instead of 5MB). This is normal optimization behavior - Arduino only erases the flash area needed for your current application, not the entire partition. Your 5MB partitions are still correctly configured and available for OTA updates.

## Troubleshooting
If you get boot errors:

1. **Clear cache:**
   ```bash
   rm -rf ~/Library/Caches/arduino
   ```

2. **Complete erase (if needed):**
   ```bash
   find ~/Library/Arduino15/packages/esp32/tools -name "esptool.py" -exec python3 {} --chip esp32s3 --port /dev/cu.usbserial-* erase_flash \;
   ```

3. **Then upload normally using Upload button**

## Cache Clearing Schedule
Clear Arduino cache:
- ✅ Before creating partitions.csv
- ✅ Before uploading
- ✅ Whenever you get unexpected partition behavior
- ✅ After any ESP32 platform updates

**Command to remember:**
```bash
rm -rf ~/Library/Caches/arduino
```