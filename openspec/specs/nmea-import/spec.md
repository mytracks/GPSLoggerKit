# nmea-import Specification

## Purpose
Reads NMEA 0183 files of GPS loggers and cameras, for example of the Sony GPS CS-1/CS-3, Canon EOS LOG, AMOD and ATP. They become tracks and waypoints, without gpsbabel.

## Requirements

### Requirement: Supported sentences
The import SHALL evaluate the sentences RMC, GGA, GLL, GSA, VTG, ZDA and WPL. It SHALL accept every talker, in particular GP, GN, GL, GA, GB, BD and GQ. It SHALL ignore other sentences, such as GSV and proprietary `$P…` sentences, without a warning.

#### Scenario: Multi-GNSS logger
- **WHEN** a file contains `$GNRMC` and `$GNGGA` sentences
- **THEN** they are evaluated like `$GPRMC` and `$GPGGA`

#### Scenario: Satellite status
- **WHEN** a file contains `$GPGSV` sentences
- **THEN** they are skipped without a warning

### Requirement: Lines and checksums
The import SHALL accept lines ending with LF, CRLF or CR. It SHALL ignore lines that do not start with `$`; this includes the `@` lines of the Sony loggers. If a sentence has a checksum and it is wrong, the sentence MUST be discarded and counted as a warning. It SHALL accept sentences without a checksum.

#### Scenario: Sony log file
- **WHEN** a file contains lines that start with `@`
- **THEN** these lines are skipped, and the other sentences are evaluated

#### Scenario: Wrong checksum
- **WHEN** an RMC sentence has a wrong checksum
- **THEN** no point results from it, and a warning counts the discarded sentence

#### Scenario: Truncated last line
- **WHEN** the last line of a file ends in the middle of a sentence
- **THEN** it is discarded, and all earlier points are kept

### Requirement: Only valid positions
The import MUST discard positions without a valid fix: RMC and GLL with status `V`, GGA with fix quality 0 and sentences without coordinates. Such sentences SHALL NOT produce a warning.

#### Scenario: Logger without reception
- **WHEN** a file contains 50 RMC sentences with status `V` and then 10 with status `A`
- **THEN** the track contains 10 points

### Requirement: One point per point in time
Sentences with the same UTC time SHALL be combined into one point. The rules are:
- Coordinates, date, speed and course come from RMC.
- Altitude above sea level, number of satellites and HDOP come from GGA.
- Fix type, PDOP, HDOP and VDOP come from GSA.
- Speed and course come from VTG as a fallback.

#### Scenario: RMC and GGA in the same second
- **WHEN** a `$GPRMC` at 12:00:01 is followed by a `$GPGGA` at 12:00:01 with altitude 94.5 m and 8 satellites
- **THEN** a single point results with the coordinates from RMC, altitude 94.5 m and 8 satellites

#### Scenario: GGA only
- **WHEN** a file contains only GGA sentences
- **THEN** one point results per sentence with a valid fix

### Requirement: Date and time
The import SHALL take the date from RMC or ZDA. For sentences without a date (GGA, GLL) it SHALL use the last known date. If the time jumps from shortly before to shortly after midnight, it SHALL advance the date by one day. Points before the first date SHALL get the first date of the file. It SHALL read two-digit years as 1980 to 2079. All times are UTC; fractions of seconds are kept.

#### Scenario: Midnight
- **WHEN** an RMC of 02.11.26 at 23:59:59 is followed only by GGA sentences at 00:00:00 and 00:00:01
- **THEN** these points have the date 3 November 2026

#### Scenario: GGA before the first RMC
- **WHEN** a file starts with three GGA sentences and only then an RMC with a date follows
- **THEN** the three points get the date of this RMC

#### Scenario: File without any date
- **WHEN** a file contains only GGA sentences
- **THEN** the points have no time, and a warning points to the missing date

### Requirement: Track splitting
All points of one read operation SHALL form one track. So several files are merged, as in the old app. Every file SHALL start a new segment. A new segment SHALL also start when the time of a point lies before that of the previous one.

#### Scenario: Three files of one day
- **WHEN** three NMEA files are read together
- **THEN** the recording contains one track with three segments

#### Scenario: Time jumps back
- **WHEN** within one file a point at 09:00:00 follows one at 10:00:05
- **THEN** a new segment starts with the second point

### Requirement: Waypoints
WPL sentences SHALL be taken over as waypoints with names.

#### Scenario: WPL sentence
- **WHEN** a file contains `$GPWPL,5144.4694,N,00840.8580,E,HOME*..`
- **THEN** the recording contains a waypoint "HOME" at 51.741157° N, 8.680967° E

### Requirement: Detection
The format detection SHALL recognize a file as NMEA 0183 if its first lines contain at least three syntactically valid NMEA sentences.

#### Scenario: Canon EOS log
- **WHEN** a `.LOG` file of the Canon EOS with a header line and following RMC/GGA sentences is checked
- **THEN** the detection suggests NMEA 0183
