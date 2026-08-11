# Architektur: Coworker-Infrastruktur für Krypto- & Aktien-Trading

**Status:** 📐 Architektur-Entwurf — Umsetzung erfolgt zuhause mit Homelab-Zugriff (kein Netzwerkzugriff
auf die Proxmox-Nodes von dieser Remote-Umgebung aus).

## Überblick

Pro Thema (Krypto, Aktien) die gleiche Struktur, komplett getrennt voneinander:

```
OpenClaw-Agent (24/7, im jeweiligen Coworker-CT)
        │  scraped/rohe Marktdaten, News, Sentiment
        ▼
n8n-Workflow "Job Intake & Processing" (bestehende n8n-Instanz, CT200)
        │  Normalisierung, Deduplizierung, Enrichment, Indikator-/Signal-Berechnung
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

## Ideen & Vorschläge

- **Sentiment-Vorverarbeitung**: News/Social-Daten vor Hermes durch ein lokales Ollama-Modell
  (mistral/llama3.2) laufen lassen, um Hermes nur mit verdichteten Signalen statt Rohtext zu füttern.
- **Risk-/Safety-Schicht vor jeder Aktion**: ein einfacher Circuit-Breaker/Regelsatz zwischen
  Hermes-Entscheidung und tatsächlicher Aktion (v.a. bei echtem Geld) — z. B. Max-Positionsgrößen,
  Cooldown nach Fehlentscheidungen, Pflicht-Bestätigung ab bestimmter Schwelle.
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

## Offene Punkte (werden zuhause mit Homelab-Zugriff geklärt)
- **Signal-Strategie**: Marko hat bereits eine Kauf-/Verkaufs-Strategie/Indikatoren im Kopf — wird zuhause
  ergänzt und in die `signals`-Tabelle/n8n-Berechnung übernommen, statt hier eine neue zu erfinden.
- **OpenClaw-Konfiguration**: bereits im Homelab vorhanden, genaue Einbindung in die zwei neuen
  Coworker-CTs folgt zuhause.
- **TradingView-Zugriff**: Scraping mit Rate-Limiting vs. Pine-Script-Alert-Webhooks — Entscheidung
  zuhause anhand der bestehenden OpenClaw-Fähigkeiten.
- **CT-IDs, IP-Adressen, Node-Zuweisung** der zwei neuen Coworker-Container.
