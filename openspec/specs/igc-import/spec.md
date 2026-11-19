# igc-import Specification

## Purpose
Reads IGC flight recordings according to the FAI specification as written by variometers, Flarm and flight apps. They become tracks, without gpsbabel.

## Requirements

### Requirement: Records
The import SHALL read the file line by line and accept lines ending with LF, CRLF or CR. It SHALL evaluate B records (position fixes), the I record (extensions of the B records) and the H record `HFDTE` (date). It SHALL ignore all other records, in particular A, C, E, F, G, K and L records, without a warning.

#### Scenario: Task declaration
- **WHEN** a file contains C records with turn points
- **THEN** neither waypoints nor warnings result from them

#### Scenario: Signature
- **WHEN** a file contains G records at the end
- **THEN** they are skipped, and the signature is not checked

### Requirement: Position fixes
A B record SHALL be read according to this fixed layout: time `HHMMSS` (UTC), latitude `DDMMmmm` with `N` or `S`, longitude `DDDMMmmm` with `E` or `W`, validity `A` or `V`, pressure altitude and GNSS altitude with five digits each in meters, possibly with a leading minus. A B record that is shorter than 35 characters, contains invalid digits or a position outside the valid range MUST be discarded and counted as a warning.

#### Scenario: Valid fix
- **WHEN** the file contains the B record `B1101355206343N00006198WA0058700558`
- **THEN** a point results at 52.105717° north and 0.103300° west with the time 11:01:35 UTC and the elevation 558 m

#### Scenario: South and east
- **WHEN** a B record has `S` and `E` as hemispheres
- **THEN** the latitude is negative and the longitude positive

#### Scenario: Truncated record
- **WHEN** a B record ends after 20 characters
- **THEN** no point results from it, and a warning counts the discarded record

### Requirement: Elevation and fix type
The import SHALL use the GNSS altitude as elevation if the validity is `A` and the GNSS altitude is not 0. Otherwise it SHALL use the pressure altitude if it is not 0, else no elevation. The validity `A` SHALL be taken over as fix type `3d` and `V` as `2d`.

#### Scenario: Device without pressure sensor
- **WHEN** a B record has the validity `A`, pressure altitude `00000` and GNSS altitude `00412`
- **THEN** the point has the elevation 412 m and the fix type `3d`

#### Scenario: Fix without GNSS altitude
- **WHEN** a B record has the validity `V`, pressure altitude `00380` and GNSS altitude `00000`
- **THEN** the point has the elevation 380 m and the fix type `2d`

#### Scenario: No elevation
- **WHEN** pressure altitude and GNSS altitude are both `00000`
- **THEN** the point has no elevation

### Requirement: Extensions
The import SHALL evaluate the I record, which states the number, start and end position and code of the extensions in the B records. If the extension `SIU` is present, its value SHALL be taken over as the number of satellites. It SHALL skip other extensions without a warning. A B record that is shorter than an announced extension SHALL still count as a point without this extension.

#### Scenario: Satellites
- **WHEN** the I record is `I023638FXA3940SIU` and a B record has the value `09` at position 39–40
- **THEN** the point has 9 satellites

#### Scenario: Unknown extension
- **WHEN** the I record announces only `GSP` and `TRT`
- **THEN** the points are read without a number of satellites and without a warning

### Requirement: Date and time
The import SHALL read the date from the H record `HFDTE`, both in the form `HFDTEddmmyy` and `HFDTEDATE:ddmmyy,nn`. Two-digit years SHALL be read as 1980–2079. If the time of a B record is smaller than that of the previous one, the date SHALL be advanced by one day. If the date is missing, the import SHALL take over the points without a time and produce a warning.

#### Scenario: New notation
- **WHEN** the file contains `HFDTEDATE:150726,01`
- **THEN** the points have the date 15 July 2026

#### Scenario: Flight over midnight
- **WHEN** the date is 31.12.2025 and a B record at 23:59:58 is followed by one at 00:00:02
- **THEN** the second point has the time 2026-01-01T00:00:02Z

#### Scenario: Missing date
- **WHEN** a file has no `HFDTE` record
- **THEN** the points have no time, and a warning names the number of points without a date

### Requirement: Tracks
Every file read SHALL result in exactly one track with one segment. The track SHALL carry the file name without extension as its name. If several files are read, each SHALL result in a track of its own.

#### Scenario: Two flights
- **WHEN** the files `flight1.igc` and `flight2.igc` are read together
- **THEN** the recording contains two tracks with the names `flight1` and `flight2`

### Requirement: Detection
The format detection SHALL recognize a file as IGC if its first non-empty line is an A record (`A` followed by three letters or digits) and the examined beginning contains at least one H or B record. The file extension `.igc` SHALL only serve as a hint.

#### Scenario: Extension missing
- **WHEN** an IGC file has the extension `.txt`
- **THEN** the detection suggests IGC

#### Scenario: Not an IGC file
- **WHEN** an NMEA file has the extension `.igc`
- **THEN** the detection suggests NMEA 0183
