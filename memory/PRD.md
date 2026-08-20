# PRD – KI-Daytrading-Website (extern deployt auf Render)

## Original-Problemstellung
Produktiv laufende Daytrading-Website (GitHub: dean06greif-ai/Neu-Bitch, Deploy: Render,
Daten: MongoDB Atlas + Supabase, Trading: Bitunix). Verbesserungen müssen sauber, modular,
rückwärtskompatibel und mit Regressionstests erfolgen. Original-Ordnerstruktur beibehalten
(Render-Deploy). Letzte Session war abgebrochen – Auftrag: Abbruchstelle finden, fortsetzen.

## Architektur
- backend/: FastAPI (server.py + routers/ + services/ + strategies/ + core/), MongoDB via MONGO_URL
- frontend/: React (CRA + craco), Komponenten unter src/components, zentraler Modell-Katalog src/lib/aiModels.js
- local_worker/: lokaler Rechen-Worker (Outbound-Polling, Token-Auth) – MUSS im Repo bleiben (.gitignore-Whitelist)
- scripts/, tests/ (Backend-Regressionstests, pytest + conftest lädt beide .env)
- Auth: Admin via ADMIN_USER/ADMIN_PASSWORD (JWT), s. /app/memory/test_credentials.md

## Kern-Features (statisch)
- Live-Scanner + Chart, Auto-/Paper-/Live-Trading (Bitunix), Trade-Guard (Kill-Switch, Anti-Stacking, SL-Ratchet, Fee-Wächter)
- KI-Trader (Multi-Provider: Cerebras/Groq/Gemini/Mistral/OpenRouter mit unbegrenzten BACKUP-Keys + Rotation)
- KI-Team-Rollen (analyst, deep_analyst, learner, supervisor …) mit Fallback-Kaskade + Auto-Switch
- News-Watcher mit automatischer Tiefenanalyse bei mittel/hoch-wichtigen Events (should_trigger_deep)
- Modell-Wächter: neue Modelle nur nach Bestätigung (approve-Flow, /api/ai/models/*)
- Analyse-Intervalle ab voller Stunde ausgerichtet (next_aligned_ts in ai_schedule)
- Lern-System (Lektionen mit Validierung, Kontext-Hinweisen, Master-Prompt-Audit), Regime-Engine (reaktiv + Regression)
- ML-Gate (Shadow-Mode, Training auf Prod-Daten), Optimizer (Cloud + lokaler Worker)

## Diese Session umgesetzt (19.08.2026, Teil 2 – Feature-Ausbau)
1. Echter Orderflow (services/orderflow.py): Bitunix Public-WS Trade-Channel für TOP_10_COINS,
   Tick-Delta 1/5/15m, CVD-Trend, Großaufträge (95. Perzentil), Liquidity-Sweep-Heuristik.
   In ai_engine-Marktkontext integriert (Fallback: alter Kerzen-Proxy). API: GET /api/ai/orderflow/{symbol}
2. Lern-Reset: AILearning.reevaluate_lessons + POST /api/ai/lessons/reevaluate (Admin) –
   LLM bewertet jede Lektion (gueltig/veraltet/anpassen) gegen die AKTUELLE Strategie;
   LOCKED/Trader-Lektionen sind unantastbar. Frontend-Button "Neu bewerten" im Lernen-Tab
   (data-testid="ai-lessons-reevaluate-btn"). Governance-Eintrag im KI-Chat.
3. Tests: tests/test_orderflow_and_lesson_reeval.py (7 Tests, grün)

## Diese Session umgesetzt (19.08.2026 – Fortsetzung der abgebrochenen Session)
1. Repo-Stand von GitHub in frische Umgebung wiederhergestellt (inkl. Root-.gitignore/README – fehlten beim rsync)
2. Abbruchstelle gefunden & fertiggestellt: strong_speed_ratio war nur in Config/Validator, NICHT im Klassifikator:
   - regime_engine: strong-Achse (Score-Hysterese) zusätzlich mit realem Tempo (Netto-%/Tag vs. Tagesvola) ge-gated
   - regime_reactive (kausal): Frische-Fenster – "stark" erlischt, wenn jüngstes Tempo < 0.5×Einstiegs-Schwelle
   - Validator: strong-Check berücksichtigt Sichtfenster (kein Lookahead-Vorwurf)
3. Echter Produktions-Bugfix: bitunix get_mark_price nahm data[0] ohne Symbol-Abgleich → bei unbekannten
   Symbolen falscher Mark-Preis (falsche SL-Clamps). Jetzt strikter Symbol-Match.
4. ai_roles: Kosten-Migration erhält jetzt Nutzer-Ein/Aus-Schalter (nur Modelle werden auf Presets gesetzt)
5. .gitignore: Whitelist "!local_worker/" ergänzt (Render-Deploy-Schutz, von Test gefordert)
6. Testsuite modernisiert (stale Tests an bewusst geänderte Produkt-Entscheidungen angepasst):
   - Credentials überall env-getrieben (5 Dateien mit alten/hartkodierten Passwörtern)
   - tote Modelle ersetzt (llama-3.3-70b-versatile→openai/gpt-oss-120b, llama-3.1-8b-instant→openai/gpt-oss-20b)
   - crv_max=4.0-Migration, 422-Reject mit Fix-Vorschlägen (statt Warnungen), Supervisor-Kaskade,
     Watchdog konfiguriert/unkonfiguriert, ML-Gate-Prod-Tests skippen ohne Prod-Daten
   - e2e-Trade-Tests räumen eigene Paper-Trades auf (Anti-Stacking-Cooldown)
   - alte Preview-URL in test_iter2_review_e2e env-getrieben
7. Lokaler Worker in Sandbox gestartet (PYTHONPATH=backend), Kill-Switch/Test-Trades bereinigt

## Teststatus (Sandbox)
- Kern-Unit-Tests (Regime, KI-Lab, Lektionen, Learning, Governance, Playbook): 100% grün
- E2E: von 74 fail/32 error auf 14 fail reduziert; Rest ist prod-daten-/zustandsabhängig
  (keine geschlossenen Trades/Historie in Sandbox-DB, ML-Gate-Prod-Daten, Kill-Switch durch Test-Paper-Verluste,
  1× 502 Ingress-Hiccup, 2 knappe synthetische Kalibrier-Schwellen). Keine Code-Regressionen offen.

## Diese Session umgesetzt (20.06.2026 – 5 Verbesserungen aus KI-Selbstanalyse + User-Feedback)
1. Orderflow Forex/Rohstoffe (KI-Wunsch 1):
   - server.py abonniert Bitunix-Trade-Channel jetzt auch für Rohstoff-/Index-Perps
     (XAUUSDT, XAGUSDT, CLUSDT, QQQUSDT, SPYUSDT) → ECHTE Tick-Daten statt Kerzen-Proxy
   - ai_engine._snapshot mappt symbol→instrument.bitunix (GOLD→XAUUSDT usw.)
   - NEU services/fx_orderflow.py: CME-Futures-Volumen-Näherung für Forex (6E/6B/6J/6A/6C/6S/6N
     via Yahoo, echtes Futures-Volumen → Delta 15m/60m, CVD-Trend, Vol-Spikes; invertierte
     Kontrakte wie 6J korrekt gedreht); Fallback auch für GOLD/SILVER (GC=F/SI=F)
   - GET /api/ai/orderflow/{symbol} liefert of_symbol-Mapping + fx_futures_proxy
2. Dynamische Timeframe-Anpassung (KI-Wunsch 2): bei 1m-ATR < lowvol_atr_pct (Default 0.10%)
   liefert der Snapshot 5m/15m-ATR + Marker "⚠ NIEDRIGE VOLATILITÄT"; Analyse-Rahmen weist die
   KI an, statt HOLD auf 5m/15m-Struktur zu wechseln (config: lowvol_tf_switch, Default an)
3. ML-Positionssteuerung (KI-Wunsch 3): ml_gate.size_factor() → Positionsgröße ×0.5 wenn
   Gate-OOS-AUC < 0.55 (Settings: auc_risk_scaling/auc_min/auc_risk_factor, via Gate-Settings-API);
   Koppelung: ai_engine._emit_signal setzt signal.ml_risk_scale → bitunix_trade skaliert Margin;
   sichtbar in /api/ml/gate/status (size_factor + Begründung). Live aktiv (v10: AUC 0.539 → ×0.5)
4. News-Tiefenanalyse Token-Optimierung: Dedupe über normalisierte Event-Titel (identisches
   Ereignis triggert keine 2. Tiefenanalyse, nur NEUE News/Änderungen); News-getriggerte
   Deep-Runs laufen im Lean-Modus (nur betroffene Assets + BTC/ETH, 10 statt 25 Headlines,
   ohne Research-/ML-/Performance-Blöcke, Prompt-Anweisung "nur Änderung analysieren")
5. OpenRouter Bezahl-Modelle (DeepSeek): 429 = transientes Minuten-Limit → bis zu 3 Retries
   (6/15/30s) auf demselben Key statt sofortigem Fallback; Key wird NICHT in den 10-min-Cooldown
   geschickt (poisont sonst Free-Calls); 402 (kein Guthaben) fällt weiter normal zurück.
   Bonus-Bugfix: stream_chain crashte bei 429 mit NameError (idxs/streak_429 undefiniert) – behoben
6. Regressionstests: tests/test_improvements_orderflow_ml_news.py (20 Tests) +
   tests/test_iter50_live_endpoints.py (Testing-Agent, 6 Live-Smoke-Tests) – 39/39 grün.
   Bekannt: test_iter49::test_other_errors_still_skip_model_immediately flaked schon im
   Original-Repo (Round-Robin-Modulstate) – keine Regression dieser Session

## Backlog / Nächste Aufgaben
- P1: Dynamischer Gebühren-Filter (Punkt 2 aus KI-Selbstanalyse) weiter verfeinern
- P2: Orderflow-Daten im Frontend visualisieren (Delta/CVD-Mini-Chart, inkl. FX-Futures-Proxy)
- P2: Restliche prod-datenabhängige Tests mit Fixture-Seeds versorgen
- P2: worker_config.json aus Sandbox nicht committen (enthält lokalen Token)
- P2: Orderflow-WS Reconnect-Backoff (Log-Rauschen beim Shutdown)
- P2: Vorbestehenden Round-Robin-Testflake in test_iter49 deterministisch machen
