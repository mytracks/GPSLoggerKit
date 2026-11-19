# gns2000-device Specification

## Purpose
Reads the track of the Bluetooth logger GNS 2000 (also GNS 2000 plus) and erases its memory, clean-room in GPSLoggerKit after the developer's own myTracks driver.

## Requirements

### Requirement: GNS 2000 sentences
The driver SHALL send commands as `$<content>*<xx>` with CR LF, where `xx` is the XOR of all characters of the content as two lowercase hex digits. It SHALL read CR LF lines. It MUST ignore lines with a missing or wrong checksum; the checksum is compared case-insensitively.

#### Scenario: Checksum of a command
- **WHEN** the driver sends the read command
- **THEN** it sends `$PGNS,R*` with the XOR of `PGNS,R` as two lowercase hex digits and CR LF

#### Scenario: Garbled line
- **WHEN** a `$PMTKLOX` line has a wrong checksum
- **THEN** the driver does not evaluate it and continues reading

### Requirement: Device readiness
After opening the connection the driver SHALL wait for the first line that starts with `$`, because the device sends NMEA sentences continuously. Only then SHALL it send its command. If no such line arrives within 10 seconds, the operation MUST end with a timeout.

#### Scenario: Device sends NMEA
- **WHEN** the device sends `$GPGGA,…` after opening
- **THEN** the driver sends its command afterwards

#### Scenario: Device silent
- **WHEN** the device sends nothing for 10 seconds after opening
- **THEN** the operation ends with a timeout

### Requirement: Read the memory
The driver SHALL send `PGNS,R` and wait for `$PMTKLOX,0,<count>`. Then it SHALL read `$PMTKLOX,1,<index>,<words…>` up to `$PMTKLOX,2` and report the progress as read of announced packets. Every four words of 8 hex digits form a 16-byte record in text order. If 10 seconds pass between two lines without data, the operation MUST end with a timeout.

#### Scenario: Reading
- **WHEN** the device announces and sends 3 packets with 40 records in total
- **THEN** the driver returns the valid points from them and reports the progress from 0 to 3 packets

#### Scenario: End missing
- **WHEN** the device falls silent after the last data packet
- **THEN** the operation ends with a timeout after 10 seconds

### Requirement: Records
A record SHALL contain in little-endian format the Unix timestamp (UInt32, UTC) at 0, the fix type (UInt8) at 4, latitude and longitude (Float32) at 5 and 9 and the elevation in meters (Int16) at 13. The driver MUST discard records consisting only of `FF` and those with a fix type outside 1 to 4.

#### Scenario: Conversion
- **WHEN** a record contains the timestamp 1793620800, fix type 3, latitude 50.5, longitude 8.25 and elevation 123
- **THEN** a point at 50.5° N and 8.25° E results, 123 m high, with the time 2026-11-02T12:00:00Z

#### Scenario: Invalid fix
- **WHEN** a record has the fix type 0
- **THEN** no point results from it

### Requirement: Tracks of the GNS 2000
If the timestamp of a point is smaller than that of the previous one, the driver SHALL start a new track. Every track SHALL be named after the time of its first point, in the format `yyyy-MM-dd HH:mm` in UTC. Empty tracks are dropped.

#### Scenario: Time jumps back
- **WHEN** points of 2026-11-02 are followed by points of 2026-11-01
- **THEN** two tracks result, "2026-11-02 12:00" and "2026-11-01 08:00"

### Requirement: Erase the memory of the GNS 2000
The driver SHALL send `PGNS,C` and wait for `$PMTK001`. After every run of more than 10 consecutive lines that do not start with `$P`, it SHALL send the command again, at most five times in total. If no confirmation arrives, the operation MUST end with a protocol error.

#### Scenario: Erasing confirmed
- **WHEN** the device answers `PGNS,C` with `$PMTK001,…`
- **THEN** erasing ends successfully

#### Scenario: Device misses the command
- **WHEN** the device sends only NMEA sentences after the first `PGNS,C`
- **THEN** the driver sends the command again after 11 such lines
