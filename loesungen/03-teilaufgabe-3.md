# 03 – `SimpleQuoteConsumer` vervollständigen (nur TODOs, ausführlich Zeile für Zeile)

## Betroffene Datei
- `backend/src/main/java/de/berlin/htw/trading/quote/SimpleQuoteConsumer.java`

> Wichtig: In dieser Anleitung werden **nur** die zwei echten TODO-Stellen aus der Klasse gelöst:
> 1. Verarbeitung in `applyChanges(...)`
> 2. Rückgabe in `getQuotes(...)`

---

## TODO 1: `applyChanges(...)`

### Ersetze in `applyChanges(...)` nur den TODO-Bereich durch diesen Code (mit Kommentaren im Java-Code)
```java
// Menge aller Symbole, die in diesem Batch geändert wurden (thread-sicher)
Set<SymbolKey> updatedKeys = ConcurrentHashMap.newKeySet();

// Untere Zeitgrenze für Retention: jetzt - 1 Stunde
long minStart = (System.currentTimeMillis() / 1000) - retention.getSeconds();

// Jede Änderung aus dem Buffer verarbeiten
for (var cr : changes) {
    // Allgemeine ChangeRecord-Instanz als QuoteChange lesen
    var qc = (IMarketDataBuffer.QuoteChange) cr;

    // Quote-Reihe zum Symbol holen oder neu anlegen
    var dq = series.computeIfAbsent(qc.key(), k -> new ConcurrentLinkedDeque<>());

    // Neues Quote am Ende der Zeitreihe anhängen
    dq.addLast(qc.quote());

    // Letztes bekanntes Quote separat speichern (für schnelle Einzelabfrage)
    last.put(qc.key(), qc.quote());

    // Alte Daten am Anfang entfernen, solange sie älter als die Retention-Grenze sind
    while (!dq.isEmpty() && dq.peekFirst().tsUnixSec() < minStart) {
        dq.removeFirst();
    }

    // Symbol als aktualisiert markieren
    updatedKeys.add(qc.key());
}

// Für jedes aktualisierte Symbol ein asynchrones Quote-Event senden
for (var key : updatedKeys) {
    quoteEvent.fireAsync(new QuoteEvent(key));
}
```

### Erklärung Zeile für Zeile
1. `Set<SymbolKey> updatedKeys = ConcurrentHashMap.newKeySet();`  
   Erstellt eine thread-sichere Menge für alle Symbole, die in diesem Batch geändert wurden.
2. `long minStart = ...`  
   Berechnet die Zeitgrenze (jetzt minus Retention in Sekunden). Alles ältere wird entfernt.
3. `for (var cr : changes) {`  
   Geht jede Änderung aus dem Buffer nacheinander durch.
4. `var qc = (IMarketDataBuffer.QuoteChange) cr;`  
   Wandelt den allgemeinen Change in einen Quote-Change um, damit `key()` und `quote()` verfügbar sind.
5. `var dq = series.computeIfAbsent(...)`  
   Holt die Quote-Reihe für das Symbol oder legt eine neue leere Deque an.
6. `dq.addLast(qc.quote());`  
   Hängt das neueste Quote ans Ende der Zeitreihe.
7. `last.put(qc.key(), qc.quote());`  
   Speichert das letzte bekannte Quote pro Symbol für schnelle Einzelabfragen.
8. `while (!dq.isEmpty() && dq.peekFirst().tsUnixSec() < minStart) { dq.removeFirst(); }`  
   Entfernt alte Quotes am Anfang der Deque, bis alle verbleibenden Einträge im Retention-Fenster liegen.
9. `updatedKeys.add(qc.key());`  
   Merkt das Symbol als „geändert“, damit später genau dafür Events gesendet werden.
10. `for (var key : updatedKeys) { quoteEvent.fireAsync(new QuoteEvent(key)); }`  
    Sendet pro geändertem Symbol ein asynchrones Update-Event.

---

## TODO 2: `getQuotes(...)`

### Ersetze die TODO-Methode komplett durch diesen Code (mit Kommentaren im Java-Code)
```java
public List<Quote> getQuotes(SymbolKey key, Duration window) {
    // Gespeicherte Quote-Reihe zum Symbol holen
    var dq = series.get(key);

    // Wenn keine Daten vorhanden sind, leere Liste zurückgeben
    if (dq == null || dq.isEmpty()) {
        return List.of();
    }

    // Untere Zeitgrenze aus gewünschtem Fenster berechnen
    long minTs = (System.currentTimeMillis() / 1000) - window.getSeconds();

    // Ergebnisliste für alle passenden Quotes
    List<Quote> out = new ArrayList<>();

    // Nur Quotes übernehmen, die innerhalb des Fensters liegen
    for (Quote q : dq) {
        if (q.tsUnixSec() >= minTs) {
            out.add(q);
        }
    }

    // Gefilterte Liste zurückgeben
    return out;
}
```

### Erklärung Zeile für Zeile
1. `var dq = series.get(key);`  
   Holt die gespeicherte Quote-Reihe für das gewünschte Symbol.
2. `if (dq == null || dq.isEmpty()) { return List.of(); }`  
   Wenn es noch keine Daten gibt, wird eine leere Liste zurückgegeben.
3. `long minTs = ...`  
   Berechnet die untere Zeitgrenze aus dem gewünschten Fenster (`window`).
4. `List<Quote> out = new ArrayList<>();`  
   Ergebnisliste für passende Quotes.
5. `for (Quote q : dq) { ... }`  
   Iteriert über alle vorhandenen Quotes der Reihe.
6. `if (q.tsUnixSec() >= minTs) { out.add(q); }`  
   Übernimmt nur Quotes, die im angeforderten Zeitfenster liegen.
7. `return out;`  
   Gibt genau die gefilterte Zeitreihe zurück.

---

## Warum genau diese TODO-Lösung korrekt ist
- Sie ergänzt exakt die fehlende Verarbeitung von eingehenden Quote-Änderungen.
- Sie implementiert exakt die fehlende Fensterabfrage für historische Quotes.
- Sie verändert **keine** Methodensignatur und **keine** zusätzliche Architektur.
- Sie bleibt minimal und passt zur bestehenden Logik in der Klasse.

## Kurz-Erklärung für den Professor
„Ich habe nur die beiden TODOs in `SimpleQuoteConsumer` umgesetzt: erstens das Einpflegen und Aufräumen der Quote-Reihe bei Änderungen, zweitens die gefilterte Rückgabe nach Zeitfenster für das Frontend.“
