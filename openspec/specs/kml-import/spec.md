# kml-import Specification

## Purpose
Reads KML files (Keyhole Markup Language) and KMZ archives as written by Google Earth, tracker apps, gpsbabel and myTracks. Paths and Google tracks become tracks, points become waypoints, without gpsbabel.

## Requirements

### Requirement: Tracks from paths
Every `Placemark` that contains at least one `LineString` or one `gx:Track` SHALL result in exactly one track. Every `LineString` and every `gx:Track` in it SHALL result in a segment of its own, in the order of the file, also inside `MultiGeometry` and `gx:MultiTrack`. The points of a `LineString` SHALL be taken over without a time. Segments without points SHALL be dropped.

#### Scenario: Simple path
- **WHEN** a `Placemark` contains a `LineString` with 100 coordinates
- **THEN** one track results with one segment of 100 points without a time

#### Scenario: Path of several parts
- **WHEN** a `Placemark` contains a `MultiGeometry` with two `LineString` of 3 coordinates each
- **THEN** one track results with two segments of 3 points each

#### Scenario: Google multi-track
- **WHEN** a `Placemark` contains a `gx:MultiTrack` with two `gx:Track`
- **THEN** one track results with two segments

### Requirement: Google tracks
A `gx:Track` SHALL build its points from the `gx:coord` elements (longitude, latitude and optionally altitude, separated by spaces). The n-th `when` SHALL be the time of the n-th point. If there are fewer `when` than `gx:coord`, the rest SHALL be taken over without a time and counted as a warning. Surplus `when` SHALL be ignored. An unreadable time SHALL count like a missing one.

#### Scenario: Track with times
- **WHEN** a `gx:Track` contains the times `2026-05-01T08:00:00Z` and `2026-05-01T08:00:10Z` and the coordinates `8 50 120` and `8.001 50.001 121`
- **THEN** two points result with latitude 50 and 50.001, longitude 8 and 8.001, the elevations 120 m and 121 m and these times

#### Scenario: Too few times
- **WHEN** a `gx:Track` contains 3 `gx:coord` but only 2 `when`
- **THEN** the third point has no time, and a warning counts one point without a date

### Requirement: Waypoints
Every `Point` in a `Placemark` SHALL result in a waypoint, also when the `Placemark` carries a time. The waypoint SHALL get `name` as name, `description` as comment and the time from `TimeStamp/when`, where present. An unreadable time of a waypoint SHALL be dropped without a warning. Excepted are the points of the folder "Points" according to the requirement "Times from the gpsbabel points folder".

#### Scenario: Marker with description
- **WHEN** a `Placemark` "Hut" with the description "Rest" and a `Point` occurs
- **THEN** a waypoint "Hut" with the comment "Rest" results

#### Scenario: Time-stamped points
- **WHEN** a folder "Photos" contains 5 `Placemark` with `Point` and `TimeStamp`
- **THEN** 5 waypoints with these times result and no track

### Requirement: Names of the tracks
A track SHALL get the `name` of its `Placemark`. If it is missing, empty or "Path" (the fixed placeholder of gpsbabel), the track SHALL get the name of the nearest enclosing `Folder` or `Document` with a name, otherwise the file name without extension.

#### Scenario: Track from gpsbabel
- **WHEN** a `Placemark` "Path" lies in a `Folder` "Morning run"
- **THEN** the track is called "Morning run"

#### Scenario: Path without a name
- **WHEN** the file `tour.kml` contains a `Placemark` without `name` directly below a `Document` without `name`
- **THEN** the track is called "tour"

### Requirement: Coordinates
A `coordinates` element SHALL contain tuples of longitude, latitude and optionally altitude, separated by commas. The tuples are separated by white space, and white space after a comma SHALL be tolerated. A tuple with fewer than two readable numbers MUST be discarded and counted as a warning. So must a position outside the valid range.

#### Scenario: Order longitude, latitude
- **WHEN** a `LineString` contains the tuple `8.5,50.25,100`
- **THEN** the point has the latitude 50.25 and the longitude 8.5

#### Scenario: Invalid latitude
- **WHEN** a tuple has the latitude 95
- **THEN** it is discarded, and a warning counts it

### Requirement: Altitude
The altitude of a `LineString`, a `Point` or a `gx:Track` SHALL be taken over if its `altitudeMode` or `gx:altitudeMode` is `absolute` or missing. With an explicitly set `clampToGround`, `relativeToGround`, `clampToSeaFloor` or `relativeToSeaFloor` the altitude SHALL stay empty, likewise when all altitudes of this geometry are 0.

#### Scenario: Export without altitude reference
- **WHEN** a `LineString` without `altitudeMode` contains the altitudes 100, 101 and 102
- **THEN** the points have these elevations

#### Scenario: On the ground
- **WHEN** a `LineString` with `altitudeMode` `clampToGround` contains the altitude 100
- **THEN** the points have no elevation

#### Scenario: Only zeros
- **WHEN** all tuples of a `LineString` have the altitude 0
- **THEN** the points have no elevation

### Requirement: Times from the gpsbabel points folder
A `Folder` "Points" SHALL provide the times to the track of its neighboring placemark if:
- it has exactly one `Placemark` with paths in the same container;
- every `Placemark` in it has a `Point` with a readable time;
- count and coordinates, to 0.000001°, match all points of this track.

Its points SHALL then not result in waypoints. Otherwise it SHALL be read like any other folder.

#### Scenario: Standard output of gpsbabel
- **WHEN** a `Folder` "Run" contains a `Placemark` "Path" with a `MultiGeometry` of two `LineString` of 3 points each and a `Folder` "Points" with 6 matching time-stamped points
- **THEN** one track "Run" results with two segments of 3 points each with these times and no waypoint

#### Scenario: Points do not match
- **WHEN** the `Folder` "Points" contains 5 points, but the track 6
- **THEN** the track stays without times, and the 5 points become waypoints

### Requirement: Tracks without times
If a file contains at least one track whose points all have no time, one single warning per file SHALL point that out. There SHALL be no warning "without a date" for these points.

#### Scenario: Export from myTracks
- **WHEN** a file contains a path with 500 coordinates without times
- **THEN** one track with 500 points results, and exactly one warning names the file as without times

### Requirement: Unknown content
The import SHALL skip `Polygon`, `LinearRing`, `Model`, overlays, styles, `ExtendedData`, `TimeSpan`, camera elements, `gx:Tour` and unknown elements without a warning. `NetworkLink` SHALL NOT be fetched. Folders and documents SHALL be read at any depth.

#### Scenario: Area boundary
- **WHEN** a file contains a `Placemark` with `Polygon` next to a path
- **THEN** only the track of the path results, without a warning

#### Scenario: Network link
- **WHEN** a file contains only a `NetworkLink`
- **THEN** no connection is made, and reading reports "no usable data"

### Requirement: Namespaces
The import SHALL treat files with the namespace of KML 2.2 (`http://www.opengis.net/kml/2.2`), with the Google namespaces `http://earth.google.com/kml/2.0`, `2.1` and `2.2` and without a namespace the same. The elements of Google's extension SHALL be recognized by their name, with or without a prefix.

#### Scenario: Old Google Earth file
- **WHEN** a file uses the namespace `http://earth.google.com/kml/2.1`
- **THEN** the same tracks and waypoints result as with KML 2.2

### Requirement: KMZ archives
A KMZ archive SHALL be read as a ZIP archive. The document `doc.kml` in the root directory is read, otherwise the first file with the extension `.kml` in the root directory. The entries can be uncompressed ("stored") or compressed with "deflate". All other entries SHALL be ignored. The same rules as for a KML file apply to the document.

#### Scenario: Archive with images
- **WHEN** a KMZ archive contains a compressed `doc.kml` with a path and two images
- **THEN** the track of the path results

#### Scenario: Other file name
- **WHEN** a KMZ archive contains only `tour.kml` and no `doc.kml`
- **THEN** `tour.kml` is read

### Requirement: Damaged KMZ archives
A wrong checksum (CRC-32) of the document read SHALL result in a warning that names the file. The document SHALL be read anyway. An archive without a KML document or a truncated archive MUST lead to an error with the file. So must an encrypted document, an unknown compression method, ZIP64 or a document that would be larger than 1 GiB when unpacked.

#### Scenario: Wrong checksum
- **WHEN** the CRC-32 of `doc.kml` does not match its content
- **THEN** the recording contains the content of the document, and a warning names the file

#### Scenario: No KML in the archive
- **WHEN** a ZIP archive that contains only images is read as KMZ
- **THEN** reading ends with an error that names the file

#### Scenario: Truncated archive
- **WHEN** a KMZ archive ends after half of it
- **THEN** reading ends with an error that names the file

### Requirement: Errors
A KML document that is not well-formed XML MUST lead to an error with file and line.

#### Scenario: Truncated file
- **WHEN** a KML file ends in the middle of a `coordinates` element
- **THEN** reading ends with an error that names file and line

### Requirement: Detection
A file SHALL be recognized as KML if its first element is `kml`, with or without a namespace prefix. A ZIP archive SHALL be recognized as KML if its first entry is a file with the extension `.kml`. The extensions `.kml` and `.kmz` SHALL only serve as a hint.

#### Scenario: KML with extension .xml
- **WHEN** a KML file has the extension `.xml`
- **THEN** the detection suggests KML

#### Scenario: Other ZIP archive
- **WHEN** a Word document (`.docx`, a ZIP archive) is checked
- **THEN** it is not recognized as KML
