# Überblick über die Übungsaufgabe

## Kontext
Dieses Repository enthält eine **Verteilte-Anwendungen-Übung** mit Quarkus (Backend) und Vue (Frontend). Laut README ist das Frontend bereits fertig, während im Backend gezielt unvollständige Stellen ergänzt werden sollen.

## Ziel der Aufgabe
Die Gesamtaufgabe besteht aus vier Teilen:
1. WebSocket-Server im Backend fertigstellen (`WebsocketServer`).
2. WebSocket-Client für externe Kursdaten fertigstellen (`QuoteClient`, `QuoteController`).
3. Kursverarbeitung für Ticker/Line-Chart ergänzen (`SimpleQuoteConsumer`).
4. Integrationstests für Server und Client schreiben.

## Wichtige gefundene TODO-Bereiche
- `WebsocketServer.java`
- `QuoteClient.java`
- `QuoteController.java`
- `SimpleQuoteConsumer.java`

## Vorgehensweise bei der Lösung
- Nur die markierten Stellen ergänzen.
- Bestehende Strukturen (Events, DTOs, Consumer-Pattern) weiterverwenden.
- Keine unnötigen Signaturänderungen.
- Keine zusätzlichen Features außerhalb der Aufgabenstellung.

## Annahmen
- Die Architektur mit CDI-Events (`SubEvent`, `UnsubEvent`, `InitialQuoteEvent`, `QuoteDeltaEvent`) ist bewusst vorgegeben und soll nicht umgebaut werden.
- Die Nachrichtenformate müssen strikt zu Frontend (`frontend/src/services/ws.ts`) und README passen.
- Da aktuell keine Testklassen vorhanden sind, werden in den Lösungsdateien konkrete Testideen und minimale Integrations-Testfälle beschrieben.
