# suunto-import Specification

## Purpose
Reads recordings of the Suunto Ambit watches in the SML format as Moveslink and Movescount store them as `.sml` or `.xml`. They become tracks, without gpsbabel and without Python.

## Requirements

### Requirement: Samples
The import SHALL read the elements `Sample` below `sml/DeviceLog/Samples`. Every sample with `Latitude` and `Longitude` SHALL result in a track point, independent of its `SampleType`, for example `gps-base` or `gps-small`. Samples without a position SHALL NOT result in points.

#### Scenario: Periodic sample
- **WHEN** a sample of the type `periodic` contains only altitude, temperature and air pressure
- **THEN** no point results from it

### Requirement: Values of the points
The import SHALL take over the following values:
- **Position:** `Latitude` and `Longitude` in radians, converted to degrees.
- **Time:** `UTC` (XML Schema `dateTime`).
- **Movement:** `GPSSpeed` in m/s and `GPSHeading` in radians, converted to degrees.
- **Quality:** `GpsHDOP` as HDOP, `NumberOfSatellites` as number of satellites.

#### Scenario: Convert a GPS sample
- **WHEN** a sample contains `Latitude` 0.8726646259971648, `Longitude` 0.17453292519943295, `UTC` `2026-05-01T08:00:00Z`, `GPSHeading` 3.141592653589793 and `NumberOfSatellites` 12
- **THEN** a point at 50° north and 10° east results, with this time, the course 180° and 12 satellites

### Requirement: Faulty samples
A position outside the valid range or an unreadable number MUST be discarded and counted as a warning. If a readable `UTC` time is missing, the point SHALL be taken over without a time and counted as a warning.

#### Scenario: Invalid latitude
- **WHEN** a sample has the `Latitude` 2.0 (more than π/2)
- **THEN** it is discarded, and a warning counts it

### Requirement: Elevation
If the last periodic sample with `Altitude` lies at most 60 seconds (by `UTC`) before the point, its value SHALL count as the elevation of the point (barometric altitude). Several points can use the same periodic sample. Otherwise `GPSAltitude` of the sample SHALL be used, else no elevation.

#### Scenario: Barometric altitude
- **WHEN** a GPS sample with `GPSAltitude` 130 is preceded ten seconds earlier by a periodic sample with `Altitude` 118
- **THEN** the point has the elevation 118 m

#### Scenario: GPS altitude only
- **WHEN** no periodic sample lies within 60 seconds before a GPS sample with `GPSAltitude` 130
- **THEN** the point has the elevation 130 m

### Requirement: Track and pauses
Every file SHALL result in one track named after the file name without extension. An event `Pause` with `State` `True` SHALL end the current segment. Points after a `Pause` event with `State` `False` SHALL go into a new segment. Segments without points SHALL be dropped.

#### Scenario: Recording with a pause
- **WHEN** a file contains 20 GPS samples, then a pause (`State` `True`), a resume (`State` `False`) and 10 more GPS samples
- **THEN** the track has two segments with 20 and 10 points

#### Scenario: Start event
- **WHEN** the first sample of the file contains a `Pause` event with `State` `False`
- **THEN** no empty segment results from it

### Requirement: Errors
A file that is not well-formed XML MUST lead to an error with file and line. Header data (`Header`, `Device`) SHALL be ignored.

#### Scenario: Truncated file
- **WHEN** an SML file ends in the middle of a sample
- **THEN** reading ends with an error that names file and line

### Requirement: Detection
A file SHALL be recognized as Suunto SML if its first element is `sml`, with or without a namespace prefix. The file extensions `.sml` and `.xml` SHALL only serve as a hint.

#### Scenario: SML with extension .xml
- **WHEN** an SML file has the extension `.xml`
- **THEN** the detection suggests Suunto SML and not GPX
