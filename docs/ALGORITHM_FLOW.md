# SmartRoadSense Algorithm Flow Diagram

This document provides visual representations of the SmartRoadSense road roughness detection algorithm flow.

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile Application                       │
│                                                                 │
│  ┌────────────────┐         ┌─────────────────┐               │
│  │  Accelerometer │ 100 Hz  │  GPS Sensor     │  1 Hz         │
│  │  (3-axis)      ├────────►│  (Location)     ├───────┐       │
│  └────────────────┘         └─────────────────┘       │       │
│         │                                              │       │
│         │                                              │       │
│         ▼                                              ▼       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              SensorPack (Data Preprocessing)          │   │
│  │  • Calibration scaling                                │   │
│  │  • Quality validation                                 │   │
│  │  • Speed filtering                                    │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   │                                            │
│                   │ DataEntry                                  │
│                   ▼                                            │
│  ┌────────────────────────────────────────────────────────┐   │
│  │    SmartRoadSense.Core.Engine (External DLL)          │   │
│  │    • Signal processing                                │   │
│  │    • Power Spectral Density computation               │   │
│  │    • PPE calculation                                  │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   │                                            │
│                   │ Result (PPE + metadata)                    │
│                   ▼                                            │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Recorder (Session Management)            │   │
│  │  • Tracks recording sessions                          │   │
│  │  • Creates DataPiece objects                          │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   │                                            │
│                   │ DataPiece                                  │
│                   ▼                                            │
│  ┌────────────────────────────────────────────────────────┐   │
│  │         DataCollector (Batch & Serialize)             │   │
│  │  • Batches data (1000 pieces)                         │   │
│  │  • Serializes to JSON                                 │   │
│  │  • Writes to disk                                     │   │
│  └────────────────┬───────────────────────────────────────┘   │
│                   │                                            │
└───────────────────┼────────────────────────────────────────────┘
                    │
                    │ JSON Files
                    ▼
         ┌──────────────────────────┐
         │   SmartRoadSense Server  │
         │  • Map matching          │
         │  • Aggregation (20m)     │
         │  • Open Data publishing  │
         └──────────────────────────┘
```

## Calibration Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Calibration Process                       │
└─────────────────────────────────────────────────────────────┘

    User Action: Place device on flat, stable surface
              │
              ▼
    ┌─────────────────────────┐
    │   Start Accelerometer   │
    └───────────┬─────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │   Drop first 50 samples │  ◄── Device settling time
    └───────────┬─────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │ Collect 300 samples     │
    │ Calculate magnitude:    │
    │ √(x² + y² + z²)         │
    └───────────┬─────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │ Compute Statistics:     │
    │ • Mean                  │
    │ • Standard Deviation    │
    └───────────┬─────────────┘
                │
                ▼
    ┌─────────────────────────────────────┐
    │ Check: StdDev/Mean < 1% ?           │
    └──┬───────────────────────────────┬──┘
       │ NO                            │ YES
       │                               │
       ▼                               ▼
    ┌────────────────┐      ┌────────────────────────┐
    │ FAIL           │      │ SUCCESS                │
    │ Device moving  │      │ Scale = 9.80665 / Mean │
    └────────────────┘      │ Store calibration      │
                            └────────────────────────┘
```

## Real-Time Data Processing Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    Data Collection Loop                       │
└──────────────────────────────────────────────────────────────┘

Every 10ms (100 Hz):                    Every 1000ms (1 Hz):
    ┌────────────────┐                      ┌────────────────┐
    │ Accelerometer  │                      │   GPS Update   │
    │ Sample (X,Y,Z) │                      │ (Lat,Lng,etc.) │
    └───────┬────────┘                      └───────┬────────┘
            │                                       │
            ▼                                       ▼
    ┌────────────────┐                      ┌────────────────┐
    │ Scale by       │                      │ Validate:      │
    │ Calibration    │                      │ • Accuracy ≤15m│
    └───────┬────────┘                      │ • Speed ≥20km/h│
            │                               └───────┬────────┘
            │                                       │
            └───────────────┬───────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │  Quality Checks:      │
                │  • GPS fixed?         │
                │  • GPS recent (<1.5s)?│
                │  • Speed valid?       │
                └─────────┬─────────────┘
                          │ PASS
                          ▼
                ┌───────────────────────┐
                │ Create DataEntry:     │
                │ • Timestamp           │
                │ • Accel (X,Y,Z)       │
                │ • GPS data            │
                └─────────┬─────────────┘
                          │
                          ▼
                ┌───────────────────────┐
                │ Engine.Register()     │
                │ (Core Algorithm)      │
                └─────────┬─────────────┘
                          │
                          │ Periodically...
                          ▼
                ┌───────────────────────┐
                │ ComputationCompleted  │
                │ Event                 │
                │ → Result with PPE     │
                └─────────┬─────────────┘
                          │
                          ▼
                ┌───────────────────────┐
                │ Create DataPiece      │
                │ • PPE values          │
                │ • GPS coordinates     │
                │ • Timestamps          │
                └─────────┬─────────────┘
                          │
                          ▼
                ┌───────────────────────┐
                │ Collect in batch      │
                │ (1000 pieces)         │
                └─────────┬─────────────┘
                          │
                          ▼
                ┌───────────────────────┐
                │ Serialize to JSON     │
                │ Write to disk         │
                └───────────────────────┘
```

## PPE Value Interpretation

```
PPE Value Range          Quality Level        Color
─────────────────────────────────────────────────────────
0.00 - 0.30             Excellent          ████ (Green)
0.30 - 0.70             Very Good          ████ (Light Green)
0.70 - 1.30             Good               ████ (Yellow-Green)
1.30 - 1.80             Fair               ████ (Yellow)
1.80 - 2.40             Poor               ████ (Orange)
2.40+                   Very Poor          ████ (Red)
```

## State Machine: Location Sensor Status

```
                    ┌──────────┐
                    │  Fixing  │ ◄──── Initial State
                    └─────┬────┘
                          │
           GPS Lock       │
         ┌────────────────┘
         │
         ▼
    ┌──────────┐
    │ Working  │ ◄──────────┐
    └────┬─────┘            │
         │                  │
         │ GPS Lost         │ New GPS Fix
         │ (5s timeout)     │
         │                  │
         ▼                  │
    ┌──────────┐            │
    │  Fixing  ├────────────┘
    └────┬─────┘
         │
         │ Stationary > 5 min
         │ OR Speed < 20 km/h
         │
         ▼
    ┌──────────────┐
    │ Error States │
    │ • Stationary │
    │ • Too Slow   │
    └──────────────┘
```

## Data Batch Processing

```
DataPiece Collection:

    Piece 1  ──┐
    Piece 2  ──┤
    Piece 3  ──┤
       ...     ├──► Batch (1000 pieces)
    Piece 999 ─┤
    Piece 1000─┘
                    │
                    ▼
              ┌──────────────┐
              │  Serialize   │
              │  to JSON     │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Write File   │
              │ (async)      │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Queue for    │
              │ Upload       │
              └──────────────┘

If buffer exceeds 10,000 pieces:
    → Drop oldest data
    → Log warning
    → Continue collection
```

## Memory Management

```
Accelerometer Samples @ 100 Hz:
    ├─ Drop if frequency > 100 Hz (prevent buffer overflow)
    ├─ Apply calibration scaling
    └─ Feed to Engine

Engine Internal Processing:
    ├─ Accumulate samples in time windows
    ├─ Compute spectral analysis
    ├─ Generate PPE output
    └─ Raise ComputationCompleted event

DataCollector Buffering:
    ├─ Queue: 0-999 pieces (normal operation)
    ├─ Flush at 1000 pieces
    ├─ Drop at 10,000 pieces (overflow protection)
    └─ Async serialization (non-blocking)
```

## Timeline Example

```
Time (s)    Accelerometer    GPS         Engine          Output
────────────────────────────────────────────────────────────────
0.00        Sample 1         Fix         Register        -
0.01        Sample 2         -           Register        -
0.02        Sample 3         -           Register        -
...         ...              ...         ...             ...
1.00        Sample 100       Update      Register        -
1.01        Sample 101       -           Register        -
...         ...              ...         ...             ...
2.00        Sample 200       Update      Register        -
...         ...              ...         ...             ...
~5.00       Sample 500       Update      Computation     PPE=1.23
                                         Complete
...         ...              ...         ...             ...
~10.00      Sample 1000      Update      Computation     PPE=1.45
                                         Complete
```

*Note*: The exact timing of PPE computation depends on the internal windowing and processing logic of the `SmartRoadSense.Core.Engine`, which is not available in the source code.

## Error Handling Flow

```
                ┌────────────────┐
                │  Data Sample   │
                └───────┬────────┘
                        │
                        ▼
        ┌───────────────────────────┐
        │   Validation Checks       │
        └─┬─────────────────────┬───┘
          │ FAIL                │ PASS
          │                     │
          ▼                     ▼
    ┌─────────────┐      ┌──────────────┐
    │ Discard     │      │ Process      │
    │ Sample      │      │ Sample       │
    └─────────────┘      └──────┬───────┘
                                │
                                ▼
                    ┌───────────────────┐
                    │ Engine.Register() │
                    └────┬──────────┬───┘
                         │ Error    │ Success
                         │          │
                         ▼          ▼
                  ┌───────────┐   ┌─────────┐
                  │ Log Error │   │ Continue│
                  │ Reset     │   └─────────┘
                  │ Engine    │
                  └───────────┘
```

## Key Timing Constants

| Parameter                      | Value          | Purpose                           |
|--------------------------------|----------------|-----------------------------------|
| Accelerometer Rate             | 100 Hz         | High-frequency vibration capture  |
| GPS Rate                       | 1 Hz           | Location and speed tracking       |
| Calibration Window             | 300 samples    | Stable measurement period         |
| GPS Unfixed Timeout            | 5 seconds      | Detect GPS signal loss            |
| GPS Stationary Timeout         | 5 minutes      | Detect lack of movement           |
| Maximum Recording Pause        | 1 hour         | Auto-restart session              |
| Data Batch Size                | 1000 pieces    | Efficient serialization           |
| Data Drop Threshold            | 10,000 pieces  | Memory overflow protection        |
| Minimum GPS Accuracy           | 15 meters      | Quality threshold                 |
| Minimum Recording Speed        | 20 km/h        | Filter out slow movement          |
| Maximum Valid Speed            | 300 m/s        | Detect GPS jumps/errors           |

