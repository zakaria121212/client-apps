# SmartRoadSense Core Algorithm for Road Roughness Detection

This document describes the core algorithm used by SmartRoadSense mobile applications for detecting and classifying road roughness.

## Overview

The SmartRoadSense system uses smartphone sensors (accelerometers and GPS) to detect and measure road surface quality. The algorithm processes accelerometer data to produce a **PPE (Power Spectral Density Estimate)** value that represents road roughness.

## Architecture

The road roughness detection system consists of several key components:

### 1. Sensor Data Collection (`SensorPack.cs`)

**Location**: `src/Shared/Data/SensorPack.cs`

The `SensorPack` class is responsible for:
- Collecting accelerometer data at **100 Hz** (100 samples per second)
- Collecting GPS data at **1 Hz** (1 sample per second)
- Preprocessing and filtering sensor data
- Feeding processed data to the core algorithmic engine

**Key Parameters**:
- **Accelerometer Sampling Rate**: 100 Hz desired (10ms between samples)
- **GPS Sampling Rate**: 1 Hz (1000ms between samples)
- **Minimum GPS Accuracy**: 15 meters
- **Minimum Speed for Recording**: 20 km/h (~5.5 m/s) in production mode
- **Maximum Speed Threshold**: 300 m/s (~1080 km/h) - values above this are considered GPS errors

### 2. Accelerometer Calibration (`Calibrator.cs`)

**Location**: `src/Shared/Calibration/Calibrator.cs`

Before data collection begins, the accelerometer must be calibrated:

1. **Calibration Process**:
   - Device must be placed on a stable, flat surface
   - Collects 300 samples after dropping the first 100 samples
   - Computes the mean acceleration magnitude
   - Calculates standard deviation to mean ratio
   
2. **Calibration Success Criteria**:
   - Standard deviation to mean ratio must be < 1% (0.01)
   - Calibration factor = `9.80665 / mean_magnitude`
   - Reference gravitational acceleration: 9.80665 m/s²

3. **Purpose**:
   - Normalizes accelerometer readings across different devices
   - Compensates for hardware variations
   - Ensures consistent measurements

### 3. Core Algorithm Engine (`SmartRoadSense.Core`)

**Location**: External DLL (referenced in project files as `../../../Core Algorithm/bin/SmartRoadSense.Core.dll`, though the exact location may vary)

The core algorithm is implemented in a separate library (`SmartRoadSense.Core.dll`) that is not included in this repository's source code. The library includes:

#### Data Input (`DataEntry`)
The engine receives preprocessed data containing:
- **Timestamp**: UTC timestamp of measurement
- **Accelerometer Data**: 
  - `accX`: X-axis acceleration (m/s²)
  - `accY`: Y-axis acceleration (m/s²)
  - `accZ`: Z-axis acceleration (m/s²)
  - All axes are scaled by the calibration factor
- **GPS Data**:
  - `latitude`: Latitude in degrees
  - `longitude`: Longitude in degrees
  - `speed`: Speed in m/s
  - `bearing`: Bearing in degrees
  - `accuracy`: GPS accuracy in meters

#### Output (`Result` via `EngineComputationEventArgs`)
The engine produces:
- **PPE**: Overall roughness index (Power Spectral Density Estimate)
- **PPE Components**:
  - `PpeX`: X-axis component
  - `PpeY`: Y-axis component
  - `PpeZ`: Z-axis component
- **Aggregated GPS Data**:
  - First and last timestamps of the computation window
  - Average latitude and longitude
  - Average bearing and accuracy

### 4. PPE (Roughness) Classification

**Location**: `src/Shared/PpeMapper.cs` and `src/Shared/PpeColorMapper.cs`

PPE values are mapped to quality categories and colors:

#### Six-Bin Classification (`PpeMapper.cs`):
- **Bin 0**: PPE < 0.3 (Excellent road quality)
- **Bin 1**: 0.3 ≤ PPE < 0.7 (Very good)
- **Bin 2**: 0.7 ≤ PPE < 1.3 (Good)
- **Bin 3**: 1.3 ≤ PPE < 1.8 (Fair)
- **Bin 4**: 1.8 ≤ PPE < 2.4 (Poor)
- **Bin 5**: PPE ≥ 2.4 (Very poor)

#### Ten-Level Color Mapping (`PpeColorMapper.cs`):
The system uses a gradient from green (smooth roads) to red (rough roads):
- PPE ≤ 0.0811: Dark green (0, 128, 0)
- 0.0811 < PPE ≤ 0.2058: Yellow-green
- ...
- PPE > 1.7985: Red (255, 0, 0)

## Data Processing Pipeline

### Complete Flow:

1. **Initialization**:
   - User calibrates accelerometer
   - Calibration factor is stored in settings
   - GPS acquires fix

2. **Data Collection** (while recording):
   - Accelerometer samples collected at 100 Hz
   - Each sample scaled by calibration factor
   - GPS updates at 1 Hz
   - Speed validation (minimum 20 km/h in production)

3. **Engine Processing**:
   - `Engine.Register(DataEntry)` called for each valid sample
   - Engine accumulates data in internal buffers
   - Computation performed over time windows
   - `ComputationCompleted` event raised with PPE result

4. **Data Recording** (`Recorder.cs`):
   - PPE values collected into `DataPiece` objects
   - Each `DataPiece` contains:
     - Track ID (unique session identifier)
     - Start/end timestamps
     - PPE value and components (X, Y, Z)
     - GPS coordinates
     - Vehicle type and anchorage information
     
5. **Data Serialization** (`DataCollector.cs`):
   - Data batched in groups of 1000 pieces
   - Serialized to JSON format
   - Written to disk for later upload

6. **Upload and Aggregation**:
   - JSON files uploaded to SmartRoadSense server
   - Server performs map matching to OpenStreetMap roads
   - Data aggregated into 20-meter road segments
   - Published as open data

## Key Features

### Quality Assurance

1. **GPS Validation**:
   - Accuracy must be ≤ 15 meters
   - Speed must be ≥ 20 km/h (production mode)
   - GPS updates older than 1.5 seconds are discarded
   - Jumps exceeding 300 m/s trigger reset

2. **Motion Detection**:
   - Monitors if device is stationary
   - Pauses recording if no movement detected for 5 minutes (production)
   - Resets computation after long pauses (>1 hour)

3. **Error Handling**:
   - Engine computation errors trigger automatic reset
   - Invalid data gracefully handled
   - Memory pressure management to prevent excessive allocations

### Offline Mode

The system supports offline operation:
- GPS data replaced with dummy values (0.0, 0.0)
- PPE computation continues without location tracking
- Useful for testing and algorithm validation

## Algorithm Design Goals

Based on the code structure and implementation details:

1. **Real-time Processing**: Data is processed on-device with minimal latency
2. **Device Independence**: Calibration ensures consistency across devices
3. **Privacy**: No personally identifiable information collected
4. **Efficiency**: Batched processing and serialization minimize overhead
5. **Robustness**: Extensive validation and error handling

## External Resources

- **Core Algorithm Repository**: https://github.com/SmartRoadSense/core
  - Note: Repository exists but contains no source code (empty)
  - Core algorithm distributed as compiled DLL only
  
- **Website Documentation**: https://github.com/SmartRoadSense/website
  - General overview of the system
  - User-facing documentation about the process

## Technical Notes

### PPE Metric
The exact computation of the PPE (Power Spectral Density Estimate) metric is proprietary and implemented in the `SmartRoadSense.Core.dll` library. Based on the name and usage:

- PPE likely represents the **power spectral density** of accelerometer vibrations
- Higher PPE values indicate more vibration/roughness
- Computed separately for each axis (X, Y, Z) and combined into overall PPE
- The metric is dimensionless and normalized

### Data Format
Data pieces are stored as JSON with the following structure:
```json
{
  "TrackId": "guid",
  "StartTimestamp": "ISO8601",
  "EndTimestamp": "ISO8601",
  "Ppe": 1.23,
  "PpeX": 0.45,
  "PpeY": 0.56,
  "PpeZ": 0.22,
  "Latitude": 43.7228,
  "Longitude": 12.6369,
  "Speed": 13.89,
  "Bearing": 45.0,
  "Accuracy": 10,
  "Vehicle": "Car",
  "Anchorage": "Windshield",
  "NumberOfPeople": 1
}
```

## Implementation Files

Key source files for understanding the algorithm:

1. **`src/Shared/Data/SensorPack.cs`**: Sensor management and data preprocessing
2. **`src/Shared/Calibration/Calibrator.cs`**: Accelerometer calibration logic
3. **`src/Shared/Data/Recorder.cs`**: Recording session management
4. **`src/Shared/Data/DataCollector.cs`**: Data batching and serialization
5. **`src/Shared/Data/DataPiece.cs`**: Data structure for road roughness measurements
6. **`src/Shared/PpeMapper.cs`**: PPE value classification
7. **`src/Shared/PpeColorMapper.cs`**: Visual representation of PPE values

## Platform-Specific Implementations

The system has platform-specific sensor implementations:
- **Android**: `src/Shared/Data/SensorPackAndroid.cs`
- **iOS**: `src/Shared/Data/SensorPackIOS.cs`
- **Windows**: `src/Shared/Data/SensorPackWinRT.cs`
- **Fake** (testing): `src/Shared/Data/SensorPackFake.cs`

All platforms implement the same abstract interface defined in `SensorPack.cs`.
