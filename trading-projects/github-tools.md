# GitHub-Recherche: Tools & Bibliotheken für die Trading-Coworker

Kuratierte, nach Sternen sortierte Auswahl echter GitHub-Repos (Stand 2026-08-15, per GitHub-Suche
ermittelt — keine erfundenen Zahlen) zu den sechs von Marko angefragten Themenfeldern, jeweils mit Bezug
zur Architektur in [architektur.md](./architektur.md). Dient als Nachschlagewerk für die Umsetzung
zuhause — nichts davon ist installiert, alles sind Vorschläge zur Wiederverwendung statt Neuentwicklung.

## Überraschungsfund: OpenClaw + Trading-Skills-Ökosystem

Bei der Suche aufgefallen: **OpenClaw ist ein echtes, extrem populäres Open-Source-Projekt**
([`openclaw/openclaw`](https://github.com/openclaw/openclaw), **386.383★**, "Your own personal AI
assistant. Any OS. Any Platform.") — das erklärt, was im Homelab schon als Basis läuft. Dazu existiert
ein ganzes Skills-Ökosystem, darunter bereits fertige Trading-Skills:

| Repo | Sterne | Was es liefert |
|---|---|---|
| [`atilaahmettaner/tradingview-mcp`](https://github.com/atilaahmettaner/tradingview-mcp) | 3.970★ | TradingView-MCP-Server: Echtzeitdaten, technische Analyse, Screener, Backtesting für Aktien/Krypto/Forex/Futures — **löst die offene Frage "TradingView-Zugriff" konkret**, statt selbst zu scrapen |
| [`MobiusQuant/OpenMobius-skill`](https://github.com/MobiusQuant/OpenMobius-skill) | 641★ | ICT/Smart-Money-Concepts-Handelswissen als Skill — explizit für "Claude Code / Codex / OpenClaw / **Hermes**" gebaut, passt direkt zur bestehenden Hermes-Anbindung |
| [`aicoincom/coinos-skills`](https://github.com/aicoincom/coinos-skills) | 52★ | Krypto-Kurse, K-Lines, Funding-Rates, Whale-Tracking, CCXT- und Freqtrade-Anbindung als fertige OpenClaw-Skill |
| [`DaviddTech/ai-trading-agent`](https://github.com/DaviddTech/ai-trading-agent) | 49★ | Pine-Script/TradingView-MCP-Skill, Backtesting, Parameter-Tuning |

**Empfehlung:** vor dem Selbstbauen zuhause zuerst prüfen, ob diese Skills direkt in die bestehende
OpenClaw-Instanz eingebunden werden können — spart Entwicklungszeit gegenüber Neubau.

## 1. Trading (Plattformen/Engines)

| Repo | Sterne | Fokus |
|---|---|---|
| [`freqtrade/freqtrade`](https://github.com/freqtrade/freqtrade) | 53.308★ | Freier Krypto-Trading-Bot (Python) |
| [`ccxt/ccxt`](https://github.com/ccxt/ccxt) | 43.636★ | Einheitliche Trading-API für 100+ Krypto-Börsen |
| [`mementum/backtrader`](https://github.com/mementum/backtrader) | 22.855★ | Python-Backtesting-Bibliothek |
| [`QuantConnect/Lean`](https://github.com/QuantConnect/Lean) | 21.227★ | Multi-Asset Algo-Trading-Engine (Python/C#) |
| [`quantopian/zipline`](https://github.com/quantopian/zipline) | 20.040★ | Pythonic Algorithmic Trading Library |
| [`hummingbot/hummingbot`](https://github.com/hummingbot/hummingbot) | 19.467★ | High-Frequency-Krypto-Trading-Bots |
| [`StockSharp/StockSharp`](https://github.com/StockSharp/StockSharp) | 10.566★ | Algo-Trading-Plattform: Aktien, Forex, Krypto |

## 2. Algorithmen/Algorithmus (Strategien, ML-getrieben)

| Repo | Sterne | Fokus |
|---|---|---|
| [`stefan-jansen/machine-learning-for-trading`](https://github.com/stefan-jansen/machine-learning-for-trading) | 20.458★ | ML für Trading, Datenbeschaffung bis Live-Ausführung |
| [`kernc/backtesting.py`](https://github.com/kernc/backtesting.py) | 8.852★ | Backtesting von Trading-Strategien in Python |
| [`polakowo/vectorbt`](https://github.com/polakowo/vectorbt) | 8.687★ | Sehr schnelles Backtesting (Vektorisierung) |
| [`jesse-ai/jesse`](https://github.com/jesse-ai/jesse) | 8.323★ | Krypto-Trading-Bot-Framework |
| [`bukosabino/ta`](https://github.com/bukosabino/ta) | 5.141★ | Technische-Analyse-Indikatoren (Pandas/Numpy) |

## 3. Berechnung (numerische Methoden / Quant-Finance-Toolkits)

| Repo | Sterne | Fokus |
|---|---|---|
| [`wilsonfreitas/awesome-quant`](https://github.com/wilsonfreitas/awesome-quant) | 28.811★ | Kuratierte Meta-Liste aller Quant-Finance-Tools |
| [`goldmansachs/gs-quant`](https://github.com/goldmansachs/gs-quant) | 12.009★ | Echtes Goldman-Sachs-Open-Source-Toolkit |
| [`cantaro86/Financial-Models-Numerical-Methods`](https://github.com/cantaro86/Financial-Models-Numerical-Methods) | 7.346★ | Options-Pricing, PDEs, Monte Carlo (deckt auch Punkt 4 mit ab) |
| [`google/tf-quant-finance`](https://github.com/google/tf-quant-finance) | 5.473★ | TensorFlow-Bibliothek für Quant-Finance |

## 4. Wahrscheinlichkeitsrechnung (Stochastik in der Finanzmathematik)

| Repo | Sterne | Fokus |
|---|---|---|
| [`cantaro86/Financial-Models-Numerical-Methods`](https://github.com/cantaro86/Financial-Models-Numerical-Methods) | 7.346★ | Brownian Motion, Jump-Diffusion/Merton, Stochastic Differential Equations, Kalman-Filter, Lévy-Prozesse — **bester Treffer, direkt anwendbar** |
| [`google/tf-quant-finance`](https://github.com/google/tf-quant-finance) | 5.473★ | Numerische Optimierung/Integration |

## 5. Risiko-Rechnung

| Repo | Sterne | Fokus |
|---|---|---|
| [`PyPortfolio/PyPortfolioOpt`](https://github.com/PyPortfolio/PyPortfolioOpt) | 5.964★ | Efficient Frontier, Black-Litterman |
| [`dcajasn/Riskfolio-Lib`](https://github.com/dcajasn/Riskfolio-Lib) | 4.442★ | CVaR, Risk Parity, Drawdown-Modelle |
| [`The-Swarm-Corporation/AutoHedge`](https://github.com/The-Swarm-Corporation/AutoHedge) | 4.177★ | KI-Agenten für Risikomanagement/Trade-Execution — architektonisch ähnlich zu unserem Hermes-Ansatz |
| [`skfolio/skfolio`](https://github.com/skfolio/skfolio) | 2.116★ | Portfolio-Optimierung auf scikit-learn-Basis |
| [`fortitudo-tech/fortitudo.tech`](https://github.com/fortitudo-tech/fortitudo.tech) | 304★ | CVaR-Portfolio-Optimierung + Stress-Testing |

## 6. Hochschul-Mathematik (akademischer Bezug)

| Repo | Sterne | Fokus |
|---|---|---|
| [`cantaro86/Financial-Models-Numerical-Methods`](https://github.com/cantaro86/Financial-Models-Numerical-Methods) | 7.346★ | Heston-Modell, Fourier-Inversion — Uni-Niveau |
| [`wilsonfreitas/awesome-quant`](https://github.com/wilsonfreitas/awesome-quant) | 28.811★ | Verlinkt u. a. akademische Ressourcen |
| [`Louisli0515/Financial-Markets-Yale-University-Coursea-Note`](https://github.com/Louisli0515/Financial-Markets-Yale-University-Coursea-Note) | 84★ | Notizen zu Robert Shillers bekanntem Yale-Kurs "Financial Markets" |
| [`spedygiorgio/lifecontingencies`](https://github.com/spedygiorgio/lifecontingencies) | 73★ | Aktuarmathematik (R) |

## Candlestick/Price-Action-Ergänzung (verifiziert, siehe architektur.md Komponente 6+7)

| Repo | Sterne | Fokus |
|---|---|---|
| [`TA-Lib/ta-lib-python`](https://github.com/TA-Lib/ta-lib-python) | 12.186★ | Offizieller Python-Wrapper für TA-Lib — Basis für die Candlestick-Erkennung (Komponente 6) |
| [`TA-Lib/ta-lib`](https://github.com/TA-Lib/ta-lib) | 1.656★ | Offizieller TA-Lib Core |
| [`joshyattridge/smart-money-concepts`](https://github.com/joshyattridge/smart-money-concepts) | 1.936★ | Smart-Money-Concepts/ICT: Order Blocks, Fair Value Gaps, Liquidity, BOS/CHoCH — präzisere Umsetzung von Komponente 7 (Price Action) als reine EMA-Kreuzung |
| [`cm45t3r/candlestick`](https://github.com/cm45t3r/candlestick) | 504★ | JavaScript-Alternative, reine Candlestick-Erkennung |

## Hinweis

Alle genannten Repos sind Bibliotheken/Skills, keine neue Infrastruktur — sie werden bei Bedarf in die
bereits geplante n8n-Pipeline bzw. den OpenClaw-Agenten eingebunden (siehe
[architektur.md](./architektur.md)). Vor dem Einbau jeweils Lizenz, Aktivität (letzter Commit) und
Kompatibilität mit dem bestehenden Homelab-Stack prüfen.
