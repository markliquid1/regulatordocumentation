# Sensor Systems & Data Acquisition

## Overview

The regulator uses multiple sensor systems to monitor electrical parameters, temperature, and engine status. The system implements redundant measurements with automatic fallback and data validation to ensure reliable operation.

## ADS1115 4-Channel Analog Input

### Purpose
The ADS1115 provides four 16-bit analog-to-digital conversion channels for primary system measurements:
- **Channel 0**: Battery voltage (via voltage divider)
- **Channel 1**: Alternator current (Hall effect sensor)
- **Channel 2**: Engine RPM (magnetic pickup)
- **Channel 3**: Thermistor temperature

### Hardware Configuration
```cpp
// ADS1115 initialization
ADS1115_lite adc(ADS1115_DEFAULT_ADDRESS);
adc.setGain(ADS1115_REG_CONFIG_PGA_6_144V);     // ±6.144V range
adc.setSampleRate(ADS1115_REG_CONFIG_DR_64SPS); // 64 samples/second (15.6ms)
```

**Gain Selection**: ±6.144V range provides maximum resolution for 0-5V sensor inputs
**Sample Rate**: 64 SPS balances conversion time (15.6ms) with noise rejection

### State Machine Implementation
The ADS1115 uses a non-blocking state machine to cycle through all channels:

```cpp
enum ADS1115_State {
  ADS_IDLE,
  ADS_WAITING_FOR_CONVERSION
};

void ReadAnalogInputs() {
  switch (adsState) {
    case ADS_IDLE:
      // Set channel and trigger conversion
      adc.setMux(channelConfig[adsCurrentChannel]);
      adc.triggerConversion();
      adsStartTime = millis();
      adsState = ADS_WAITING_FOR_CONVERSION;
      break;
      
    case ADS_WAITING_FOR_CONVERSION:
      if (millis() - adsStartTime >= ADSConversionDelay) {
        Raw = adc.getConversion();
        processChannelData(adsCurrentChannel, Raw);
        adsCurrentChannel = (adsCurrentChannel + 1) % 4;
        adsState = ADS_IDLE;
      }
      break;
  }
}
```

**Benefits**:
- Non-blocking operation prevents main loop delays
- Complete 4-channel cycle every 80ms (20ms per channel)
- Automatic channel rotation ensures fresh data

### Channel Processing

#### Channel 0: Battery Voltage
```cpp
Channel0V = Raw / 32767.0 * 6.144 / 0.0697674419;  // Voltage divider compensation
BatteryV = Channel0V;
if (BatteryV > 5.0 && BatteryV < 70.0) {            // Sanity check
  MARK_FRESH(IDX_BATTERY_V);
}
```
**Voltage Divider**: 1MΩ and 75kΩ resistors provide ~14:1 scaling for 12-48V systems

#### Channel 1: Alternator Current
```cpp
Channel1V = Raw / 32767.0 * 6.144 * 2;           // 2:1 voltage divider
MeasuredAmps = (Channel1V - 2.5) * 100;          // Hall sensor: 2.5V = 0A
if (InvertAltAmps == 1) MeasuredAmps *= -1;      // Polarity correction
MeasuredAmps -= AlternatorCOffset;               // User calibration offset
if (AutoAltCurrentZero == 1) {
  MeasuredAmps -= DynamicAltCurrentZero;         // Auto-zero correction
}
```
**Hall Sensor**: Linear output with 2.5V center point, 100A full scale

#### Channel 2: Engine RPM
```cpp
Channel2V = Raw / 32767.0 * 2 * 6.144 * RPMScalingFactor;
RPM = Channel2V;
if (RPM < 100) RPM = 0;                          // Eliminate noise at idle
```
**RPM Sensor**: Magnetic pickup provides pulse train proportional to engine speed

#### Channel 3: Thermistor Temperature
```cpp
Channel3V = Raw / 32767.0 * 6.144 * 833 * 2;    // Signal conditioning
temperatureThermistor = thermistorTempC(Channel3V);
if (temperatureThermistor > 500) temperatureThermistor = -99;  // Error indication
```

### Thermistor Calculation
```cpp
int thermistorTempC(float V_thermistor) {
  float R_thermistor = R_fixed * (V_thermistor / (5.0 - V_thermistor));
  float T0_K = T0_C + 273.15;
  float tempK = 1.0 / ((1.0 / T0_K) + (1.0 / Beta) * log(R_thermistor / R0));
  return (int)(tempK - 273.15);
}
```
**Steinhart-Hart Equation**: Converts resistance to temperature using thermistor constants

## INA228 High-Precision Battery Monitor

### Purpose
The INA228 provides high-accuracy battery voltage and current measurement with hardware overvoltage protection:
- **Shunt voltage**: ±163.84mV range with 40nV resolution
- **Bus voltage**: 0-85V range with 3.125mV resolution
- **Hardware protection**: Configurable overvoltage threshold

### Configuration
```cpp
INA.setMode(11);                           // Continuous shunt and bus voltage
INA.setAverage(4);                         // 4-sample averaging
INA.setBusVoltageConversionTime(7);        // 4120µs conversion time
INA.setShuntVoltageConversionTime(7);      // 4120µs conversion time

// Hardware overvoltage protection
uint16_t thresholdLSB = (uint16_t)(VoltageHardwareLimit / 0.003125);
INA.setBusOvervoltageTH(thresholdLSB);
INA.setDiagnoseAlertBit(INA228_DIAG_BUS_OVER_LIMIT);
```

**Update Rate**: ~529ms total conversion time with averaging and conversion settings

### Data Processing
```cpp
void ReadINA228() {
  if (INADisconnected == 0) {
    IBV = INA.getBusVoltage();                              // Battery voltage
    ShuntVoltage_mV = INA.getShuntVoltage() / 1000;        // Shunt voltage
    
    if (!isnan(IBV) && IBV > 5.0 && IBV < 70.0) {
      Bcur = ShuntVoltage_mV * 1000.0f / ShuntResistanceMicroOhm;  // Current calculation
      Bcur = Bcur + BatteryCOffset;                         // User calibration
      
      if (InvertBattAmps == 1) Bcur = -Bcur;               // Polarity correction
      
      // Dynamic gain correction (Learning mode)
      if (AutoShuntGainCorrection == 1 && AmpSrc == 1) {
        Bcur = Bcur * DynamicShuntGainFactor;
      }
      
      MARK_FRESH(IDX_IBV);
      MARK_FRESH(IDX_BCUR);
    }
  }
}
```

### Hardware Overvoltage Protection
```cpp
// Check hardware alert status
uint16_t alertStatus = readINA228AlertRegister(INA.getAddress());
if (!inaOvervoltageLatched && (alertStatus & 0x0080)) {
  inaOvervoltageLatched = true;
  inaOvervoltageTime = millis();
  queueConsoleMessage("INA228 hardware overvoltage detected! Field disabled until corrected");
}

// Auto-clear after 10 seconds if condition resolved
if (inaOvervoltageLatched && millis() - inaOvervoltageTime >= 10000) {
  clearINA228AlertLatch(INA.getAddress());
  if (!(readINA228AlertRegister(INA.getAddress()) & 0x0080)) {
    inaOvervoltageLatched = false;
  }
}
```

**Benefits**:
- Hardware-level protection independent of software
- Automatic recovery when overvoltage condition clears
- Prevents alternator runaway scenarios

## OneWire Temperature Sensors

### Purpose
DS18B20 digital temperature sensors provide alternator temperature monitoring with high accuracy and immunity to electrical noise.

### Hardware Configuration
```cpp
#define ONE_WIRE_BUS 13
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void initializeOneWire() {
  sensors.begin();
  sensors.setResolution(12);                    // 12-bit resolution (0.0625°C)
  sensors.getAddress(tempDeviceAddress, 0);     // Get first sensor address
}
```

### FreeRTOS Task Implementation
Temperature reading runs in a separate task to prevent main loop blocking:

```cpp
void TempTask(void *parameter) {
  esp_task_wdt_add(NULL);                       // Add to watchdog monitoring
  
  for (;;) {
    if (millis() - lastTempRead < 10000) {      // 10-second update interval
      vTaskDelay(pdMS_TO_TICKS(1000));
      esp_task_wdt_reset();
      continue;
    }
    
    // Trigger conversion
    sensors.requestTemperaturesByAddress(tempDeviceAddress);
    
    // Wait for conversion (5 seconds with watchdog feeding)
    for (int i = 0; i < 25; i++) {
      vTaskDelay(pdMS_TO_TICKS(200));
      esp_task_wdt_reset();
    }
    
    // Read result
    if (sensors.readScratchPad(tempDeviceAddress, scratchPad)) {
      int16_t raw = (scratchPad[1] << 8) | scratchPad[0];
      float tempF = (raw / 16.0) * 1.8 + 32.0;
      
      if (tempF > -50 && tempF < 300) {          // Sanity check
        AlternatorTemperatureF = tempF;
        if (AlternatorTemperatureF > MaxAlternatorTemperatureF) {
          MaxAlternatorTemperatureF = AlternatorTemperatureF;
        }
        MARK_FRESH(IDX_ALTERNATOR_TEMP);
      }
    }
    
    lastTempRead = millis();
  }
}
```

**Task Benefits**:
- Non-blocking main loop operation
- Independent watchdog monitoring
- Automatic error handling and recovery

## Data Validation and Quality Control

### Sanity Checking
Each sensor reading undergoes validation before use:

```cpp
// Example: Battery voltage validation
if (BatteryV > 5.0 && BatteryV < 70.0 && !isnan(BatteryV)) {
  // Valid reading - mark fresh and use
  MARK_FRESH(IDX_BATTERY_V);
} else {
  // Invalid reading - do not mark fresh, data goes stale
  Serial.println("Invalid battery voltage reading: " + String(BatteryV));
}
```

**Validation Criteria**:
- **Range limits**: Physical limits for each measurement type
- **NaN detection**: Arithmetic error checking
- **Rate limits**: Maximum change rate between readings
- **Cross-validation**: Compare redundant sensors when available

### Data Freshness Tracking
```cpp
enum DataIndex {
  IDX_ALTERNATOR_TEMP = 0,    // OneWire temperature
  IDX_BATTERY_V,              // ADS1115 battery voltage
  IDX_MEASURED_AMPS,          // ADS1115 alternator current
  IDX_IBV,                    // INA228 battery voltage
  IDX_BCUR,                   // INA228 battery current
  IDX_RPM,                    // ADS1115 engine RPM
  IDX_VICTRON_VOLTAGE,        // VE.Direct voltage
  IDX_VICTRON_CURRENT,        // VE.Direct current
  // ... additional indices
  MAX_DATA_INDICES = 17
};

unsigned long dataTimestamps[MAX_DATA_INDICES];
const unsigned long DATA_TIMEOUT = 10000;      // 10 seconds

#define MARK_FRESH(index) dataTimestamps[index] = millis()
#define IS_STALE(index) (millis() - dataTimestamps[index] > DATA_TIMEOUT)
```

**Benefits**:
- Web interface shows stale data indicators
- Prevents control decisions based on old readings
- Enables automatic sensor fallback strategies
- Facilitates troubleshooting sensor failures

### Sensor Fallback Strategy
```cpp
float getBatteryVoltage() {
  switch (BatteryVoltageSource) {
    case 0:  // INA228 (preferred)
      if (!IS_STALE(IDX_IBV) && IBV > 8.0 && IBV < 70.0) {
        return IBV;
      }
      // Fall through to ADS1115
      
    case 1:  // ADS1115 (backup)
      if (!IS_STALE(IDX_BATTERY_V) && BatteryV > 8.0 && BatteryV < 70.0) {
        return BatteryV;
      }
      // Fall through to VE.Direct
      
    case 2:  // VE.Direct (tertiary)
      if (!IS_STALE(IDX_VICTRON_VOLTAGE) && VictronVoltage > 8.0) {
        return VictronVoltage;
      }
      break;
  }
  
  queueConsoleMessage("No valid battery voltage source available");
  return 999;  // Error indication
}
```

**Fallback Hierarchy**:
1. **Primary source**: User-configured preference
2. **Secondary sources**: Automatic fallback to validated alternatives
3. **Error handling**: Graceful degradation with user notification

## Performance Characteristics

### Update Rates
| Sensor | Update Rate | Notes |
|--------|-------------|-------|
| ADS1115 | 80ms (4 channels) | Complete cycle every 80ms |
| INA228 | 529ms | With averaging and high precision |
| OneWire | 10 seconds | Separate task, temperature changes slowly |
| NMEA2K | 100ms-2s | Variable based on message type |
| VE.Direct | 2 seconds | Victron device dependent |

### Accuracy Specifications
| Parameter | Sensor | Resolution | Accuracy |
|-----------|--------|------------|----------|
| Battery Voltage | INA228 | 3.125mV | ±0.1% |
| Battery Current | INA228 | 40nV (shunt) | ±0.5% |
| Alternator Current | Hall Sensor | ~100mA | ±2% |
| Temperature | DS18B20 | 0.0625°C | ±0.5°C |
| Engine RPM | Magnetic Pickup | ~1 RPM | ±1% |

### Resource Usage
- **CPU time**: ~15% of main loop for sensor reading
- **Memory**: 68 bytes for data timestamps, ~200 bytes for sensor buffers
- **I2C bandwidth**: ~2kHz for ADS1115 + INA228 continuous operation
- **Task stack**: 4KB for temperature task

## Error Handling and Recovery

### Sensor Disconnection Detection
```cpp
// ADS1115 connection check
if (!adc.testConnection()) {
  ADS1115Disconnected = 1;
  queueConsoleMessage("ADS1115 connection failed");
}

// INA228 connection check  
if (!INA.begin()) {
  INADisconnected = 1;
  queueConsoleMessage("INA228 connection failed");
}
```

### Automatic Recovery
- **I2C bus reset**: Automatic recovery from bus lockup conditions
- **Sensor reinitialization**: Periodic connection testing and recovery
- **Graceful degradation**: Continue operation with available sensors
- **User notification**: Console messages for failed sensors

### Data Integrity
- **Checksums**: Protocol-level validation for digital communications
- **Range checking**: Physical limit validation for all measurements
- **Trend analysis**: Rate-of-change validation to detect sensor glitches
- **Cross-validation**: Compare redundant measurements when available

This sensor system provides the reliable, accurate data foundation required for safe alternator control and comprehensive system monitoring.