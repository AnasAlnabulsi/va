# 02 – WebSocket-Client (`QuoteClient`, `QuoteController`) (Copy-Paste-fähig)

## Betroffene Dateien
- `backend/src/main/java/de/berlin/htw/boundary/ws/client/QuoteController.java`
- `backend/src/main/java/de/berlin/htw/boundary/ws/client/QuoteClient.java`

## A) `QuoteController` – fehlende Verbindung

### Zusätzliche Imports
```java
import java.net.URI;
import jakarta.websocket.ContainerProvider;
```

### Vollständiger Inhalt für `start()`
```java
@PostConstruct
public void start() {
    logger.info("Starting QuoteController...");
    try {
        var container = ContainerProvider.getWebSocketContainer();
        this.session = container.connectToServer(QuoteClient.class, URI.create("wss://quotepush.stock3.com/delta"));
    } catch (Exception e) {
        logger.error("Failed to start QuoteController.", e);
        System.exit(1);
    }

    logger.info("QuoteController started.");
}
```

## B) `QuoteClient` – vollständige Endpoint-Methoden

### Zusätzliche Imports
```java
import jakarta.websocket.OnClose;
import jakarta.websocket.OnError;
import jakarta.websocket.OnMessage;
import jakarta.websocket.OnOpen;
import jakarta.websocket.Session;
```

### In die Klasse `QuoteClient` einfügen
```java
@OnOpen
public void onOpen(Session session) {
    logger.info("Connected to stock3 quote feed.");
}

@OnMessage
public void onMessage(String message) {
    if (message == null || message.isBlank()) {
        return;
    }

    try {
        // Willkommenstext ignorieren
        if (message.startsWith("[stock3-")) {
            return;
        }

        // Initialnachricht als JSON
        if (message.startsWith("{")) {
            Quote q = MAPPER.readValue(message, Quote.class);
            if (q != null && q.subId() != null && q.s() != null) {
                subMap.put(q.subId(), q.s());
                quoteEvent.fireAsync(new InitialQuoteEvent(q));
            }
            return;
        }

        // Deltanachricht (z. B. 22:49032.7:3:::::)
        DeltaQuote dq = DeltaQuote.parse(message);
        if (dq != null) {
            quoteDeltaEvent.fireAsync(new QuoteDeltaEvent(dq));
        }
    } catch (Exception e) {
        logger.errorf(e, "Failed to parse quote message: %s", message);
    }
}

@OnClose
public void onClose(Session session) {
    logger.info("Connection to stock3 quote feed closed.");
}

@OnError
public void onError(Session session, Throwable throwable) {
    logger.error("Error in stock3 quote feed.", throwable);
}
```

## Warum das korrekt ist
- README beschreibt genau diese zwei Datenarten (einmal JSON initial, danach Delta-Frames).
- Die Events `InitialQuoteEvent` und `QuoteDeltaEvent` sind dafür schon vorbereitet.

## Professor-Erklärung
„Ich habe nur den Transport-Layer vervollständigt: externe WS-Verbindung öffnen und Nachrichtenformat korrekt in bestehende Domain-Events überführen.“
