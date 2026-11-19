# csv-logger-import Specification

## Purpose
Reads the CSV files of the Columbus V-900/Visiontac VGPS-900 and of the iBlue software, clean-room in GPSLoggerKit. They become tracks and, from markers, waypoints.

## Requirements

### Requirement: Lines
The import SHALL read CSV files line by line and accept lines ending with LF, CRLF or CR. It SHALL skip empty lines and lines that consist only of padding bytes without a warning.

#### Scenario: Empty line at the end
- **WHEN** a file ends with an empty line or a line of NUL bytes
- **THEN** neither a point nor a warning results from it

### Requirement: Fields
The import SHALL split the fields of a line at the comma and strip spaces, tabs and NUL bytes from the start and end of every field. An empty field SHALL count as a missing value.

#### Scenario: Fields with padding bytes
- **WHEN** a field of a V-900 file contains `100` followed by two NUL bytes
- **THEN** it is read as `100`

### Requirement: Header line
A file SHALL start with a header line. The columns SHALL be assigned by the names of this header line, case-insensitively. An empty field after a trailing comma SHALL NOT form a column of its own. If a required column of the format is missing, reading MUST end with an error that names the file and line 1.

#### Scenario: Required column missing
- **WHEN** an iBlue CSV file lacks the column `E/W`
- **THEN** reading ends with an error for line 1

### Requirement: Discarded lines
A data line with an invalid or missing date, an invalid time or an unreadable position MUST be discarded and counted as a warning, with the line number of the first case. A position outside the valid range MUST be discarded and counted as an invalid coordinate. Unreadable elevations, speeds, courses and DOP values SHALL count as missing without a warning.

#### Scenario: Unusable line
- **WHEN** a data line has fewer fields than needed for date, time and position
- **THEN** no point results from it, and a warning counts the discarded record with its line number

#### Scenario: Latitude out of range
- **WHEN** a line has the latitude 95°
- **THEN** no point results from it, and a warning counts the invalid coordinate

### Requirement: V-900 columns
The format `v900` SHALL read files whose header line starts with the columns `INDEX`, `TAG`, `DATE`, `TIME`, `LATITUDE N/S` and `LONGITUDE E/W`. It SHALL evaluate the columns of the simple mode (`HEIGHT`, `SPEED`, `HEADING`, `VOX`) and of the professional mode (additionally `FIX MODE`, `VALID`, `PDOP`, `HDOP`, `VDOP`), where present.

#### Scenario: Simple mode
- **WHEN** a file contains the header line `INDEX,TAG,DATE,TIME,LATITUDE N/S,LONGITUDE E/W,HEIGHT,SPEED,HEADING,VOX` and the line `1,T,261102,120000,50.000000N,008.000000E,100,36,45,`
- **THEN** a point results at 50.0° north and 8.0° east with the time 2026-11-02T12:00:00Z, the elevation 100 m, the speed 10 m/s and the course 45°

### Requirement: V-900 time and position
The format `v900` SHALL read `DATE` as `YYMMDD` with the years 2000–2099 and `TIME` as `hhmmss`, together as UTC. It SHALL read `LATITUDE N/S` and `LONGITUDE E/W` as decimal degrees with a trailing `N`, `S`, `E` or `W`, where `S` and `W` make the value negative. Another letter MUST discard the line.

#### Scenario: South and west
- **WHEN** a line has the position `50.000200S` and `008.000100W`
- **THEN** latitude and longitude are negative

#### Scenario: Invalid hemisphere
- **WHEN** a line has the latitude `50.000000X`
- **THEN** no point results from it, and a warning counts the discarded record

### Requirement: V-900 measurements
The format `v900` SHALL read `HEIGHT` as elevation in meters, `SPEED` in km/h and `HEADING` as course in degrees. In professional mode `VALID` = `DGPS` SHALL result in the fix type `dgps`, otherwise `FIX MODE` = `2D` or `3D` in the fix type `2d` or `3d`. `PDOP`, `HDOP` and `VDOP` SHALL be taken over if they are greater than 0.

#### Scenario: Professional mode
- **WHEN** a file additionally has the columns `FIX MODE`, `VALID`, `PDOP`, `HDOP` and `VDOP` and a line contains the values `3D`, `SPS`, `2.1`, `1.2` and `1.8`
- **THEN** the point has the fix type `3d`, PDOP 2.1, HDOP 1.2 and VDOP 1.8

#### Scenario: Differential GPS
- **WHEN** a line in professional mode has `VALID` = `DGPS`
- **THEN** the point has the fix type `dgps`

#### Scenario: Negative elevation
- **WHEN** the field `HEIGHT` has the value `-5`
- **THEN** the point has the elevation -5 m

### Requirement: V-900 markers
Every valid line of a V-900 file SHALL result in a track point. If `TAG` has the value `C` or `D` (marker by button), a waypoint SHALL additionally result with position, time and elevation of the point, named `POI` followed by the running number of the marker in the file. For `TAG` = `V` (voice recording) the waypoint SHALL be named like the recording, that is like the value from `VOX` with the extension `.WAV`. If `VOX` is empty, it SHALL be named like a marker by button.

#### Scenario: Marker by button
- **WHEN** the second marker of a file is a line with `TAG` = `C`
- **THEN** a track point and a waypoint named `POI 2` result at the same position

#### Scenario: Voice recording
- **WHEN** a line has `TAG` = `V` and `VOX` = `VOX00003`
- **THEN** a waypoint named `VOX00003.WAV` results next to the track point

### Requirement: iBlue columns
The format `iblue-csv` SHALL read files whose header line starts with `INDEX` and contains the columns `DATE`, `TIME`, `LATITUDE`, `N/S`, `LONGITUDE` and `E/W`, in any order. It SHALL also evaluate `VALID`, `HEIGHT`, `SPEED`, `HEADING`, `PDOP`, `HDOP`, `VDOP` and the satellite column `NSAT`, where present. It SHALL skip other columns without a warning.

#### Scenario: Export of an iBlue 747
- **WHEN** a file contains the header line `INDEX,RCR,DATE,TIME,VALID,LATITUDE,N/S,LONGITUDE,E/W,HEIGHT,SPEED,HEADING` and the line `1,T,2026/11/02,12:00:00,3d,50.000000,N,8.000000,E,100,5.4,45`
- **THEN** a point results at 50.0° north and 8.0° east with the time 2026-11-02T12:00:00Z, the elevation 100 m, the speed 1.5 m/s, the course 45° and the fix type `3d`

#### Scenario: Other column order
- **WHEN** a file has the columns in the order `INDEX,DATE,TIME,LATITUDE,N/S,LONGITUDE,E/W,HEIGHT`
- **THEN** position, time and elevation are read from the named columns

### Requirement: iBlue time and position
The format `iblue-csv` SHALL read `DATE` as `YYYY/MM/DD` or `DD/MM/YYYY` and `TIME` as `hh:mm:ss`, optionally with fractions of seconds, together as UTC. It SHALL read `LATITUDE` and `LONGITUDE` as decimal degrees. The sign SHALL result only from `N/S` or `E/W`; a minus sign in the number SHALL be ignored. Letters other than `N`, `S`, `E` and `W` MUST discard the line.

#### Scenario: Date of an iBlue 757
- **WHEN** the field `DATE` has the value `02/11/2026`
- **THEN** the point has the date 2 November 2026

#### Scenario: Milliseconds
- **WHEN** the field `TIME` has the value `12:00:00.500`
- **THEN** the point has the time 2026-11-02T12:00:00.500Z

#### Scenario: Sign in the number
- **WHEN** a line has the longitude `-8.000456` with `E/W` = `W`
- **THEN** the point has the longitude -8.000456°

### Requirement: iBlue measurements
The format `iblue-csv` SHALL read `HEIGHT` as elevation in meters, `SPEED` in km/h and `HEADING` as course in degrees. `VALID` SHALL result in the fix type with the values `2d`, `3d` and `DGPS`, case-insensitively; other values result in none. `PDOP`, `HDOP` and `VDOP` SHALL only be taken over if they are greater than 0. From `NSAT` the leading number SHALL give the number of used satellites; negative values count as missing.

#### Scenario: Satellites
- **WHEN** the field `NSAT (USED/VIEW)` has the value `7(10)`
- **THEN** the point has 7 satellites

#### Scenario: Unknown values
- **WHEN** a line has `VALID` = `Unknown`, `PDOP` = `0.000000` and `NSAT` = `-1(`
- **THEN** the point has neither a fix type nor PDOP nor a number of satellites

### Requirement: Tracks of the CSV formats
Every file read SHALL result in exactly one track with one segment if it contains at least one valid point. The track SHALL carry the file name without extension as its name. If several files are read, each SHALL result in a track of its own, in the order in which they were passed.

#### Scenario: Two recordings
- **WHEN** the files `26110201.CSV` and `26110202.CSV` are read together
- **THEN** the recording contains two tracks with the names `26110201` and `26110202`

### Requirement: Detection of the CSV formats
The format detection SHALL recognize a V-900 file by its first line starting, after trimming the fields, with the six required columns of the format `v900`. It SHALL recognize an iBlue CSV file by its first line starting with `INDEX` and containing all required columns of the format `iblue-csv`. Other CSV files SHALL trigger neither of the two formats.

#### Scenario: V-900 file with extension .CSV
- **WHEN** the file `26110201.CSV` has a V-900 header line
- **THEN** the detection suggests the format `v900`

#### Scenario: iBlue file
- **WHEN** a `.csv` file has the header line of an iBlue 747 export
- **THEN** the detection suggests the format `iblue-csv`

#### Scenario: Foreign CSV file
- **WHEN** a `.csv` file has the header line `Name,First name,City`
- **THEN** the result is "unknown"
