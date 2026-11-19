# GPSLoggerKit

Standalone Swift library (MIT) for file formats and logger protocols of GPS loggers, with a common data model and
GPX 1.1 output. Used by myTracks Loader (`mytracks/gpsloader`, GPL) and myTracks Neo (App Store, closed source), both
by package URL and version.

## Way of working

- Spec-driven with **OpenSpec** (`openspec/`). Before new features, create a change (`/opsx:propose`) instead of
  implementing directly.
- The context and rules in `openspec/config.yaml` are binding. Points that are "not yet confirmed" there must be
  clarified with the developer before implementing them.
- Language (decided on 2026-11-19): everything in this repository is English, including OpenSpec artifacts. The
  conversation with the developer is in German. Localized messages of the package exist in German and English.
- A change modifies only this repository. The apps take over new versions with changes of their own in their
  repositories.
- The planning history up to 2026-11-19 is in `mytracks/gpsloader` under `openspec/changes/archive/` (in German).

## Clean-room (important)

- **MIT, no GPL code.** Never read, translate or copy source code of gpsbabel, igotu2gpx, wsg2000.c or other GPL/CC-BY-SA
  projects. Permitted: public specifications, the developer's own myTracks code, black-box observation.
  `PROVENANCE.md` names the sources of every format; a test enforces the entry.
- Ports from GPL sources belong in `mytracks/GPSLoggerKitGPL`, never here.

## Important places

- `Sources/GPSLoggerKit/`: data model (`Model/`), formats (`Formats/`), device interface (`Device/`) and protocols
  (`Devices/`), processing (`Processing/`), GPX output (`Output/`).
- `Tests/GPSLoggerKitTests/`: tests with synthetic fixtures. No real location data.
- Old sources, read only, in the neighboring checkout: `../gpsloader/input/mytracks4mac/` (myTracks), functional analysis
  `../gpsloader/docs/legacy-analysis.md`.

## Technology

- macOS 12+ and iOS/iPadOS 27+; Swift 6 language mode, strict concurrency; no dependencies.
- No transports (serial, HID, Bluetooth) in the package; the app provides them through `DeviceConnection`.
- Tests: `swift test` and on the iOS simulator (see README).
- Releases: `Scripts/release.sh <version>` (tag and GitHub release, only after the developer's approval).
