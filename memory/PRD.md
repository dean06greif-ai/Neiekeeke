# PRD – Neu-dBitch Daytrading-Plattform

## Original problem statement
Die Website (Daytrading-Plattform) funktioniert bereits produktiv und stabil. Alle Verbesserungen sollen sauber und modular integriert werden. Priorisiert wurden zuletzt Orderbuch-Proxies für Forex/Rohstoffe, dynamische Timeframes, ML-Risiko-Skalierung, News-Deduplikation, OpenRouter-Priorisierung, Fee-Guard V3, manuelle Lesson-Validierung, clearable Number-Inputs, Strategy-Toggle für externe Trades und Mobile-Overflow-Fix.

## Architecture
- Backend: FastAPI (Python 3.11), MongoDB (primär), Supabase (Vektor-Spiegel).
- Frontend: React (CRA), Tailwind-ähnliche eigene CSS.
- Trading: Bitunix API.
- KI-Provider (via ai_providers.py): OpenRouter (+ Backup-Keys), Gemini, Groq, Mistral, Cerebras.

## Implemented (kumulativ)
- Forex/Commodities Orderbook Proxies via CME Futures.
- Dynamische Timeframe-Anpassung (5m/15m bei niedriger Vola).
- ML-Gate Risiko-Halbierung bei AUC < 0.55.
- News-Depth Deduplikation (Token-Ersparnis).
- OpenRouter Prio & Fallback-Kette.
- Fee Guard V3 (CRV Relax > 3.0).
- Manuelle Validierung + Verwerfen von Lesson-Kandidaten (Trash-Button).
- Strategy-Toggle "AI passt externe Trades an" pro Coin.
- Clearable Number-Input Component (NumInput) + Codemod (88 Felder).
- max_trades_per_day = 0 Bug (blockierte AI) gefixt.
- Mobile Horizontal Overflow gefixt (mobile.css, App.css).
- **[Feb 2026]** Bezahl-Modelle (deepseek-v4-pro/flash, glm-5.2, qwen3.7-flash, grok-4.20) nutzen NUR den Primär-OpenRouter-Key. Backup-Keys werden übersprungen (die haben meist kein Guthaben → 402). Fällt der Primär-Key aus, greift direkt die Free-Modell-Fallback-Kette. (ai_providers.py `_generate_chain` + `stream_chain`)

## Backlog / Roadmap
- P1: AUC-Skalierung Anzeige-Badge im AI-Trader-Panel.
- P1: Verworfene-Ansicht mit Wiederherstellen.
- P2: Fee-Relax Statistik-Panel.
- P2: Wochenend-Modus (Krypto stärker gewichten).

## Test credentials
Siehe `/app/memory/test_credentials.md`.
