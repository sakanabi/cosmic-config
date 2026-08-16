# Projekt: Aktien Trading

**Status:** 📝 Entwurf — noch nicht ins lokale Gedächtnis übertragen

## Beschreibung
_(wird ergänzt)_

## Ziele
- [ ]

## Watchlist / Ticker

**Status: erste Beobachtungsliste, keine Kaufempfehlung.** Gesammelt aus TikTok-Inhalten, die Marko
interessant fand — unterschiedlich seriöse Quellen (siehe Hinweis unten). Ersetzt **nicht** das
systematische 52-Wochen-Tief-Screening über das gesamte Flatex-Universum aus
[architektur.md](./architektur.md#7-price-action--trendkontext), sondern dient als erster
Beobachtungs-/Interessens-Kandidatenkreis, der zusätzlich beobachtet wird.

**Qualitäts-/Dividenden-Werte (breiter etablierte Listen):**
Microsoft, Visa, Mastercard, Broadcom, ASML, S&P Global, Moody's, KLA, MSCI, Eli Lilly, BlackRock,
Costco, Trane Technologies, Rollins, Cintas, Schneider Electric, Münchener Rück, Ares Management, Texas
Instruments, Booking Holdings, Johnson & Johnson, Veolia, Deere & Company, BHP, GE Vernova,
A.P. Møller-Mærsk, Realty Income, PepsiCo, Allianz, Unilever, TotalEnergies, Enbridge, Chevron, Nestlé,
Procter & Gamble

**Nahe 52-Wochen-Tief / Drawdown-Kandidaten (passt zur eigenen Screening-Strategie):**
Netflix (NFLX), Nike (NKE), Oracle (ORCL), Xiaomi (1810.HK), McDonald's (MCD), Intuit (INTU), Adobe
(ADBE), Salesforce (CRM), Palantir (PLTR), Alibaba (BABA), S&P Global (SPGI), Meta (META)

**Wachstum/Tech (Q1-2026-Käufe aus einem Investoren-Portfolio-Screenshot, unverifiziert):**
JPMorgan, Goldman Sachs, AMD, Intel, Dell

**Quantum-Computing-Themenkorb** (spekulativ, hohe Volatilität — eigenes Risiko-Profil nötig):
IONQ, QUBT, RGTI, INFQ, QNT, QBTS, sowie große Plattform-Werte (Microsoft, Google, Nvidia, Amazon, IBM,
Honeywell) sowie Zulieferer (Intel, Teradyne, Coherent, IPG Photonics, MKS Instruments)

**Sehr spekulativ / Kleinstwerte:** TransMedics (TMDX), DLocal (DLO), Harrow (HROW), Pagaya (PGY),
Applied Optoelectronics (AAOI), Blacksky, Recursion, Intuitive Machines, Ouster, Crispr

> **Hinweis zu den Quellen:** Zwei der Ursprungs-Posts sind mit Vorsicht zu genießen — ein Post mit
> Kurszielen wie "24 $ → 670 $" trägt selbst den Disclaimer "keine Haftung für die Richtigkeit der
> Daten", ein anderer ("Buy the Dip?") ist als bezahlte Werbung ("Ad") für ein Analyse-Tool markiert.
> Beide liefern nur die Firmennamen als Ausgangspunkt — keine der dort genannten Kursziele/Bewertungen
> wird übernommen. Vor jeder tatsächlichen Beobachtung sollten diese Titel durch das eigene
> 52-Wochen-Tief-Screening und die Indikatoren-Kombination (Komponente 6+7) laufen statt sich auf die
> TikTok-Einschätzung zu verlassen.

## Strategie
_(wird ergänzt)_

## Verknüpfte Services (Homelab)
- Market-Analyst (CT205, :5056)
- Trading-Dashboard (`trading.html`, CT120)

## Infrastruktur
Siehe [Architektur-Dokument](./architektur.md) für die vollständige Coworker-Infrastruktur
(gemeinsam für Krypto & Aktien). Themen-eigene Bausteine für Aktien:
- **Aktien-Coworker** (neuer LXC-Container) mit OpenClaw-Agent — Hauptquelle **TradingView**
  (Charts/Indikatoren, Watchlist-Symbole), ergänzt durch allgemeine Finanznews-Suche.
- **Datenbank:** `aktien_trading` (PostgreSQL, CT110).
- Gemeinsam genutzt: Hermes-Agent (`aktien_decision`-Tool), n8n-Workflows, Grafana-Signal-Dashboard.

## Nächste Schritte
- [ ] Inhalte vervollständigen
- [ ] Ins lokale Gedächtnis übertragen (Qdrant `ai_memory` via n8n `/webhook/qdrant-store`)
