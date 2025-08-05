# Communication Interfaces

## Overview

The regulator integrates with marine communication protocols to access navigation data, battery monitoring information, and other vessel systems. Three primary interfaces provide comprehensive data integration:

- **NMEA 2000**: CAN-based network for navigation and engine data
- **NMEA 0183**: Serial ASCII protocol for legacy navigation equipment
- **Victron VE.Direct**: Serial protocol for Victron energy system integration

## NMEA 2000 Integration

### Purpose
NMEA 2000 provides access to vessel navigation data, engine parameters, and electrical system information via a CAN bus network.

### Hardware Configuration
```cpp
#define ESP32_CAN_RX_PIN GPIO_NUM_16
#define ESP32_CAN_TX_PIN GPIO_NUM_17
#include <NMEA2000_CAN.h>
#include <N2kMessages.h>

void initializeNMEA2000() {
  NMEA2000.SetForwardType(tNMEA2000::fwdt_Text);
  NMEA2000.SetForwardStream(OutputStream);
  NMEA2000.EnableForward(false);
  NMEA2000.SetMsgHandler(HandleNMEA2000Msg);
  NMEA2000.Open();
}
```

**Pin Configuration**: GPIO 16/17 provide CAN transceiver interface
**Message Filtering**: Selective message processing to reduce CPU load

### Message Handler System
```cpp
typedef struct {
  unsigned long PGN;
  void (*Handler)(const tN2kMsg &N2kMsg);
} tNMEA2000Handler;

tNMEA2000Handler NMEA2000Handlers[] = {
  { 126992L, &SystemTime },
  { 127245L, &Rudder },
  { 127250L, &Heading },
  { 127257L, &Attitude },
  { 127506L, &DCStatus },
  { 127513L, &BatteryConfigurationStatus },
  { 128259L, &Speed },
  { 128267L, &WaterDepth },
  { 129026L, &COGSOG },
  { 129029L, &GNSS },
  { 129540L, &GNSSSatsInView },
  { 0, 0 }
};

void HandleNMEA2000Msg(const tN2kMsg &N2kMsg) {
  for (int i = 0; NMEA2000Handlers[i].PGN != 0; i++) {
    if (N2kMsg.PGN == NMEA2000Handlers[i].PGN) {
      NMEA2000Handlers[i].Handler(N2kMsg);
      break;
    }
  }
}
```

**Message Processing**: Table-driven handler dispatch for registered PGNs

### Key Message Handlers

#### GNSS Position Data (PGN 129029)
```cpp
void GNSS(const tN2kMsg& N2kMsg) {
  static unsigned long lastGNSSUpdate = 0;
  if (millis() - lastGNSSUpdate < 2000) return;  // Throttle to 2 seconds
  lastGNSSUpdate = millis();

  unsigned char SID;
  uint16_t DaysSince1970;
  double SecondsSinceMidnight;
  double Latitude, Longitude, Altitude;
  tN2kGNSStype GNSStype;
  tN2kGNSSmethod GNSSmethod;
  unsigned char nSatellites;
  double HDOP, PDOP, GeoidalSeparation;

  if (ParseN2kGNSS(N2kMsg, SID, DaysSince1970, SecondsSinceMidnight,
                   Latitude, Longitude, Altitude, GNSStype, GNSSmethod,
                   nSatellites, HDOP, PDOP, GeoidalSeparation, ...)) {
    
    // Validate GPS data
    if (!isnan(Latitude) && !isnan(Longitude) && 
        Latitude != 0.0 && Longitude != 0.0 && 
        abs(Latitude) <= 90.0 && abs(Longitude) <= 180.0 && 
        nSatellites > 0) {
      
      LatitudeNMEA = Latitude;
      LongitudeNMEA = Longitude;
      SatelliteCountNMEA = nSatellites;
      
      MARK_FRESH(IDX_LATITUDE_NMEA);
      MARK_FRESH(IDX_LONGITUDE_NMEA);
      MARK_FRESH(IDX_SATELLITE_COUNT);
    }
  }
}
```

**Data Validation**: Range checking and satellite count verification prevent invalid position data

#### Heading Data (PGN 127250)
```cpp
void Heading(const tN2kMsg& N2kMsg) {
  static unsigned long lastHeadingUpdate = 0;
  if (millis() - lastHeadingUpdate < 2000) return;
  lastHeadingUpdate = millis();
  
  unsigned char SID;
  tN2kHeadingReference HeadingReference;
  double Heading, Deviation, Variation;

  if (ParseN2kHeading(N2kMsg, SID, Heading, Deviation, Variation, HeadingReference)) {
    HeadingNMEA = Heading * 180.0 / PI;  // Convert radians to degrees
    MARK_FRESH(IDX_HEADING_NMEA);
  }
}
```

#### DC Status (PGN 127506)
```cpp
void DCStatus(const tN2kMsg& N2kMsg) {
  unsigned char SID, DCInstance;
  tN2kDCType DCType;
  unsigned char StateOfCharge, StateOfHealth;
  double TimeRemaining, RippleVoltage, Capacity;

  if (ParseN2kDCStatus(N2kMsg, SID, DCInstance, DCType, StateOfCharge, 
                       StateOfHealth, TimeRemaining, RippleVoltage, Capacity)) {
    // Process battery monitoring data from NMEA 2000 network
    // Can be used as alternative to local INA228 measurements
  }
}
```

### Performance and Throttling
```cpp
// Message rate limiting prevents CPU overload
static unsigned long lastUpdate = 0;
if (millis() - lastUpdate < THROTTLE_INTERVAL) return;
lastUpdate = millis();
```

**Throttling Strategy**: 
- GNSS updates: 2-second minimum interval
- Heading updates: 2-second minimum interval
- Other messages: Variable based on update rate requirements

## NMEA 0183 Integration

### Purpose
NMEA 0183 provides ASCII sentence-based communication with legacy navigation equipment and GPS devices.

### Hardware Configuration
```cpp
#define NMEA0183_RX_PIN 25
#define NMEA0183_TX_PIN -1  // Receive only

void initializeNMEA0183() {
  Serial1.begin(19200, SERIAL_8N1, NMEA0183_RX_PIN, NMEA0183_TX_PIN, 0);
  Serial1.flush();
}
```

**Baud Rate**: Standard 19200 bps for NMEA 0183
**Configuration**: Receive-only implementation (no transmission required)

### Message Parsing
```cpp
void readNMEA0183() {
  while (Serial1.available()) {
    char c = Serial1.read();
    
    if (c == '$') {
      // Start of new sentence
      nmeaBuffer[0] = c;
      nmeaIndex = 1;
    } else if (c == '\n' || c == '\r') {
      // End of sentence
      if (nmeaIndex > 0) {
        nmeaBuffer[nmeaIndex] = '\0';
        parseNMEASentence(nmeaBuffer);
        nmeaIndex = 0;
      }
    } else if (nmeaIndex < NMEA_BUFFER_SIZE - 1) {
      nmeaBuffer[nmeaIndex++] = c;
    }
  }
}

void parseNMEASentence(char* sentence) {
  if (strncmp(sentence, "$GPGGA", 6) == 0) {
    parseGGA(sentence);
  } else if (strncmp(sentence, "$GPRMC", 6) == 0) {
    parseRMC(sentence);
  }
  // Additional sentence types as needed
}
```

**Sentence Types**:
- **GGA**: Global positioning system fix data
- **RMC**: Recommended minimum specific GPS/transit data
- **VTG**: Track made good and ground speed

### Current Implementation Status
```cpp
int NMEA0183Data = 0;  // Set to 1 if NMEA serial data exists
```

**Implementation Note**: NMEA 0183 parsing framework exists but is not fully implemented. The hardware interface and basic parsing structure are in place for future development.

## Victron VE.Direct Integration

### Purpose
VE.Direct provides access to Victron energy system data including battery monitors (BMV-712), solar charge controllers, and inverters.

### Hardware Configuration
```cpp
#define VEDIRECT_RX_PIN 26
#define VEDIRECT_TX_PIN -1  // Receive only

#include "VeDirectFrameHandler.h"
VeDirectFrameHandler myve;

void initializeVEDirect() {
  Serial2.begin(19200, SERIAL_8N1, VEDIRECT_RX_PIN, VEDIRECT_TX_PIN, 1);
  Serial2.flush();
}
```

**Baud Rate**: 19200 bps standard for VE.Direct
**Protocol**: Text-based label/value pairs with checksum validation

### Data Processing
```cpp
void ReadVEData() {
  if (VeData != 1) return;
  
  static unsigned long lastVEDataRead = 0;
  const unsigned long VE_DATA_INTERVAL = 2000;  // 2 seconds
  
  if (millis() - lastVEDataRead <= VE_DATA_INTERVAL) return;
  
  int start1 = micros();
  bool dataReceived = false;
  
  while (Serial2.available()) {
    myve.rxData(Serial2.read());
    
    for (int i = 0; i < myve.veEnd; i++) {
      if (strcmp(myve.veName[i], "V") == 0) {
        float newVoltage = (atof(myve.veValue[i]) / 1000);
        if (newVoltage > 0 && newVoltage < 100) {
          VictronVoltage = newVoltage;
          MARK_FRESH(IDX_VICTRON_VOLTAGE);
          dataReceived = true;
        }
      }
      
      if (strcmp(myve.veName[i], "I") == 0) {
        float newCurrent = (atof(myve.veValue[i]) / 1000);
        if (newCurrent > -1000 && newCurrent < 1000) {
          VictronCurrent = newCurrent;
          MARK_FRESH(IDX_VICTRON_CURRENT);
          dataReceived = true;
        }
      }
    }
    yield();  // Allow other processes
  }
  
  int end1 = micros();
  VeTime = end1 - start1;  // Performance monitoring
  lastVEDataRead = millis();
}
```

### VE.Direct Protocol Details
```
Label: Value
V: 12750        // Battery voltage (mV)
I: -1230        // Battery current (mA)
P: -15          // Instantaneous power (W)
CE: -1500       // Consumed energy (mAh)
SOC: 876        // State of charge (‰)
TTG: 65535      // Time to go (minutes)
Alarm: OFF      // Alarm status
Relay: OFF      // Relay status
AR: 0           // Alarm reason
BMV: 712        // Model identification
FW: 0212        // Firmware version
Checksum: 9
```

**Data Format**: Text labels followed by colon and numeric value
**Validation**: Checksum verification and range checking for each parameter

### Integration with Control System
```cpp
// VE.Direct data can be used as battery voltage/current source
int BatteryCurrentSource = 3;  // 0=INA228, 1=NMEA2K, 2=NMEA0183, 3=Victron

float getBatteryCurrent() {
  switch (BatteryCurrentSource) {
    case 3:  // Victron VE.Direct
      if (abs(VictronCurrent) > 0.1) {
        return VictronCurrent;  // Direct use, no polarity inversion needed
      } else {
        queueConsoleMessage("Victron current not available, using INA228");
        return Bcur;  // Fallback to local sensor
      }
  }
}
```

**Sensor Hierarchy**: VE.Direct can serve as primary or backup data source for battery monitoring

## Communication Interface Management

### Data Source Selection
```cpp
// Battery voltage source priority
int BatteryVoltageSource = 0;  // 0=INA228, 1=ADS1115, 2=VictronVeDirect, 3=NMEA0183, 4=NMEA2K

float getBatteryVoltage() {
  static unsigned long lastWarningTime = 0;
  
  switch (BatteryVoltageSource) {
    case 0:  // INA228 (preferred)
      if (!IS_STALE(IDX_IBV) && IBV > 8.0 && IBV < 70.0) {
        return IBV;
      }
      // Automatic fallback with user notification
      if (millis() - lastWarningTime > 10000) {
        queueConsoleMessage("INA228 unavailable, falling back to ADS1115");
        lastWarningTime = millis();
      }
      return BatteryV;  // ADS1115 fallback
      
    case 2:  // VE.Direct
      if (!IS_STALE(IDX_VICTRON_VOLTAGE) && VictronVoltage > 8.0) {
        return VictronVoltage;
      }
      queueConsoleMessage("Victron voltage invalid, using INA228");
      return IBV;  // Fallback to local sensor
  }
}
```

### Performance Monitoring
```cpp
// Timing measurement for each interface
int VeTime = 0;        // VE.Direct processing time (microseconds)
int NMEA2KData = 0;    // NMEA 2000 data availability flag
int VeData = 0;        // VE.Direct data availability flag
```

**Performance Tracking**: Microsecond-level timing for interface processing overhead

### Error Handling and Recovery
```cpp
// Interface health monitoring
if (VeData == 1 && IS_STALE(IDX_VICTRON_VOLTAGE)) {
  queueConsoleMessage("VE.Direct communication timeout");
}

if (NMEA2KData == 1 && IS_STALE(IDX_LATITUDE_NMEA)) {
  queueConsoleMessage("NMEA 2000 GPS data timeout");
}
```

**Automatic Recovery**:
- Serial interface reset on prolonged timeouts
- CAN bus reinitialization on communication failures
- Graceful fallback to local sensors when remote data unavailable

## Configuration and Control

### Interface Enable/Disable
```cpp
// User-configurable interface enables
int VeData = 0;        // 0=disabled, 1=enabled
int NMEA0183Data = 0;  // 0=disabled, 1=enabled  
int NMEA2KData = 0;    // 0=disabled, 1=enabled
```

**Runtime Control**: Interfaces can be enabled/disabled via web interface without firmware changes

### Verbose Debug Output
```cpp
int NMEA2KVerbose = 0;  // 0=quiet, 1=serial debug output

if (NMEA2KVerbose == 1) {
  Serial.print("NMEA2K PGN: ");
  Serial.println(N2kMsg.PGN);
}
```

**Debugging**: Optional verbose output for troubleshooting communication issues

## Resource Usage and Performance

### CPU Utilization
| Interface | Processing Time | Update Rate | CPU Impact |
|-----------|----------------|-------------|------------|
| NMEA 2000 | ~100µs per message | Variable (1Hz-10Hz) | ~1-2% |
| VE.Direct | ~2000µs per update | 2 seconds | <1% |
| NMEA 0183 | Not implemented | N/A | 0% |

### Memory Usage
- **NMEA 2000**: ~2KB for message buffers and handler tables
- **VE.Direct**: ~1KB for frame parsing buffers
- **Data storage**: 68 bytes for freshness timestamps

### Communication Bandwidth
- **CAN bus**: ~125kbps physical, ~1-5% utilization typical
- **Serial interfaces**: 19200 bps each, ~10-50% utilization
- **Total overhead**: <5% of main loop execution time

This communication interface system provides comprehensive integration with marine electronics while maintaining efficient resource utilization and robust error handling.