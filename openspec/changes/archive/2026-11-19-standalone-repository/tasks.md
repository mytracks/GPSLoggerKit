# Tasks: standalone-repository

## 1. Anforderung

- [x] 1.1 Prüfen, dass ein frischer Klon dieses Repositorys ohne weitere Repositories baut und testet. Prüfen: `swift build` und `swift test` sind in einem Klon im Scratchpad grün.
- [x] 1.2 Die Einbindung per URL prüfen, sobald 0.1.0 veröffentlicht ist (Szenario „Einbindung per URL“). Prüfen: Ein leeres Testpaket bindet `https://github.com/mytracks/GPSLoggerKit.git` mit `.upToNextMinor(from: "0.1.0")` ein und baut für macOS und iOS.
