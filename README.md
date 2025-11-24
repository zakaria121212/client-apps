# SmartRoadSense Apps

This repository contains the source code for the mobile client applications of the *SmartRoadSense* service.
Mobile applications for the following platforms are included:

* Android
* iOS

## Core Algorithm Documentation

The SmartRoadSense system uses smartphone sensors (accelerometers and GPS) to detect and classify road surface quality. For detailed information about the road roughness detection algorithm:

* **[Core Algorithm Overview](CORE_ALGORITHM.md)** - Comprehensive documentation of the PPE-based road roughness detection algorithm
* **[Algorithm Flow Diagrams](docs/ALGORITHM_FLOW.md)** - Visual representations of data processing pipelines and system architecture

Key highlights:
- **PPE Metric**: Power Spectral Density Estimate used to quantify road roughness (0.2 = excellent, 2.4+ = very poor)
- **Sensor Fusion**: Combines 100Hz accelerometer data with 1Hz GPS data
- **Calibration**: Device-independent measurements through automatic calibration
- **Real-time Processing**: On-device computation with minimal latency

## How to build

The project requires Visual Studio 2017 and Xamarin as mobile middleware.

### Who do I talk to?

* [Alessandro Bogliolo](https://twitter.com/neutralaccess), project coordinator
* [Lorenz Cuno Klopfenstein](https://twitter.com/lorenzck), dev
* Brendan D. Paolini, dev
