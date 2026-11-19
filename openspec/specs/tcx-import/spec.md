# tcx-import Specification

## Purpose
Reads files in Garmin Training Center XML (TCX, version 2) as exported by sports watches, Garmin Connect and training apps. Activities and courses become tracks and waypoints, without gpsbabel.

## Requirements

### Requirement: Activities
Every `Activity` below `Activities` SHALL result in a track without a name. Every `Track` element of its laps (`Lap`) SHALL result in a segment of its own, in the order of the file. Segments without points SHALL be dropped.

#### Scenario: Two laps
- **WHEN** an activity has two laps with one `Track` of 10 track points each
- **THEN** one track with two segments of 10 points each results

#### Scenario: Two activities
- **WHEN** a file contains two activities
- **THEN** two tracks result in the order of the file

### Requirement: Track points
A `Trackpoint` with `Position` SHALL result in a point:
- **Position:** `LatitudeDegrees` and `LongitudeDegrees`.
- **Time:** `Time` (XML Schema `dateTime`).
- **Elevation:** `AltitudeMeters`.
- **Speed:** `Speed` from the extension `TPX` (ActivityExtension v2) in m/s, if present.

Heart rate, cadence, distance and lap values SHALL be ignored.

#### Scenario: Point with speed
- **WHEN** a `Trackpoint` contains the time `2026-05-01T08:00:00Z`, the position 53.55° / 9.99°, the elevation 12.5 m and the TPX speed 3.2
- **THEN** a point with these values and the speed 3.2 m/s results

### Requirement: Incomplete track points
Track points without `Position` SHALL be skipped without a warning. A position outside the valid range or an unreadable number MUST be discarded and counted as a warning. A point without a readable time SHALL be taken over without a time and counted as a warning.

#### Scenario: Pause without position
- **WHEN** a `Trackpoint` contains only `Time` and `HeartRateBpm`
- **THEN** neither a point nor a warning results from it

#### Scenario: Invalid latitude
- **WHEN** a `Trackpoint` has the latitude 95
- **THEN** it is discarded, and a warning counts it

#### Scenario: Missing time
- **WHEN** a `Trackpoint` has a position but no `Time`
- **THEN** a point without a time results, and a warning counts the points without a date

### Requirement: Courses
Every `Course` below `Courses` SHALL result in a track with the name from `Name`, with one segment per `Track`. Every `CoursePoint` SHALL result in a waypoint. It gets `Name`, `Time` and `AltitudeMeters` if present, and `Notes` as comment.

#### Scenario: Course with course points
- **WHEN** a file contains a course "Elbe" with 50 track points and two course points
- **THEN** the recording contains one track "Elbe" with 50 points and two waypoints

### Requirement: Content not evaluated
The import SHALL skip `MultiSportSession`, `Workouts`, `Folders`, `Author` and unknown extensions without a warning.

#### Scenario: Workout
- **WHEN** a file contains a workout next to an activity
- **THEN** only the track of the activity results, without a warning

### Requirement: Errors
A file that is not well-formed XML MUST lead to an error with file and line.

#### Scenario: Truncated file
- **WHEN** a TCX file ends in the middle of a `Trackpoint`
- **THEN** reading ends with an error that names file and line

### Requirement: Detection
A file SHALL be recognized as TCX if its first element is `TrainingCenterDatabase`, with or without a namespace prefix. The file extension `.tcx` SHALL only serve as a hint.

#### Scenario: TCX with extension .xml
- **WHEN** a TCX file has the extension `.xml`
- **THEN** the detection suggests TCX and not GPX

#### Scenario: GPX stays GPX
- **WHEN** a GPX file has the extension `.tcx`
- **THEN** the detection suggests GPX
