# Proposal: GPSLoggerKit als eigenständiges Repository

## Why

GPSLoggerKit ist am 2026-11-19 aus `mytracks/gpsloader` in dieses Repository umgezogen (Change `split-repositories`
dort). Die Anforderung „Eigenständigkeit“ beschreibt noch den alten Ort: Sie prüft, ob der Ordner
`Packages/GPSLoggerKit` herausgelöst aus dem gpsloader-Repository baut. Jetzt zählt, dass das Paket aus einem frischen
Checkout dieses Repositorys und als Abhängigkeit per URL baut.

## What Changes

- „Eigenständigkeit“ verlangt den Build aus einem frischen Checkout dieses Repositorys und per Paket-URL, ohne andere
  Repositories und ohne Pakete von Drittanbietern.
- Keine Änderung am Code: Das Paket hängt schon heute von nichts anderem ab.

## Betroffene Formate und Geräte

Keine. Formate, Protokolle und Datenmodell bleiben unverändert.

## Kompatibilität

Keine Änderung für Nutzer; erste Version ist 0.1.0.

## Nicht im Umfang

- Release-Skript und Release-Notes (eigene Schritte des Umzugs, ohne Anforderungsänderung).
- Änderungen an Formaten, Protokollen oder der öffentlichen Schnittstelle.

## Capabilities

### New Capabilities

(keine)

### Modified Capabilities

- `logger-kit`: „Eigenständigkeit“ bezieht sich auf dieses Repository und die Einbindung per URL.

## Impact

Nur `openspec/specs/logger-kit/spec.md`.
