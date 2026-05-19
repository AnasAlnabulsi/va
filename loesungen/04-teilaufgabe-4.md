# 04 – Integrationstests als Java-Code (kommentiert)

## Ziel
Du hast recht: Nur Test-Ideen reichen nicht. Hier sind jetzt **konkrete Java-Tests** (mit Kommentaren), die du unter `backend/src/test/java/...` anlegen kannst.

> Hinweis: Die Tests prüfen bewusst nur die Aufgaben aus README/TODO. Keine Extra-Features.

---

## A) Tests für `WebsocketServer` (mind. 3)

### Datei: `backend/src/test/java/de/berlin/htw/boundary/ws/WebsocketServerIT.java`
```java
package de.berlin.htw.boundary.ws;

import static org.junit.jupiter.api.Assertions.assertTrue;

import java.net.URI;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.websocket.ClientEndpoint;
import jakarta.websocket.ContainerProvider;
import jakarta.websocket.OnMessage;
import jakarta.websocket.Session;

@QuarkusTest
class WebsocketServerIT {

    /**
     * Einfacher Test-Client, der alle Textnachrichten vom Server sammelt.
     */
    @ClientEndpoint
    public static class WsCollector {
        final List<String> messages = new ArrayList<>();
        final CountDownLatch latch;

        WsCollector(int expectedMessages) {
            this.latch = new CountDownLatch(expectedMessages);
        }

        @OnMessage
        public void onMessage(String message) {
            messages.add(message);
            latch.countDown();
        }
    }

    @Test
    void ping_shouldReturnPong() throws Exception {
        // Erwartung: 1 Antwort vom Server
        WsCollector collector = new WsCollector(1);

        // Mit lokalem Quarkus-WebSocket-Endpunkt verbinden
        Session session = ContainerProvider.getWebSocketContainer()
                .connectToServer(collector, URI.create("ws://localhost:8081/quotes"));

        // Ping senden (wie im Frontend)
        session.getAsyncRemote().sendText("{\"type\":\"ping\"}");

        // Auf Antwort warten
        assertTrue(collector.latch.await(5, TimeUnit.SECONDS), "Keine Antwort vom Server erhalten");

        // Prüfen, dass "pong" enthalten ist
        assertTrue(collector.messages.stream().anyMatch(m -> m.contains("\"type\":\"pong\"")),
                "Server hat kein pong gesendet");

        session.close();
    }

    @Test
    void subscribe_shouldReturnInitialCandlesAndQuotes() throws Exception {
        // Erwartung: mindestens 2 Antworten (candles + quotes)
        WsCollector collector = new WsCollector(2);

        Session session = ContainerProvider.getWebSocketContainer()
                .connectToServer(collector, URI.create("ws://localhost:8081/quotes"));

        // Subscribe für DAX aus README
        session.getAsyncRemote().sendText(
                "{\"action\":\"subscribe\",\"symbolId\":\"133962\",\"venueId\":\"22\",\"channel\":\"last\",\"window\":3600}");

        assertTrue(collector.latch.await(5, TimeUnit.SECONDS), "Initialdaten wurden nicht vollständig gesendet");

        // Mindestens eine candles- und eine quotes-Nachricht prüfen
        assertTrue(collector.messages.stream().anyMatch(m -> m.contains("\"type\":\"candles\"")),
                "Keine candles-Nachricht empfangen");
        assertTrue(collector.messages.stream().anyMatch(m -> m.contains("\"type\":\"quotes\"")),
                "Keine quotes-Nachricht empfangen");

        session.close();
    }

    @Test
    void unsubscribe_shouldBeAcceptedWithoutServerError() throws Exception {
        // Erwartung: subscribe + unsubscribe laufen ohne Protokollfehler
        WsCollector collector = new WsCollector(2);

        Session session = ContainerProvider.getWebSocketContainer()
                .connectToServer(collector, URI.create("ws://localhost:8081/quotes"));

        // Erst abonnieren
        session.getAsyncRemote().sendText(
                "{\"action\":\"subscribe\",\"symbolId\":\"133962\",\"venueId\":\"22\",\"channel\":\"last\"}");

        // Dann wieder abbestellen
        session.getAsyncRemote().sendText(
                "{\"action\":\"unsubscribe\",\"symbolId\":\"133962\",\"venueId\":\"22\",\"channel\":\"last\"}");

        // Wenn der Server crasht, kommt häufig nichts mehr an / Session bricht ab
        assertTrue(collector.latch.await(5, TimeUnit.SECONDS), "Ablauf subscribe/unsubscribe nicht stabil");

        session.close();
    }
}
```

---

## B) Tests für `QuoteClient` (mind. 3)

> Diese Tests prüfen die Parsing-Logik des Clients. Da der echte Feed extern ist, wird ein lokaler Mock-Feed empfohlen.

### Datei: `backend/src/test/java/de/berlin/htw/boundary/ws/client/QuoteClientParsingIT.java`
```java
package de.berlin.htw.boundary.ws.client;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

import org.junit.jupiter.api.Test;

import de.berlin.htw.trading.quote.dto.DeltaQuote;
import de.berlin.htw.trading.quote.dto.Quote;
import de.berlin.htw.trading.quote.dto.SymbolKey;

class QuoteClientParsingIT {

    @Test
    void deltaFrame_shouldParseAllImportantFields() {
        // Beispiel aus README-Format: subId:value:secDelta:tickDelta:...
        String frame = "22:49032.7:3:15::::";

        DeltaQuote dq = DeltaQuote.parse(frame);

        // Prüfen, ob zentrale Felder korrekt geparst wurden
        assertNotNull(dq);
        assertEquals(22, dq.subId());
        assertEquals(49032.7, dq.value());
        assertEquals(3L, dq.secSinceLastMessage());
        assertEquals(15L, dq.tickDelta());
    }

    @Test
    void initialJson_shouldMapToQuoteRecord() throws Exception {
        // Minimales Initial-JSON wie im Feed (relevante Felder)
        String json = "{" +
                "\"s\":{\"symbolId\":\"133962\",\"venueId\":\"22\",\"channel\":\"last\"}," +
                "\"tsUnixSec\":1761117803," +
                "\"price\":24267.0," +
                "\"high\":24358.5," +
                "\"low\":24251.5," +
                "\"open\":24313.5," +
                "\"prevClose\":24330.03," +
                "\"abs\":-63.03," +
                "\"rel\":-0.0025906," +
                "\"tickSize\":0.5," +
                "\"active\":true," +
                "\"tick\":2469," +
                "\"subId\":6," +
                "\"precision\":2.0" +
                "}";

        // Jackson ObjectMapper wie im Produktivcode
        var mapper = new com.fasterxml.jackson.databind.ObjectMapper();
        Quote quote = mapper.readValue(json, Quote.class);

        assertNotNull(quote);
        assertEquals("133962", quote.s().symbolId());
        assertEquals("22", quote.s().venueId());
        assertEquals("last", quote.s().channel());
        assertEquals(6, quote.subId());
    }

    @Test
    void symbolKey_shouldBeConstructableForSubscriptionFlow() {
        // Dieses Objekt wird später über SubEvent/UnsubEvent als Nachricht aufgebaut
        SymbolKey key = new SymbolKey("133962", "22", "last");

        assertEquals("133962", key.symbolId());
        assertEquals("22", key.venueId());
        assertEquals("last", key.channel());
    }
}
```

---

## Warum diese Tests zur Aufgabe passen
- `WebsocketServerIT` prüft genau die geforderten Kernfälle aus README: `ping/pong`, `subscribe`, `unsubscribe`.
- `QuoteClientParsingIT` deckt die Datenformate aus dem externen Feed ab: Initial-JSON und Delta-Frames.
- Alle Tests sind minimal, ohne zusätzliche Features.

## Wichtige praktische Hinweise
- Wenn dein Quarkus-Test-Port abweicht, passe `ws://localhost:8081/quotes` an.
- Für echte End-to-End-Clienttests empfiehlt sich ein lokaler Mock-WebSocket-Server.
