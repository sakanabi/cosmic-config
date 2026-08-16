# Architektur: Coworker-Infrastruktur für Krypto- & Aktien-Trading

**Status:** ✅ Architektur-Planung abgeschlossen — Umsetzung offen, erfolgt zuhause mit Homelab-Zugriff
(kein Netzwerkzugriff auf die Proxmox-Nodes von dieser Remote-Umgebung aus).

> **Hinweis:** Ein Teil der hier beschriebenen Container/Dienste existiert im Homelab bereits. Bevor
> irgendetwas neu aufgesetzt wird, zuerst die Bestandsaufnahme aus der ToDo-Liste unten durchführen.

## Grundsatzentscheidungen (Stand heute, mit Marko geklärt)

- **Automatisierungsgrad: Nur Empfehlungen.** Das System liefert Signale + Begründung im Dashboard,
  Marko trifft die finale Kauf-/Verkaufsentscheidung und handelt manuell. **Kein automatischer
  Trade-Execution**, damit auch **keine Broker-/Exchange-API-Anbindung nötig** — Hermes "entscheidet"
  im Sinne von "empfiehlt", nicht im Sinne von "löst eine Order aus".
- **Start mit Paper-Trading/Simulation.** Die Signal-Strategie (Candlestick + Price Action + klassische
  Indikatoren, Komponenten 6–8) wird zunächst simuliert anhand der `signals`-Tabelle validiert
  (Backtesting), bevor mit echtem Geld gehandelt wird.
- **Baureihenfolge: Krypto- und Aktien-Coworker parallel**, da die gemeinsame Infrastruktur (n8n, Hermes,
  Grafana, Postgres) ohnehin geteilt aufgebaut wird.
- **Risiko pro Trade: 1 % des Kapitals** (konservativ) — konkreter Wert für die Positionsgrößen-Regel in
  Komponente 9.
- **Kapitalrahmen** (korrigiert — zwei getrennte Kapitaltöpfe statt einem gemeinsamen):
  - **Krypto (OKX)**: Start mit **300 €**, monatlich **+50 €** zusätzlich.
  - **Aktien**: Start mit **150 €**, monatlich **+150 €** zusätzlich.
  - Bei 1 % Risiko entspricht das zu Beginn ca. **3 € Risiko pro Krypto-Trade** bzw. ca.
    **1,50 € Risiko pro Aktien-Trade** — sehr kleine, aber realistische Positionsgrößen, die im
    Paper-Trading erst validiert werden, bevor größere Beträge bewegt werden.
  - **Aktien-Besonderheit**: die tatsächliche Positionsgröße hängt zusätzlich vom Aktienkurs ab — es
    können nur ganze Aktien gekauft werden (außer Flatex unterstützt Bruchstücke), das 1 %-Risikoziel
    ist daher eine Obergrenze/Richtwert, kein exakt erreichbarer Wert bei jedem Titel.
  - _(Anmerkung: "300%" wurde wie beim vorherigen "150%" als Tippfehler für 300 € gelesen — bitte
    korrigieren, falls tatsächlich etwas anderes gemeint war.)_
- **Cooldown**: max. **1 Arbeitswoche (5 Handelstage)** nach Verlustserie/Circuit-Breaker-Trigger — für
  beide Themen gleich (Komponente 9).
- **Mindest-Chance-Risiko-Verhältnis**: **1:2** für beide Themen — ein Signal wird erst zum
  Trade-Kandidaten, wenn das potenzielle Gewinnziel mindestens doppelt so weit entfernt ist wie der
  Stop-Loss. Bewährter Standardwert, der bei den kleinen Positionsgrößen hier zusätzlich hilft, Gebühren/
  Spread relativ zur Positionsgröße zu verkraften.
- **Tagesverlust-Schwelle**: **3 %** des jeweiligen Kapitaltopfs (Krypto- und Aktien-Topf getrennt
  betrachtet) — bei Erreichen pausiert der Handel im jeweiligen Thema für den Rest des Tages. Entspricht
  bei 1 % Risiko pro Trade etwa 3 aufeinanderfolgenden Verlust-Trades als Tagesgrenze.
- **Stop-Loss-ATR-Multiplikator** (Startwert, im Paper-Trading zu validieren): ATR(14) als Basis,
  **Aktien 2,5–3× ATR(14)** (mehr Spielraum für die 52-Wochen-Tief-Strategie, siehe unten),
  **Krypto 1,5–2× ATR(14)** (engerer Stop passend zum kurzen Zeithorizont — die höhere Volatilität ist
  in ATR selbst schon enthalten, der Multiplikator muss dafür nicht zusätzlich steigen).
- **Aktien-Universum**: kein festes kleines Watchlist, sondern Screening über das **gesamte über Flatex
  handelbare Aktienuniversum plus Forex über OKX** — siehe Komponente 1 und 7.
- **Krypto-Universum**: feste Watchlist von ca. **20 Coins** (Liste folgt).

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

**Universum pro Thema (grundlegend unterschiedlich):**
- **Aktien-Coworker**: kein festes Watchlist, sondern Screening über das **gesamte über Flatex
  handelbare Aktienuniversum plus Forex über OKX** — der Aktien-Coworker beobachtet also global, nicht
  nur einzelne Symbole. Siehe 52-Wochen-Tief-Screening in Komponente 7.
- **Krypto-Coworker**: feste, kuratierte Watchlist von ca. **20 Coins** (Liste folgt von Marko) —
  bewusst eng, da bei Krypto Geschwindigkeit ("schnelles Geld") statt Breite im Vordergrund steht.

**OpenClaw-Skills-Ökosystem (GitHub-Recherche):** OpenClaw (`openclaw/openclaw`, 386.383★) hat ein
eigenes Skills-Ökosystem, in dem bereits fertige Trading-Skills existieren — vor Neubau prüfen, ob diese
direkt einbindbar sind: `atilaahmettaner/tradingview-mcp` (3.970★, TradingView-MCP-Server mit
Echtzeitdaten/TA/Screener/Backtesting — löst die TradingView-Zugriffsfrage direkt),
`MobiusQuant/OpenMobius-skill` (641★, ICT/Smart-Money-Concepts-Wissen, explizit für OpenClaw **und**
Hermes gebaut), `aicoincom/coinos-skills` (52★, Krypto-Kurse/CCXT/Freqtrade-Anbindung). Details siehe
[github-tools.md](./github-tools.md).

### 2. Hermes-Agent für die erweiterte Entscheidung
Empfehlung: **keinen dritten/vierten Hermes deployen**, sondern den bestehenden Hermes-Agent
(.26:5058, POST `/agent {task}`) um zwei neue Tools/Endpoints erweitern (`krypto_decision`,
`aktien_decision`). **Scope-Klarstellung:** Hermes' "Entscheidung" ist eine Empfehlung mit Begründung im
Dashboard — kein automatischer Trade, keine Broker-Anbindung (siehe Grundsatzentscheidungen oben).
Begründung:
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

**Konkretes Panel-Layout** (Anordnung/Optik nach einer von Marko gezeigten Dashboard-Vorlage —
Kachel-Raster mit Statusleiste + KPI-Karten + breiten Analyse-Panels — befüllt mit unseren echten,
berechneten Indikatoren statt der dort gezeigten Platzhalter-/Fantasiezahlen):
- **Statusleiste (oben)**: Symbol/Thema (z. B. "BTC/USDT" oder "AAPL"), `LIVE`-Badge, Zeitstempel letzte
  Aktualisierung, Badge für den aktuell aktiven Handelszeit-Rang aus Komponente 8 (1–5).
- **KPI-Kartenreihe (Row 1)**: zwei Karten nebeneinander —
  - Links, groß: **"Aktuelle Empfehlung"** — `KAUF`/`VERKAUF`/`HALTEN` als Headline, darunter
    Sub-Metriken als kleine Chips: Trend-Richtung (EMA9/50/200-Ausrichtung), RSI-Wert, MACD-Status
    (bullisch/bärisch), aktive Candlestick-/SMC-Muster-Anzahl.
  - Rechts, kleiner: **"Letztes starkes Signal"** — Name des zuletzt ausgelösten hochgewichteten Musters
    (z. B. "Bullish Engulfing an Support-Zone") mit Mini-Sparkline des Kursverlaufs um den Zeitpunkt.
- **Analyse-Panel (Row 2, breit)**: links eine schmale Statsliste (aktive S/R-Zonen, EMA-Trio-Status,
  Bollinger-Band-Breite/Squeeze-Status), rechts groß der **Candlestick-Chart** mit allen Signal-Markern
  (Candlestick-Muster, Smart-Money-Concepts-Zonen/Order-Blocks, EMA9/50/200-Linien, Bollinger-Bänder).
- **Analyse-Panel (Row 3, breit)**: links eine Statsliste zum Risikomanagement (aktuelle Positionsgröße,
  Stop-Loss-Abstand, Chance-Risiko-Verhältnis des offenen/letzten Signals — Komponente 9), rechts ein
  **Signal-Konfluenz-Verlauf**: Zeitreihe, wie viele Indikatoren (Candlestick + Trend + RSI/MACD/BB +
  Handelszeit-Gewichtung) gleichzeitig übereinstimmten — je mehr Übereinstimmung, desto stärker das
  Signal. Ersetzt die "Probability Lattice"/Glücksspiel-Visualisierung der Vorlage durch eine
  nachvollziehbare, aus echten Werten berechnete Darstellung.
- **Tabelle (unten)**: letzte Signale/Trades mit Begründung (welche Muster + Indikatoren zum Signal
  geführt haben) — Nachvollziehbarkeit statt Blackbox.
- **Wichtig:** Übernommen wird nur die Anordnung/Optik (Statusleiste, Kachel-Raster, breite
  Analyse-Panels). Die in der Vorlage gezeigten Zahlen ($401.786 PnL, x52-Multiplikator, 71 % Winrate)
  sind nicht real und werden nicht übernommen — das Dashboard zeigt ausschließlich Werte, die aus der
  eigenen n8n-Berechnung stammen.

### 6. Candlestick-Muster als Indikator
- **Kein Neubau der Erkennung** — dafür **TA-Lib** nutzen (Standard-Bibliothek für technische Analyse,
  konkret [`TA-Lib/ta-lib-python`](https://github.com/TA-Lib/ta-lib-python), **12.186★**, offizieller
  Python-Wrapper), die ~60 fertige, geprüfte `CDL*`-Erkennungsfunktionen für praktisch alle gängigen
  Candlestick-Muster mitbringt.
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
| Starke Umkehrsignale | Bullish/Bearish Belt Hold, Bullish/Bearish Kicker (Kicking — eines der stärksten Umkehrsignale überhaupt), Stick Sandwich, Bullish/Bearish Counter Attack (Counterattack Lines — gleichschließende Kerzen gegen den vorherigen Trend) |
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
- **Trendfilter**: gleitende Durchschnitte bestimmen die übergeordnete Richtung.
- **Support/Resistance**: Swing-High/-Low-Erkennung bzw. Pivot-Points markieren relevante Kurszonen.
- **Chart-Muster als zusätzliche S/R-Bestätigung**: Doppel-Top/Doppel-Boden (zwei etwa gleich hohe
  Hochs bzw. gleich tiefe Tiefs) gelten als klassische, mehrperiodige Umkehrmuster auf Chart-Ebene —
  ergänzen die einzelkerzenbasierten Muster aus Komponente 6 um eine Bestätigung über mehrere
  Swing-Punkte hinweg.

**Klassische Indikatoren (konkrete Umsetzung des Trendfilters):**
- **EMA-Trio (9/50/200)**: EMA9 kurzfristig/schnell (Timing für Einstiege), EMA50 mittelfristiger Trend,
  EMA200 langfristiger Haupttrend. Kreuzungen (z. B. EMA50/EMA200 = "Golden Cross"/"Death Cross") als
  zusätzliches Trendsignal.
- **RSI (14 Perioden)**: Momentum-Oszillator; überkauft >70, überverkauft <30; Divergenzen zwischen
  Kursverlauf und RSI als eigenständiges Umkehrsignal.
- **MACD (12/26/9)**: Trendfolge-Momentum-Indikator; MACD-Linie kreuzt Signal-Linie = möglicher
  Momentum-Wechsel, Histogram zeigt die Stärke.
- **Bollinger Bands (20-Perioden-SMA ± 2 Standardabweichungen)**: Volatilitätsbänder; Kurs am
  oberen/unteren Band = mögliche Überdehnung, "Band-Squeeze" (enge Bänder) = bevorstehender Ausbruch.
- Alle vier sind bereits Standardfunktionen in `TA-Lib/ta-lib-python` (siehe Komponente 6) — keine
  zusätzliche Bibliothek nötig, nur weitere `TA-Lib`-Aufrufe (`RSI`, `MACD`, `BBANDS`, `EMA`) im selben
  n8n-Schritt. Fließen als weitere Spalten in dieselbe `signals`-Tabelle und dieselbe Gewichtungslogik
  wie Candlestick-Muster und Smart-Money-Concepts — kein separater Signalpfad.
- **Kernregel**: ein Candlestick-Signal aus Komponente 6 gilt nur dann als belastbar, wenn es entweder
  mit dem übergeordneten Trend läuft (Fortsetzung) oder eine Umkehr direkt an einer
  Support/Resistance-Zone markiert (Trendwechsel). Ohne diesen Kontext wird das Muster nur als
  "schwaches" Signal geführt, nicht ignoriert, aber niedriger gewichtet.
- Läuft als weiterer Schritt in derselben n8n-Pipeline, Ergebnis (Trendrichtung, aktive S/R-Zonen,
  Gewichtung des Candlestick-Signals) fließt mit in die `signals`-Tabelle.
- **Konkrete Bibliothek (GitHub-Recherche):** [`joshyattridge/smart-money-concepts`](https://github.com/joshyattridge/smart-money-concepts)
  (1.936★, Python) implementiert Smart-Money-Concepts/ICT — Order Blocks, Fair Value Gaps, Liquidity,
  Break-of-Structure/Change-of-Character — eine deutlich präzisere, fertige Umsetzung des
  "Kursrichtungswechsel"-Konzepts als eine reine EMA-Kreuzung. Kandidat, um Trendfilter und
  Support/Resistance oben direkt abzudecken statt beides von Grund auf neu zu bauen.

**Aktien-spezifisches Screening: 52-Wochen-Tief-Strategie**
- Zusätzlicher Screening-Schritt **vor** der eigentlichen Signal-Berechnung, nur für den
  Aktien-Coworker: Aktien aus dem gesamten über Flatex handelbaren Universum, die nahe ihrem
  **52-Wochen-Tief** handeln, werden als Kauf-Kandidat markiert (klassische Value-/Contrarian-Screening-
  Methode) — dieselbe Logik gilt für Forex über OKX.
- Läuft als Filter-Stufe in derselben n8n-Pipeline: erst grobes Screening über das ganze Universum
  (52-Wochen-Tief-Nähe), dann erst Candlestick-/Price-Action-/Indikator-Berechnung (Komponente 6+7) nur
  noch auf den gefilterten Kandidaten — sonst wäre die Datenmenge (gesamtes Flatex-Universum) für die
  volle Pipeline zu groß.
- Ergänzt, ersetzt aber nicht die Kernregel aus Komponente 7 (Trend + S/R + Candlestick) — das
  52-Wochen-Tief liefert die Kandidatenliste, die Kernregel entscheidet über die tatsächliche
  Gewichtung des Signals.

### 8. Handelszeiten & Marktstärke
- **Aktienbörsen (Kernzeiten):** NYSE/NASDAQ 09:30–16:00 ET (≈14:30–21:00 UTC, verschiebt sich mit der
  US-Sommerzeit), Wiener Börse & XETRA/Frankfurt 09:00–17:30 MEZ, London Stock Exchange 08:00–16:30
  GMT/BST, Tokyo Stock Exchange 09:00–15:00 JST (Mittagspause 11:30–12:30), Hongkong 09:30–16:00 HKT.

**Wichtigkeits-Ranking der Zeitfenster (fließt als Gewichtung in Signal-Berechnung & Risikomanagement ein):**

| Rang | Zeitfenster | Charakteristik |
|---|---|---|
| 1 — am stärksten | Overlap London + New York, ≈14:30–17:00 MEZ | Meist die stärkste Bewegung, hohes Volumen + Liquidität, Breakouts und Richtungsentscheidungen häufig |
| 2 — sehr wichtig | New York Open, ≈15:30–17:30 MEZ | US-Daten und News bewegen stark, hohe Volatilität möglich, Vorsicht bei impulsiven Kerzen |
| 3 — wichtig | London/Europa Open, ≈09:00–11:00 MEZ | Oft erster echter Schub des Tages, mehr Aktivität als in der Asien-Phase, gut für Struktur/Setups |
| 4 — eher ruhiger | Asien-Session, ≈01:00–08:00 MEZ | Oft ruhiger, aber nicht immer; gut zum Vorbereiten wichtiger Zonen, manchmal Start für spätere Trends |
| 5 — Vorsicht | Wochenende (Krypto 24/7) | Oft dünnere Liquidität, mehr Fakeouts und unruhige Moves, Risiko/Positionsgröße anpassen |

- **Schwächste Phase innerhalb des Handelstags**: die Mittagspause/"Lunch Lull" (≈18:00–19:30 MEZ,
  US-Mittagszeit) — spürbar niedrigeres Volumen, oft seitwärts.
- **Krypto (24/7)**: kein offizieller Handelsschluss, aber die Aktivität folgt trotzdem den globalen
  Sessions — am aktivsten bei US/EU-Überlappung (Rang 1 oben), ruhiger in der späten US-Nacht/frühen
  Asien-Zeit. Wochenenden haben ein dünneres Orderbuch → relativ zur Liquidität größere Kursausschläge
  möglich, wichtig für Komponente 9 (Risikomanagement).
- **Integration**: n8n reichert jeden Datenpunkt mit einem Zeitfenster-Kontext an (welcher Rang aus der
  Tabelle oben gerade aktiv ist) — fließt in die Signal-Gewichtung und ins Risikomanagement ein (z. B.
  kleinere Positionsgrößen in Rang-4/5-Zeiten).

### 9. Risikomanagement
Sitzt als Regel-Schicht zwischen Hermes-Entscheidung und tatsächlicher Aktion, bevor irgendetwas mit
echtem Geld ausgelöst wird:
- **Positionsgrößen-Regel**: max. **1 %** des Kapitals pro Trade riskieren (konservativ, mit Marko
  festgelegt). Kapitalrahmen siehe Grundsatzentscheidungen oben (300 € Start, +150 €/Monat für Aktien).
- **Stop-Loss/Take-Profit**: gekoppelt an ATR (Average True Range), **Aktien 2,5–3× ATR(14)**,
  **Krypto 1,5–2× ATR(14)** — siehe Grundsatzentscheidungen oben; Startwert, wird im Paper-Trading
  validiert/justiert.
- **Mindest-Chance-Risiko-Verhältnis: 1:2** — ein Signal wird nur zum Trade-Kandidaten, wenn das
  potenzielle Gewinnziel mindestens doppelt so weit entfernt ist wie der Stop-Loss (siehe
  Grundsatzentscheidungen oben).
- **Tagesverlust-Circuit-Breaker: 3 % des jeweiligen Kapitaltopfs** (Krypto- und Aktien-Topf getrennt) —
  Handel pausiert automatisch für den Rest des Tages, sobald diese Schwelle erreicht ist (siehe
  Grundsatzentscheidungen oben).
- **Cooldown**: max. **1 Arbeitswoche (5 Handelstage)** nach Verlustserie/Circuit-Breaker-Trigger — für
  beide Themen gleich.
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
  Duplikate anlegen. Krypto- und Aktien-Coworker werden **parallel** aufgebaut (siehe
  Grundsatzentscheidungen oben).
- [ ] **Paper-Trading/Backtesting einrichten**: Signal-Strategie zunächst simuliert gegen historische
  Daten in der `signals`-Tabelle validieren, bevor mit echtem Geld gehandelt wird — erst danach den
  "Echtgeld"-Schalter überhaupt in Betracht ziehen.
- [ ] **Signal-Strategie**: Markos bestehende Kauf-/Verkaufs-Strategie/Indikatoren dokumentieren und in
  die `signals`-Tabelle/n8n-Berechnung übernehmen, statt eine neue zu erfinden.
- [ ] **Candlestick-Muster einbauen**: TA-Lib in die n8n-Pipeline (oder Python-Sidecar) integrieren, die
  ~35 Muster aus Komponente 6 auf OHLC-Daten laufen lassen, Ergebnis in `signals`-Tabelle + Grafana
  aufnehmen; mit der Signal-Strategie kombinieren statt isoliert zu verwenden.
- [ ] **Price Action einbauen**: Trendfilter (EMA9/EMA50/EMA200, RSI, MACD, Bollinger Bands via TA-Lib)
  und Support/Resistance-Erkennung (Swing-High/-Low bzw. Pivot-Points) umsetzen und zur Gewichtung der
  Candlestick-Signale nutzen (Komponente 7).
- [ ] **Handelszeiten-Kontext einbauen**: Session-/Öffnungszeiten-Logik in n8n ergänzen (Komponente 8),
  inkl. Anbindung an die Signal-Gewichtung und ans Risikomanagement.
- [ ] **Risikomanagement konfigurieren**: Mindest-Chance-Risiko-Verhältnis und Tagesverlust-Schwelle mit
  Marko festlegen (Komponente 9) — Positionsgröße (1 %), ATR-Multiplikatoren und Cooldown sind bereits
  festgelegt (siehe Grundsatzentscheidungen).
- [ ] **52-Wochen-Tief-Screening bauen**: Filter-Stufe vor der eigentlichen Signal-Pipeline für das
  gesamte Flatex-Aktienuniversum + OKX-Forex (Komponente 7).
- [ ] **Krypto-Watchlist eintragen**: die ca. 20 Coins von Marko in Komponente 1 ergänzen, sobald die
  Liste vorliegt.
- [ ] **OpenClaw-Einbindung**: bestehenden OpenClaw-Agent in die zwei Coworker-CTs einbinden bzw.
  vorhandene Einbindung prüfen/übernehmen.
- [ ] **TradingView-Zugriff festlegen**: erst prüfen, ob `atilaahmettaner/tradingview-mcp` (3.970★) als
  fertiger MCP-Server passt, bevor Scraping mit Rate-Limiting oder Pine-Script-Alert-Webhooks gebaut
  werden (siehe [github-tools.md](./github-tools.md)).
- [ ] **CT-IDs, IP-Adressen, Node-Zuweisung** der zwei Coworker-Container festhalten (neu oder bereits
  vorhanden).
- [ ] Fehlende Bausteine aus diesem Dokument (Hermes-Tools, Postgres-DBs, n8n-Workflows,
  Grafana-Signal-Dashboard) ergänzen — nur was laut Bestandsaufnahme wirklich fehlt.
