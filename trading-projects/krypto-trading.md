# Projekt: Krypto Trading

**Status:** 📝 Entwurf — noch nicht ins lokale Gedächtnis übertragen

## Beschreibung
_(wird ergänzt)_

## Ziele
- [ ]

## Watchlist / Coins
- [ ]

## Strategie
_(wird ergänzt)_

## Verknüpfte Services (Homelab)
- Market-Analyst (CT205, :5056)
- Trading-Dashboard (`trading.html`, CT120)

## Infrastruktur
Siehe [Architektur-Dokument](./architektur.md) für die vollständige Coworker-Infrastruktur
(gemeinsam für Krypto & Aktien). Themen-eigene Bausteine für Krypto:
- **Krypto-Coworker** (neuer LXC-Container) mit OpenClaw-Agent — Hauptquelle **CoinMarketCap**,
  ergänzt durch TradingView (Charts/Indikatoren) und allgemeine Krypto-News-Suche.
- **Datenbank:** `krypto_trading` (PostgreSQL, CT110).
- Gemeinsam genutzt: Hermes-Agent (`krypto_decision`-Tool), n8n-Workflows, Grafana-Signal-Dashboard.

## Nächste Schritte
- [ ] Inhalte vervollständigen
- [ ] Ins lokale Gedächtnis übertragen (Qdrant `ai_memory` via n8n `/webhook/qdrant-store`)
