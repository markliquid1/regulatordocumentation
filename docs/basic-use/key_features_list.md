# Features

## Field Control & Charging Management

### Control Modes
- **Standard Mode**: Traditional voltage/current regulation with safety overrides
- **Learning Mode**: Adaptive thermal management that learns safe operating limits
- **Manual Override**: Direct duty cycle control for testing and troubleshooting
- **Force Float Mode**: Precision zero-current battery maintenance charging
- **Weather Mode**: Solar-aware charging that disables alternator when sufficient solar available
- **Limp Home Mode**: Emergency operation with reduced functionality and conservative limits

### Charging Control
- **Multi-Stage Charging**: Automatic bulk/float transitions with configurable voltage targets
- **20-Point RPM Curves**: Engine speed-dependent current limiting for optimal power delivery
- **Dual Current Targets**: Separate Hi/Low mode settings with instant switching
- **BMS Integration**: Digital input for Battery Management System override protection
- **Temperature Compensation**: Real-time thermal feedback with graduated field reduction

### Learning Algorithm 
- **Thermal History Tracking**: Remembers overheat events and applies progressive penalties
- **Neighbor Learning**: Adjusts adjacent RPM points when overheating occurs
- **Automatic Recovery**: Gradual current increases during extended safe operation
- **Persistent Memory**: Learning data survives power cycles and firmware updates

## Sensor Systems & Monitoring

### Measurements
- **Dual Battery Voltage**: INA228 (±0.1% accuracy) + ADS1115 with cross-validation
- **High-Resolution Current**: 16-bit measurements with microamp sensitivity
- **Temperature Monitoring**: Digital OneWire DS18B20 sensors (±0.5°C) + analog thermistor
- **Engine RPM**: Configurable magnetic pickup or alternator stator tap sensing
- **Barometric Pressure**: BMP390 sensor for altitude and weather monitoring

### Data Quality & Reliability
- **Sensor Freshness Tracking**: Real-time monitoring of data age with staleness indicators
- **Automatic Fallback**: Seamless switching between sensor sources on failure
- **Range Validation**: Physical limit checking prevents invalid readings
- **Cross-Validation**: Disagreement detection between redundant sensors triggers safety shutdown

## Battery Management System

### State of Charge (SOC) Calculation
- **Coulomb Counting**: Amp-hour integration with floating-point precision
- **Peukert Correction**: Capacity adjustment for high discharge rates
- **Charge Efficiency**: Configurable charging losses (85-99% for different battery types)
- **Full Charge Detection**: Multi-parameter detection using voltage, current, and time criteria
- **Dynamic Calibration**: Automatic shunt gain correction based on full charge cycles

### SOC Tracking
- **Comprehensive Energy Logging**: Charged/discharged energy totals with Wh precision
- **Fuel Consumption Estimates**: Calculated diesel usage based on alternator energy output
- **Runtime Statistics**: Engine hours, alternator on-time, cycle counting
- **Historical Analysis**: Persistent storage of operational data across power cycles

## Communication & Integration

### Marine Network Integration
- **NMEA2000 (CAN)**: Full bi-directional connectivity with standard marine electronics
- **NMEA0183**: Serial data parsing for legacy GPS and navigation equipment
- **Victron VE.Direct**: Native integration with Victron battery monitors and charge controllers
- **GPS Integration**: Automatic position data from NMEA sources for weather services

### Wireless Connectivity
- **Dual WiFi Modes**: Access Point (hotspot) or Client (connect to ship's network)
- **GPIO Mode Selection**: Hardware pins select operational mode during startup
- **Intelligent Reconnection**: Exponential backoff algorithm with signal strength awareness
- **Captive Portal**: Automatic WiFi configuration interface for easy setup

## Web Interface & Real-Time Data

### Live Monitoring Dashboard
- **Real-Time Streaming**: 170+ parameters updated every 50ms
- **Interactive Plotting**: Four simultaneous time-series plots with configurable scales
- **Performance Monitoring**: CPU load, memory usage, loop timing, and system health
- **Alarm Status**: Visual indicators for all alarm conditions with historical logging

### Configuration Management
- **80+ Configurable Parameters**: Complete system customization via web interface
- **Immediate Effect**: Settings changes apply instantly without restart
- **Password Protection**: SHA-256 secured access control for all configuration changes
- **Factory Reset Options**: Individual parameter or complete system restoration

## Safety & Protection Systems

### Multi-Layer Protection
- **Hardware Overvoltage**: INA228 independent protection with <1ms response
- **Emergency Field Collapse**: Software voltage spike detection with immediate shutdown
- **Temperature Limits**: Progressive field reduction with emergency cutoff
- **Current Limiting**: Battery and alternator current protection with user limits
- **Watchdog Protection**: 15-second hang detection with automatic restart

### Fault Detection & Recovery
- **Sensor Validation**: Automatic detection of failed or disconnected sensors
- **Graceful Degradation**: Continued operation with reduced functionality on failures
- **Cross-Sensor Validation**: 0.1V disagreement between voltage sensors triggers shutdown
- **Automatic Recovery**: Self-healing from temporary faults and communication failures

## Advanced Features

### Alternator Lifetime Modeling
- **Physics-Based Calculations**: Arrhenius equation modeling for insulation degradation
- **Component Life Tracking**: Separate modeling for windings, bearings, and brushes
- **Thermal Stress Integration**: Continuous tracking of temperature exposure effects
- **Predictive Warnings**: Remaining component life estimates with color-coded indicators

### Weather Integration
- **Automatic Weather Data**: GPS-based solar irradiance forecasting
- **Multi-Day Analysis**: 3-day solar output predictions for charging decisions
- **Smart Charging**: Disables alternator when sufficient solar energy predicted
- **Open-Meteo API**: Free worldwide weather service with no API key required

### Alarm & Notification System
- **Configurable Thresholds**: User-defined limits for temperature, voltage, and current
- **Latching Options**: Choice between momentary and persistent alarm states
- **Buzzer/Relay Output**: 5V output drives up to 500mA for external devices
- **Console Messaging**: Real-time system messages with throttled warnings

## Over-The-Air (OTA) Updates

### Secure Update System
- **Cryptographic Verification**: RSA-4096 signature validation for firmware authenticity
- **Streaming Installation**: Memory-efficient TAR package extraction
- **Atomic Updates**: Complete verification before installation
- **Rollback Protection**: Automatic recovery if update fails

### Recovery Systems
- **Factory Partition**: Brick-proof recovery accessible via GPIO pin
- **Multiple Recovery Modes**: GPIO pins provide different recovery options
- **Automatic Restart**: Scheduled 2-hour restarts for memory management

## Hardware Specifications

### System Compatibility
- **Wide Input Range**: 4.2V - 65V (supports 12V, 24V, 48V systems)
- **Field Control**: 0-15A field current, PWM frequencies 50Hz - 25kHz
- **Current Measurement**: ±500A capability with external shunt
- **Temperature Range**: -40°C to +85°C operating temperature

### Power Consumption
- **Ultra-Low Quiescent**: 88µA @ 12V during normal operation
- **Sleep Mode**: 26µA @ 12V with ESP32 in deep sleep
- **USB Programming**: Alternative power source for development

### Connectivity
- **USB-C Programming**: Native ESP32-S3 USB with no converter required
- **Standard Connectors**: Widely available RJ45 and other standard connectors
- **Compact Design**: Suitable for various marine installation locations

## Development & Customization

### Open Source Platform
- **Full Source Code**: Complete firmware and hardware designs available
- **Arduino IDE Compatible**: Standard development environment
- **Extensive Documentation**: Comprehensive technical documentation
- **Community Support**: Open platform for modifications and improvements

### Configuration Options
- **File-Based Settings**: Human-readable configuration files
- **Multiple Reset Types**: Password, WiFi, and complete factory resets
- **Debug Capabilities**: Extensive logging and diagnostic capabilities
- **Performance Monitoring**: Real-time system health and timing analysis

## Future Platform Integration

### Expandability
- **Cloud Database Preparation**: Framework for Supabase integration
- **Community Features**: Potential for fleet data sharing and optimization
- **Platform Ecosystem**: Integration with future X Engineering products
- **Machine Learning**: Foundation for advanced optimization algorithms