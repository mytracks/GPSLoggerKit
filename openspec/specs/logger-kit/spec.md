# logger-kit Specification

## Purpose
GPSLoggerKit is a standalone, MIT-licensed Swift library. It reads file formats and logger protocols natively and builds a common data model from them. It runs on macOS and iOS/iPadOS.

## Requirements

### Requirement: Supported platforms
The package MUST build and run on macOS 12 and later and on iOS/iPadOS 27 and later. All file formats and the GPX output SHALL be available on both platforms. Functions that can only exist on macOS, such as device transports, MUST be separated so that the iOS build succeeds without them.

#### Scenario: Tests on the iOS simulator
- **WHEN** the tests of the package run on an iOS 27 simulator
- **THEN** they build and pass as on macOS. Only checks that need macOS tools, such as the XSD validation, are excluded

#### Scenario: Build for macOS 12
- **WHEN** the package is built with deployment target macOS 12
- **THEN** the build succeeds without availability warnings

### Requirement: Standalone
The package MUST build from a fresh checkout of its repository and as a dependency by repository URL, without other repositories and without third-party packages. It may only use the system libraries of the platform.

#### Scenario: Build standalone
- **WHEN** the repository is freshly cloned and `swift build` and `swift test` run there
- **THEN** build and tests succeed without other repositories being present

#### Scenario: Use by URL
- **WHEN** an empty Swift package declares GPSLoggerKit by repository URL and published version as a dependency and is built for macOS and iOS
- **THEN** the Swift Package Manager resolves only this package, and both builds succeed

### Requirement: License and provenance
The package MUST be licensed under the MIT License and contain a license file. It MUST NOT contain code translated or copied from GPL or CC-BY-SA licensed source code. For every format the documentation of the package MUST name the sources used: specifications, own code or black-box observation.

#### Scenario: Provenance record
- **WHEN** a format is added to the package
- **THEN** the provenance list of the package lists this format with its sources

### Requirement: Data model
The package SHALL represent recordings as tracks of segments with track points and as waypoints. A point SHALL have latitude and longitude (WGS 84). Optionally it SHALL have elevation, time (absolute point in time), speed, course, number of satellites, HDOP, VDOP, PDOP and fix type (none, 2D, 3D, DGPS). Waypoints SHALL additionally be able to have a name and a comment.

#### Scenario: Point with coordinates only
- **WHEN** a format returns only latitude and longitude
- **THEN** all other data of the point are empty, not set to 0

### Requirement: Format catalog
The package SHALL provide a catalog of all supported file formats. Every format SHALL have a fixed identifier, a localized name and the typical file extensions.

#### Scenario: Query the catalog
- **WHEN** an app queries the catalog
- **THEN** it contains at least these formats with their file extensions:
  - NMEA 0183 (identifier `nmea`)
  - GPX (identifier `gpx`)
  - IGC (identifier `igc`)
  - Garmin FIT (identifier `fit`)
  - TCX (identifier `tcx`)
  - Suunto SML (identifier `suunto-sml`)
  - KML and KMZ (identifier `kml`, extensions `kml` and `kmz`)
  - Columbus V-900 (identifier `v900`, extension `csv`)
  - iBlue CSV (identifier `iblue-csv`, extension `csv`)

### Requirement: Format detection
The package SHALL suggest the fitting format for a file based on its content. The file extension SHALL only serve as a hint. If no format recognizes the content, the result SHALL be "unknown".

#### Scenario: NMEA with a wrong extension
- **WHEN** an NMEA file has the extension `.txt`
- **THEN** the detection suggests NMEA 0183

#### Scenario: Unknown content
- **WHEN** a file contains a JPEG image
- **THEN** the result is "unknown"

### Requirement: Read several files
A format SHALL read one or more files into one common recording. The order of the files SHALL match the order in which they are passed.

#### Scenario: Two files
- **WHEN** two files of the same format are read together
- **THEN** the recording contains the points of both files in the order in which they were passed

### Requirement: Errors and warnings
If a file is not readable at all or contains not a single usable element, reading MUST end with an error. The error names the file and, if possible, the position. Single damaged places SHALL NOT stop reading; they SHALL be returned as summarized warnings with a count. Messages MUST be available in German and English.

#### Scenario: Partly damaged file
- **WHEN** a file contains 1000 valid and 3 damaged records
- **THEN** the recording contains 1000 points, and a warning reports 3 skipped records

#### Scenario: Empty file
- **WHEN** a file is empty
- **THEN** reading ends with an error that names the file

### Requirement: Warnings for binary files
For binary formats two cases SHALL each result in a warning of their own that names the file: a wrong checksum of the whole file whose content was read anyway, and a file that ends early whose readable beginning was taken over. The messages MUST be available in German and English.

#### Scenario: Incomplete binary file
- **WHEN** a binary file ends after half of it
- **THEN** the recording contains the points of the first half, and a warning names the file as incomplete

#### Scenario: Wrong checksum
- **WHEN** the checksum of an otherwise readable binary file is wrong
- **THEN** the recording contains all points of the file, and a warning names the file

### Requirement: Cancellation
Reading MUST be cancellable. After a cancellation reading SHALL end after the next data block at the latest with the result "cancelled", without a partial result.

#### Scenario: Cancel a large file
- **WHEN** reading a 50 MB file is cancelled
- **THEN** it ends promptly with "cancelled"

### Requirement: GPS week rollover correction
If the time of a point lies more than 512 weeks before a reference time, the package SHALL shift it repeatedly by 1024 weeks until the condition no longer holds. By default the reference time is the current time. The correction SHALL be active by default and can be switched off.

#### Scenario: Logger with week rollover
- **WHEN** a point has the time 15 March 2007 and the reference time is 3 November 2026
- **THEN** the point has the time 29 October 2026 (plus 1024 weeks) after the correction

#### Scenario: Correction switched off
- **WHEN** the correction is switched off
- **THEN** all times stay unchanged

### Requirement: Waypoints as track
If a recording read contains no track points but waypoints, the package SHALL on request build one track from the waypoints in their order. The waypoints are kept.

#### Scenario: File with waypoints only
- **WHEN** a file contains only 20 waypoints and the conversion is requested
- **THEN** the recording contains one track with 20 points and the 20 waypoints

### Requirement: Extensible format catalog
The package SHALL be able to build a catalog from its own formats and additional formats of other packages. Detection, lookup by identifier and reading SHALL work on such a catalog just as on its own one. The own formats SHALL come first. A duplicate identifier MUST be treated as a programming error when building the catalog. The package MUST NOT need any knowledge of the other packages for this.

#### Scenario: Detect an additional format
- **WHEN** a catalog is built from the own formats and an additional test format, and a file of this test format is checked
- **THEN** the detection suggests the test format, and an NMEA file is still detected as NMEA 0183

#### Scenario: Standard catalog unchanged
- **WHEN** an app queries the catalog as before without additions
- **THEN** it contains exactly the package's own formats

### Requirement: Device interface
The package SHALL provide a platform-neutral interface for logger protocols. It consists of a connection (write, read with a time limit, read up to a terminator, discard input, change the baud rate) and a driver contract (read, erase, erase support, progress). Drivers built on it SHALL be testable without hardware against a simulated device and SHALL build on iOS.

#### Scenario: Driver with a simulated device
- **WHEN** a test runs the GNS 2000 driver with a simulated connection
- **THEN** it runs without Bluetooth and returns the recording of the simulated device

#### Scenario: iOS build with drivers
- **WHEN** the package is built for iOS
- **THEN** the build succeeds including the device interface and the GNS 2000 driver
