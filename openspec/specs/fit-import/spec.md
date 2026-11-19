# fit-import Specification

## Purpose
Reads Garmin FIT files of sports watches, bike computers and apps according to the public FIT protocol. Activities become tracks, course files become tracks and waypoints, without gpsbabel.

## Requirements

### Requirement: File structure
The import SHALL read FIT files according to the FIT protocol:
- **File header:** 12 or 14 bytes with the signature `.FIT` and the length of the data section.
- **Messages:** definition and data messages in little and big endian, also data messages with a compressed timestamp.
- **Chaining:** if further FIT files follow after the final checksum, the import SHALL read them too.

#### Scenario: Big-endian definition
- **WHEN** a file contains a `record` message with a big-endian definition
- **THEN** position and time are read just as with little endian

#### Scenario: Compressed timestamp
- **WHEN** a `record` message with the timestamp T is followed by a message with a compressed timestamp and the offset 3
- **THEN** the second point has the time T + 3 s

#### Scenario: Chained files
- **WHEN** two complete FIT files follow each other directly in one file
- **THEN** the result contains the points of both parts

### Requirement: Unknown content
Unknown messages and fields, developer fields and fields that are larger than in the profile SHALL be skipped by their size without a warning. Fields with the invalid value of their base type SHALL count as missing.

#### Scenario: Developer fields
- **WHEN** a definition additionally contains developer fields
- **THEN** they are skipped, and the other fields of the message are read correctly

#### Scenario: Invalid altitude
- **WHEN** the field `altitude` of a `record` message has the value 0xFFFF
- **THEN** the point has no elevation

### Requirement: Track points
Every `record` message with a valid latitude and longitude SHALL result in a track point:
- **Position:** converted from semicircles to degrees (factor 180 / 2³¹).
- **Time:** UTC from the timestamp (seconds since 31 December 1989, 00:00 UTC). Values below 0x10000000 are relative device time; the point SHALL then have no time, and a warning counts it.
- **Elevation:** `enhanced_altitude`, else `altitude`.
- **Speed:** `enhanced_speed`, else `speed`, in m/s.

#### Scenario: Convert a point
- **WHEN** a `record` message contains the values `position_lat` 536870912, `position_long` −1073741824, `altitude` 3000 and `timestamp` 1100000000
- **THEN** a point at 45° north and 90° west results, at 100 m elevation and with the time 2024-11-08 11:33:20 UTC

#### Scenario: Enhanced altitude
- **WHEN** a `record` message contains `altitude` and `enhanced_altitude`
- **THEN** `enhanced_altitude` is used

#### Scenario: Relative device time
- **WHEN** a timestamp is smaller than 0x10000000
- **THEN** the point has no time, and a warning counts the points without a date

### Requirement: Messages without a valid position
`record` messages without a position SHALL be skipped without a warning, for example when only sensor data is recorded. A position outside the valid range MUST be discarded and counted as a warning.

#### Scenario: Heart rate only
- **WHEN** a `record` message contains no position but a heart rate
- **THEN** neither a point nor a warning results

#### Scenario: Latitude out of range
- **WHEN** `position_lat` corresponds to more than 90°
- **THEN** no point results, and a warning counts the message

### Requirement: Tracks and segments
Every FIT file SHALL result in one track. If the file contains a `course` message with a name, the track SHALL carry this name, otherwise the file name without extension. A timer stop (`event` `timer` with the type `stop` or `stop_all`) SHALL end the current segment; points after the next start SHALL go into a new segment. Segments without points SHALL be dropped.

#### Scenario: Pause in an activity
- **WHEN** an activity contains 10 points, then a timer stop, a timer start and 5 more points
- **THEN** the track has two segments with 10 and 5 points

#### Scenario: Course file
- **WHEN** a course file contains the `course` message with the name "Alster loop"
- **THEN** the track is called "Alster loop"

#### Scenario: Two files
- **WHEN** two FIT files are read together
- **THEN** the recording contains two tracks in the order in which they were passed

### Requirement: Course points
Every `course_point` message with a valid position SHALL result in a waypoint, with a name and, if present, a time.

#### Scenario: Course point
- **WHEN** a course file contains a `course_point` with the name "Turn"
- **THEN** the recording contains a waypoint "Turn" at this position

### Requirement: Incomplete files
If a file cannot be decoded to its end, the recording SHALL contain all points up to the damaged place, and a warning MUST report that. This applies when the file is shorter than stated in the header, a message reaches beyond the end of the file or a data message refers to an undefined local message type.

#### Scenario: Truncated activity
- **WHEN** a file with 100 points ends after the 60th point in the middle of a message
- **THEN** the track contains 60 points, and a warning reports the incomplete file

#### Scenario: Undefined message type
- **WHEN** after 10 points a data message follows without a previous definition of its local type
- **THEN** the track contains 10 points, and a warning reports the damaged record

### Requirement: Checksums and header
If the checksum of the header or of the file is wrong, the content SHALL be read anyway, and a warning MUST report that. If the header itself is invalid, reading MUST end with an error.

#### Scenario: Wrong checksum
- **WHEN** the checksum at the end of the file is wrong
- **THEN** all points are read, and a warning reports the wrong checksum

#### Scenario: Invalid header
- **WHEN** the header states a length of 20 bytes
- **THEN** reading ends with an error that names the file

### Requirement: Detection
A file SHALL be recognized as FIT if its first byte is 12 or 14 and the bytes 8 to 11 are `.FIT`. The file extension `.fit` SHALL only serve as a hint.

#### Scenario: Extension missing
- **WHEN** a FIT file without an extension is passed
- **THEN** the detection suggests FIT

#### Scenario: Other binary file
- **WHEN** a binary file with the extension `.fit` does not contain `.FIT` at the bytes 8 to 11
- **THEN** it is not recognized as FIT
