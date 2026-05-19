# 04 – Integrationstests (mindestens je 3 für Server und Client)

## Betroffene Bereiche
- Neue Testklassen unter `backend/src/test/java/...`

## Was laut README verlangt wird
- Mindestens 3 Integrationstests für `WebsocketServer`
- Mindestens 3 Integrationstests für `QuoteClient`

## Aktueller Stand
Im Repository gibt es derzeit **keine** Testklassen unter `backend/src/test`.

## Minimale notwendige Testfälle
### A) WebsocketServer
1. `ping`-Nachricht -> Antwort `pong`.
2. `subscribe` -> initiale Antwort mit `type="candles"` und `type="quotes"`.
3. `unsubscribe` -> keine weiteren Pushes für das abgemeldete Symbol.

### B) QuoteClient
1. Initial-JSON wird als `InitialQuoteEvent` publiziert.
2. Delta-Frame wird als `QuoteDeltaEvent` publiziert.
3. Welcome-Frame (`[stock3-...`) wird ignoriert.

## Beispielhafte Teststrategie
- `@QuarkusTest` für Integrationsumgebung.
- Für Client-Tests: lokaler Mock-WebSocket-Server (statt echter externer URL).
- Event-Ergebnisse über testweise Observer/Queues verifizieren.

## Erklärung für den Professor
„Die Aufgabe verlangt explizit Integrations- statt nur Unit-Tests. Daher würde ich die Quarkus-Laufzeit hochfahren, WebSocket-Verbindungen real öffnen und den Eventfluss Ende-zu-Ende prüfen.“

## Unsicherheiten / Hinweise
- Externe URL kann in CI instabil sein; deshalb besser mit lokalem Mock-Server testen.
- Ohne Implementierung der TODOs sind die Tests erwartbar rot.
