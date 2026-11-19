# gpx-import Specification

## Purpose
Reads GPX files as Garmin devices and other loggers store them in their memory. Tracks and waypoints enter the common data model this way and can be written again, normalized, as GPX 1.1.

## Requirements

### Requirement: GPX versions
The import SHALL read GPX 1.0 (`http://www.topografix.com/GPX/1/0`) and GPX 1.1 (`http://www.topografix.com/GPX/1/1`). It SHALL read files without a namespace or with an unknown one, as long as the root element is called `gpx`.

#### Scenario: GPX 1.0 of an older Garmin device
- **WHEN** a GPX 1.0 file with one track is read
- **THEN** the recording contains this track with all points

#### Scenario: Without namespace
- **WHEN** the root element `<gpx>` has no namespace
- **THEN** the file is read anyway

### Requirement: Tracks and waypoints
Every `trk` SHALL be taken over as a track with its name, every `trkseg` as a segment. Every `wpt` SHALL be taken over as a waypoint with name and comment. The import SHALL NOT take over routes (`rte`).

#### Scenario: Garmin file with tracks, waypoints and route
- **WHEN** a file contains two tracks, five waypoints and one route
- **THEN** the recording contains two tracks and five waypoints, but no route

### Requirement: Point data
For every point the import SHALL take over the following data where present: `lat`, `lon`, `ele`, `time`, `sat`, `hdop`, `vdop`, `pdop` and `fix`. For GPX 1.0 it SHALL additionally take over `speed` and `course`. It SHALL ignore extensions (`extensions`) and unknown elements without a warning.

#### Scenario: Garmin extensions
- **WHEN** track points contain `gpxtpx:TrackPointExtension` with heart rate
- **THEN** the points are taken over without these data and without a warning

### Requirement: Times
Times SHALL be read according to ISO 8601 with a time zone, that is `Z` or `±hh:mm`, with or without fractions of seconds. The result is an absolute point in time. Times without a zone SHALL count as UTC.

#### Scenario: Time with zone offset
- **WHEN** a point has the time `2026-11-02T15:30:05+01:00`
- **THEN** its point in time is 2 November 2026, 14:30:05 UTC

### Requirement: Invalid points
Points without or with an unreadable latitude or longitude, with a latitude outside ±90° or with a longitude outside ±180° MUST be skipped and counted as a warning.

#### Scenario: Point out of range
- **WHEN** a `trkpt` has the latitude 123.4
- **THEN** this point is missing in the recording, and a warning counts it

### Requirement: Files that are not well-formed
If a file is not well-formed XML, the import MUST end with an error that names file and line. It SHALL NOT return the points read until then as a partial result.

#### Scenario: Truncated file
- **WHEN** a GPX file ends in the middle of a `trkpt`
- **THEN** the import ends with an error that names file and line

### Requirement: Detection
The format detection SHALL recognize a file as GPX if its first element is called `gpx`.

#### Scenario: GPX with extension `.xml`
- **WHEN** a GPX file has the extension `.xml`
- **THEN** the detection suggests GPX
