# Architektur: Coworker-Infrastruktur für Krypto- & Aktien-Trading

**Status:** ✅ Architektur-Planung abgeschlossen — Umsetzung offen, erfolgt zuhause mit Homelab-Zugriff
(kein Netzwerkzugriff auf die Proxmox-Nodes von dieser Remote-Umgebung aus).

> **Hinweis:** Ein Teil der hier beschriebenen Container/Dienste existiert im Homelab bereits. Bevor
> irgendetwas neu aufgesetzt wird, zuerst die Bestandsaufnahme aus der ToDo-Liste unten durchführen.

## Überblick

Pro Thema (Krypto, Aktien) die gleiche Struktur, komplett getrennt voneinander:

```
OpenClaw-Agent (24/7, im jeweiligen Coworker-CT)
        │  scraped/rohe Marktdaten, News, Sentiment
        ▼
n8n-Workflow "Job Intake & Processing" (bestehende n8n-Instanz, CT200)
        │  Normalisierung, Deduplizierung, Enrichment, Indikator-/Signal-Berechnung
        │  (inkl. Candlestick-Muster-Erkennung via TA-Lib, siehe Komponente 6)
        ▼
PostgreSQL-DB (thema-eigen, auf bestehender Instanz CT110)
        │  strukturierte Ablage (Kurse, News, Signale)
        ▼
Hermes-Agent (bestehend, .26:5058) — nur bei Bedarf getriggert
        │  erweiterte Entscheidung (kaufen/verkaufen/beobachten/Alarm)
        ▼
Aktion: Alert (Telegram, sobald aktiviert) / Log / Eintrag in Qdrant ai_memory (Langzeitgedächtnis)
        ├─ speist auch trading.html-Dashboard (CT120)
        └─ Grafana-Dashboard (.104:3000): Kursverlauf + KAUF/VERKAUF-Signalmarker, live
```

## Komponenten

### 1. Zwei neue LXC-Coworker-Container
- **Krypto-Coworker** und **Aktien-Coworker**, je ein eigener Container (nicht Services in CT205).
- Vermutlich auf **node-02** (192.168.0.92), da dort bereits AI/Apps laufen (Ollama, n8n, Perplexica,
  Hermes) — endgültige Node-Wahl und freie CT-ID/IP erst vor Ort prüfen (`pct list` auf allen 3 Nodes).
- Enthält jeweils den OpenClaw-Deep-Search-Agenten (genaue Konfiguration/Deployment folgt zuhause, da
  OpenClaw bereits im Homelab existiert und von hier aus nicht einsehbar ist).
- Klare Trennung: Krypto- und Aktien-Coworker teilen sich keine Prozesse, nur die gemeinsame Infrastruktur
  (n8n, Hermes, Postgres-Instanz).

**OpenClaw — konkrete Zielquellen statt nur "Deep Web Search":**
- **Aktien-Coworker**: TradingView (Kurscharts, technische Indikatoren, Watchlist-Symbole) als
  Hauptquelle, ergänzt durch allgemeine Finanznews-Suche.
- **Krypto-Coworker**: CoinMarketCap (Preise, Marktkapitalisierung, Volumen) als Hauptquelle, ergänzt
  durch TradingView (Krypto-Charts/Indikatoren) und allgemeine Krypto-News-Suche.
- **Hinweis zur Umsetzung**: CoinMarketCap bietet eine offizielle (auch kostenlos nutzbare) API — die
  sollte OpenClaw bevorzugt nutzen statt HTML-Scraping (stabiler, kein Ban-Risiko). TradingView hat keine
  offizielle Massen-Daten-API; hier bieten sich entweder vorsichtiges Scraping mit Rate-Limiting/Caching
  oder TradingView-eigene Alert-Webhooks (Pine-Script-Alerts direkt an einen n8n-Webhook) als robustere
  Alternative an. Die konkrete Wahl klären wir zuhause mit Blick auf die bestehende OpenClaw-Konfiguration.
- Zusätzliche Quellen (News, Social/Sentiment, sonstige Deep-Web-Suche) bleiben Teil von OpenClaws
  Aufgabe, TradingView/CoinMarketCap sind ab jetzt feste Pflichtquellen.

### 2. Hermes-Agent für die erweiterte Entscheidung
Empfehlung: **keinen dritten/vierten Hermes deployen**, sondern den bestehenden Hermes-Agent
(.26:5058, POST `/agent {task}`) um zwei neue Tools/Endpoints erweitern (`krypto_decision`,
`aktien_decision`). Begründung:
- Hermes arbeitet ereignisgesteuert und läuft nicht rund um die Uhr — genau das macht ihn günstig für die
  Entscheidungsschicht, ein Duplikat pro Thema würde diesen Vorteil ohne Nutzen verdoppeln.
- Beide Themen können unterschiedliche Prompts/Tools innerhalb des gleichen Agents bekommen, ohne die
  Trennung der Datenbanken/Coworker zu verletzen.

### 3. Zwei neue PostgreSQL-Datenbanken
- Auf der bestehenden PostgreSQL-Instanz (CT110), analog zu `n8n` und `aimemory`:
  `krypto_trading` und `aktien_trading`.
- Konsistent mit dem bisherigen Stack (kein neuer DB-Typ), einfacher zu warten/backuppen.
- Empfehlung für die Tabellen später: klare Trennung zwischen Rohdaten (Scrape-Log), verarbeiteten
  Marktdaten/Kursen und generierten Signalen/Entscheidungen — hilft beim späteren Backtesting.

### 4. n8n für Informationsverarbeitung & Job-Annahme
- **Keine Neuinstallation** — n8n läuft bereits (CT200). Es werden nur die bestehenden Workflows um die
  zwei neuen Themen erweitert/angepasst.
- Pro Thema eine eigene Workflow-Gruppe: Job-Intake-Webhook (nimmt Aufträge/Scrape-Ergebnisse an) →
  Normalisierung/Enrichment → Berechnung der technischen Indikatoren/Signale → Ablage in der jeweiligen
  Postgres-DB (inkl. Signal-Tabelle, siehe Grafana unten) → bedingter Trigger an Hermes.

### 5. Grafana — Kauf-/Verkaufssignal-Dashboard
- **Keine Neuinstallation** — Grafana läuft bereits (.104:3000). Es wird um ein neues Dashboard pro Thema
  erweitert (Krypto, Aktien).
- Grafana visualisiert nur, gerechnet wird vorher in n8n (Schritt "Berechnung der Indikatoren/Signale") und
  in einer Postgres-Tabelle `signals` abgelegt (Spalten u. a.: Zeitstempel, Symbol, Indikator-Werte,
  Signal-Typ `KAUF`/`VERKAUF`/`HALTEN`, Konfidenz/Begründung).
- Dashboard-Panels: Kursverlauf mit Signal-Markern, ein "aktuelle Empfehlung"-Stat-Panel pro Symbol, sowie
  Grafana-Alerting, das bei neuem `KAUF`/`VERKAUF`-Signal benachrichtigt (z. B. später via Telegram-Bot).
- Damit ist jederzeit auf einen Blick sichtbar, wann laut Berechnung ein guter Einstiegs- bzw.
  Verkaufszeitpunkt ist — nicht nur im Chat, sondern dauerhaft im Dashboard.

### 6. Candlestick-Muster als Indikator
- **Kein Neubau der Erkennung** — dafür **TA-Lib** nutzen (Standard-Bibliothek für technische Analyse,
  Python/Node-Bindings vorhanden), die ~60 fertige, geprüfte `CDL*`-Erkennungsfunktionen für praktisch
  alle gängigen Candlestick-Muster mitbringt.
- Läuft als zusätzlicher Schritt in der bestehenden n8n-Pipeline (Function-/Code-Node oder kleiner
  Python-Sidecar) auf den OHLC-Daten von TradingView/CoinMarketCap — gilt für **beide Themen** (Krypto
  und Aktien) gleich.
- Ergebnis (Muster-Name, Richtung, Verlässlichkeits-Einschätzung) wird zusätzlich in die `signals`-Tabelle
  geschrieben und im Grafana-Dashboard als weitere Marker-Kategorie angezeigt — gleicher Signalfluss wie
  bereits geplant, nur mit einer zusätzlichen, breiteren Signalquelle neben der eigentlichen
  Signal-Strategie.

**Musterliste (bullisch und bärisch, gilt gleich für Krypto und Aktien):**

| Kategorie | Muster |
|---|---|
| Bullische Umkehr (Ende Abwärtstrend) | Hammer, Inverted Hammer (braucht Bestätigung), Bullish Engulfing, Morning Star, Piercing Line, Three White Soldiers, Bullish Harami, Tweezer Bottom, Bullish Abandoned Baby (selten), Dragonfly Doji |
| Bärische Umkehr (Ende Aufwärtstrend) | Hanging Man, Shooting Star, Bearish Engulfing, Evening Star, Dark Cloud Cover, Three Black Crows, Bearish Harami, Tweezer Top, Bearish Abandoned Baby (selten), Gravestone Doji |
| Neutral/Fortsetzung (Richtung kontextabhängig) | Doji, Spinning Top, Marubozu (Richtung je Farbe, starkes Momentum), Rising Three Methods (bullische Fortsetzung), Falling Three Methods (bärische Fortsetzung) |

**Weitere wichtige Muster (Bestätigung / starke Umkehr / Fortsetzung):**

| Kategorie | Muster |
|---|---|
| Bestätigungsmuster (verstärken Harami/Engulfing) | Three Inside Up/Down, Three Outside Up/Down |
| Starke Umkehrsignale | Bullish/Bearish Belt Hold, Bullish/Bearish Kicker (Kicking — eines der stärksten Umkehrsignale überhaupt), Stick Sandwich |
| Gap-Fortsetzung | Rising/Falling Window, Upside/Downside Tasuki Gap, Mat Hold (bullische Fortsetzung) |
| Schwache bärische Fortsetzung | On-Neck, In-Neck, Thrusting Line |
| Verstärkte Unentschlossenheit | Long-Legged Doji, High Wave |

Damit deckt die Liste zusammen ~35 der bekanntesten und meistgenutzten Candlestick-Muster ab —
bullisch und bärisch, gleichermaßen für Krypto und Aktien anwendbar.

**Verlässlichkeits-Hinweis:** Einzelne Muster sind in der TA-Literatur unterschiedlich stark belastbar
(z. B. gelten Engulfing/Kicker/Morning-Evening-Star als robuster als ein einzelner Doji). Grundsatz für
die Umsetzung: Candlestick-Signale nie allein verwenden — die genaue Gewichtung über Trendkontext und
Support/Resistance regelt Komponente 7 (Price Action). Keine erfundenen Trefferquoten, sondern ein
zusätzlicher Baustein neben der eigentlichen Signal-Strategie (siehe ToDo unten).

### 7. Price Action & Trendkontext
Der Kursrichtungswechsel (Trendwechsel) ist essentiell und wird nicht allein am Candlestick-Muster
festgemacht, sondern erst durch das Zusammenspiel mit dem übergeordneten Trend und wichtigen
Kurszonen bestätigt:
- **Trendfilter**: gleitende Durchschnitte (z. B. EMA50/EMA200) bestimmen die übergeordnete Richtung.
- **Support/Resistance**: Swing-High/-Low-Erkennung bzw. Pivot-Points markieren relevante Kurszonen.
- **Kernregel**: ein Candlestick-Signal aus Komponente 6 gilt nur dann als belastbar, wenn es entweder
  mit dem übergeordneten Trend läuft (Fortsetzung) oder eine Umkehr direkt an einer
  Support/Resistance-Zone markiert (Trendwechsel). Ohne diesen Kontext wird das Muster nur als
  "schwaches" Signal geführt, nicht ignoriert, aber niedriger gewichtet.
- Läuft als weiterer Schritt in derselben n8n-Pipeline, Ergebnis (Trendrichtung, aktive S/R-Zonen,
  Gewichtung des Candlestick-Signals) fließt mit in die `signals`-Tabelle.

### 8. Handelszeiten & Marktstärke
- **Aktienbörsen (Kernzeiten):** NYSE/NASDAQ 09:30–16:00 ET (≈14:30–21:00 UTC, verschiebt sich mit der
  US-Sommerzeit), Wiener Börse & XETRA/Frankfurt 09:00–17:30 MEZ, London Stock Exchange 08:00–16:30
  GMT/BST, Tokyo Stock Exchange 09:00–15:00 JST (Mittagspause 11:30–12:30), Hongkong 09:30–16:00 HKT.
- **Stärkste Phasen**: die ersten und letzten 30–60 Minuten einer Handelssitzung — dort ist Volumen und
  Volatilität am höchsten (typische "U-förmige" Intraday-Kurve). Die Überlappung London/New York
  (≈14:30–17:30 MEZ) ist weltweit das liquideste Zeitfenster für Aktien.
- **Schwächste Phase**: die Mittagspause/"Lunch Lull" (≈18:00–19:30 MEZ, US-Mittagszeit) — spürbar
  niedrigeres Volumen, oft seitwärts.
- **Krypto (24/7)**: kein offizieller Handelsschluss, aber die Aktivität folgt trotzdem den globalen
  Sessions — am aktivsten bei US/EU-Überlappung (≈14:00–22:00 UTC), ruhiger in der späten US-Nacht/frühen
  Asien-Zeit. Wochenenden haben ein dünneres Orderbuch → relativ zur Liquidität größere Kursausschläge
  möglich, wichtig für Komponente 9 (Risikomanagement).
- **Integration**: n8n reichert jeden Datenpunkt mit einem Zeitfenster-Kontext an (welche Börse gerade
  offen ist, ob eine Session-Überlappung aktiv ist) — fließt in die Signal-Gewichtung und ins
  Risikomanagement ein (z. B. kleinere Positionsgrößen in ruhigen Randzeiten).

### 9. Risikomanagement
Sitzt als Regel-Schicht zwischen Hermes-Entscheidung und tatsächlicher Aktion, bevor irgendetwas mit
echtem Geld ausgelöst wird:
- **Positionsgrößen-Regel**: max. X % des Kapitals pro Trade riskieren (konkreter Wert wird mit Marko
  zuhause festgelegt).
- **Stop-Loss/Take-Profit**: gekoppelt an ATR (Average True Range) oder das letzte Swing-High/-Low statt
  fixer Prozentwerte.
- **Mindest-Chance-Risiko-Verhältnis** (z. B. 1:2) — ein Signal wird nur zum Trade-Kandidaten, wenn das
  Verhältnis erreicht wird.
- **Tagesverlust-Circuit-Breaker**: Handel pausiert automatisch, sobald eine definierte Verlustschwelle
  am Tag erreicht ist.
- **Cooldown** nach mehreren Fehlsignalen/-trades hintereinander.
- **Pflicht-Bestätigung**: ab einer bestimmten Positionsgröße muss Marko manuell bestätigen, bevor
  Hermes "scharf" handelt.

## Ideen & Vorschläge

- **Sentiment-Vorverarbeitung**: News/Social-Daten vor Hermes durch ein lokales Ollama-Modell
  (mistral/llama3.2) laufen lassen, um Hermes nur mit verdichteten Signalen statt Rohtext zu füttern.
- **Caching/Rate-Limiting im OpenClaw-Agent**: Redis (bereits vorhanden) nutzen, um wiederholtes Scrapen
  derselben Quelle zu vermeiden und Bans zu verhindern.
- **Infra-Monitoring zusätzlich zum Signal-Dashboard**: neue Coworker-CTs auch in Prometheus (.109)
  aufnehmen — Job-Erfolgsraten, Scrape-Fehler, Hermes-Trigger-Häufigkeit (separates Grafana-Dashboard,
  nicht vermischt mit dem Kauf-/Verkaufssignal-Dashboard aus Komponente 5).
- **Alerting**: sobald der Telegram-Bot (offener Punkt aus dem Homelab-Backlog) aktiv ist, Trading-Signale
  darüber ausspielen statt nur ins Dashboard zu schreiben.
- **Watchlist-Kopplung**: bestehendes Config-Center (:8090) Watchlist add/remove pro Thema nutzen, damit
  OpenClaw nur beobachtete Coins/Ticker crawlt statt alles.
- **Langzeitgedächtnis**: jede finale Hermes-Entscheidung + Begründung zusätzlich als Kurzfassung in
  Qdrant `ai_memory` ablegen (`/webhook/qdrant-store`) — damit spätere Auswertung/Reflexion möglich ist.

## ToDo (zuhause)
- [ ] **Bestandsaufnahme zuerst**: prüfen, welche der oben beschriebenen Container/Dienste (Coworker,
  OpenClaw, DBs) bereits laufen (`pct list` auf allen 3 Nodes) — nur fehlende Teile neu aufsetzen, keine
  Duplikate anlegen.
- [ ] **Signal-Strategie**: Markos bestehende Kauf-/Verkaufs-Strategie/Indikatoren dokumentieren und in
  die `signals`-Tabelle/n8n-Berechnung übernehmen, statt eine neue zu erfinden.
- [ ] **Candlestick-Muster einbauen**: TA-Lib in die n8n-Pipeline (oder Python-Sidecar) integrieren, die
  ~35 Muster aus Komponente 6 auf OHLC-Daten laufen lassen, Ergebnis in `signals`-Tabelle + Grafana
  aufnehmen; mit der Signal-Strategie kombinieren statt isoliert zu verwenden.
- [ ] **Price Action einbauen**: Trendfilter (EMA50/EMA200) und Support/Resistance-Erkennung
  (Swing-High/-Low bzw. Pivot-Points) umsetzen und zur Gewichtung der Candlestick-Signale nutzen
  (Komponente 7).
- [ ] **Handelszeiten-Kontext einbauen**: Session-/Öffnungszeiten-Logik in n8n ergänzen (Komponente 8),
  inkl. Anbindung an die Signal-Gewichtung und ans Risikomanagement.
- [ ] **Risikomanagement konfigurieren**: konkrete Werte für Positionsgröße, Stop-Loss/Take-Profit-ATR,
  Mindest-Chance-Risiko-Verhältnis und Tagesverlust-Schwelle mit Marko festlegen (Komponente 9).
- [ ] **OpenClaw-Einbindung**: bestehenden OpenClaw-Agent in die zwei Coworker-CTs einbinden bzw.
  vorhandene Einbindung prüfen/übernehmen.
- [ ] **TradingView-Zugriff festlegen**: Scraping mit Rate-Limiting vs. Pine-Script-Alert-Webhooks — anhand
  der bestehenden OpenClaw-Fähigkeiten entscheiden.
- [ ] **CT-IDs, IP-Adressen, Node-Zuweisung** der zwei Coworker-Container festhalten (neu oder bereits
  vorhanden).
- [ ] Fehlende Bausteine aus diesem Dokument (Hermes-Tools, Postgres-DBs, n8n-Workflows,
  Grafana-Signal-Dashboard) ergänzen — nur was laut Bestandsaufnahme wirklich fehlt.
