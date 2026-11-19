## MODIFIED Requirements

### Requirement: Eigenständigkeit
Das Paket MUST aus einem frischen Checkout seines Repositorys und als Abhängigkeit per Repository-URL bauen, ohne andere Repositories und ohne Pakete von Drittanbietern. Es darf nur die Systembibliotheken der Plattform nutzen.

#### Scenario: Herausgelöst bauen
- **WHEN** das Repository frisch geklont und dort `swift build` und `swift test` ausgeführt werden
- **THEN** gelingen Build und Tests, ohne dass weitere Repositories vorhanden sind

#### Scenario: Einbindung per URL
- **WHEN** ein leeres Swift-Paket GPSLoggerKit per Repository-URL und veröffentlichter Version als Abhängigkeit angibt und für macOS und iOS gebaut wird
- **THEN** löst der Swift Package Manager nur dieses Paket auf, und beide Builds gelingen
