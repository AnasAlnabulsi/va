# 03 – `SimpleQuoteConsumer` vervollständigen (Copy-Paste-fähig)

## Betroffene Datei
- `backend/src/main/java/de/berlin/htw/trading/quote/SimpleQuoteConsumer.java`

## 1) `applyChanges(...)` komplett ersetzen
```java
@Override
protected void applyChanges(List<ChangeRecord> changes) {

    Set<SymbolKey> updatedKeys = ConcurrentHashMap.newKeySet();

    // Retention-Grenze: alles älter als 1 Stunde wird entfernt
    long minStart = (System.currentTimeMillis() / 1000) - retention.getSeconds();

    for (var cr : changes) {
        var qc = (IMarketDataBuffer.QuoteChange) cr;

        // Zeitreihe für Symbol holen oder neu anlegen
        var dq = series.computeIfAbsent(qc.key(), k -> new ConcurrentLinkedDeque<>());

        // Neues Quote anhängen
        dq.addLast(qc.quote());

        // Last-known Quote für Ticker/Initialdaten merken
        last.put(qc.key(), qc.quote());

        // Alte Daten aus der Reihe entfernen
        while (!dq.isEmpty() && dq.peekFirst().tsUnixSec() < minStart) {
            dq.removeFirst();
        }

        updatedKeys.add(qc.key());
    }

    // Für alle betroffenen Symbole Update-Event senden
    for (var key : updatedKeys) {
        quoteEvent.fireAsync(new QuoteEvent(key));
    }
}
```

## 2) `getQuotes(...)` komplett ersetzen
```java
public List<Quote> getQuotes(SymbolKey key, Duration window) {
    var dq = series.get(key);
    if (dq == null || dq.isEmpty()) {
        return List.of();
    }

    long minTs = (System.currentTimeMillis() / 1000) - window.getSeconds();
    List<Quote> out = new ArrayList<>();

    for (Quote q : dq) {
        if (q.tsUnixSec() >= minTs) {
            out.add(q);
        }
    }

    return out;
}
```

## Warum das korrekt ist
- Gleiche Logik wie beim `CandleQuoteConsumer`: Änderungen anwenden, alte Daten verwerfen, Event feuern.
- Frontend bekommt dadurch historische Quote-Liste (`quotes`) und Live-Quote (`quote`).

## Professor-Erklärung
„Ich habe die Quote-Reihe analog zum Candle-Consumer implementiert: append + retention + Event. Dadurch funktionieren Ticker und Line-Chart.“
