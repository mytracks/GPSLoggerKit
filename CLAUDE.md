# GPSLoggerKit

Eigenständige Swift-Bibliothek (MIT) für Dateiformate und Logger-Protokolle von GPS-Loggern, mit gemeinsamem
Datenmodell und GPX-1.1-Ausgabe. Genutzt von myTracks Loader (`mytracks/gpsloader`, GPL) und myTracks Neo
(App Store, ohne offenen Quellcode), jeweils per Paket-URL und Version.

## Arbeitsweise

- Spec-getrieben mit **OpenSpec** (`openspec/`). Vor neuen Features einen Change anlegen (`/opsx:propose`), nicht direkt
  implementieren.
- Verbindlich sind Kontext und Regeln in `openspec/config.yaml`. Punkte, die dort „noch nicht bestätigt“ sind, vor der
  Umsetzung mit dem Entwickler klären.
- Kommunikation und OpenSpec-Artefakte auf Deutsch. Code, Bezeichner, Kommentare, README und PROVENANCE auf Englisch.
- Ein Change ändert nur dieses Repository. Die Apps übernehmen neue Versionen mit eigenen Changes in ihren Repositories.
- Die Planungshistorie bis 2026-11-19 liegt in `mytracks/gpsloader` unter `openspec/changes/archive/`.

## Clean-Room (wichtig)

- **MIT, kein GPL-Code.** Quelltext von gpsbabel, igotu2gpx, wsg2000.c und anderen GPL-/CC-BY-SA-Projekten weder lesen
  noch übersetzen noch kopieren. Zulässig: öffentliche Spezifikationen, eigener myTracks-Code des Entwicklers,
  Black-Box-Beobachtung. `PROVENANCE.md` nennt die Quellen jedes Formats; ein Test erzwingt den Eintrag.
- Portierungen aus GPL-Quellen gehören nach `mytracks/GPSLoggerKitGPL`, nie hierher.

## Wichtige Orte

- `Sources/GPSLoggerKit/`: Datenmodell (`Model/`), Formate (`Formats/`), Geräteschnittstelle (`Device/`) und Protokolle (`Devices/`), Verarbeitung (`Processing/`), GPX-Ausgabe (`Output/`).
- `Tests/GPSLoggerKitTests/`: Tests mit synthetischen Fixtures. Keine echten Ortsdaten.
- Alt-Quellen nur lesen, im Nachbar-Checkout: `../gpsloader/input/mytracks4mac/` (myTracks), fachliche Analyse
  `../gpsloader/docs/legacy-analysis.md`.

## Technik

- macOS 12+ und iOS/iPadOS 27+; Swift 6 Language Mode, Strict Concurrency; keine Abhängigkeiten.
- Keine Transporte (seriell, HID, Bluetooth) im Paket; die App stellt sie über `DeviceConnection` bereit.
- Tests: `swift test` und auf dem iOS-Simulator (siehe README).
- Releases: `Scripts/release.sh <Version>` (Tag und GitHub-Release, nur nach Freigabe des Entwicklers).
