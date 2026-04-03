# Sprint Build Sequence

## Sprint 0 — Foundation (~1 week)

**Goal:** Bot boots, connects to Binance testnet, streams prices, stores candles.

Tasks:
- Project setup: venv, packages (`binance-futures-connector`, `binance-connector-python`, `websockets`, `pydantic`, `pydantic-settings`, `python-dotenv`, `loguru`, `aiosqlite`, `numpy`, `pytest`, `pytest-asyncio`)
- Exchange connection: wrap `UMFutures` client, testnet toggle via env var, health check prints balance
- OHLCV fetch: historical candles for BTCUSDT + ETHUSDT on 15m and 1h, cache 500 in deque
- WebSocket stream: `wss://fstream.binance.com/ws/<symbol>@kline_15m`, append on candle close (`k.x == true`), auto-reconnect with backoff
- SQLite store: `trades` table (id, symbol, side, entry_price, exit_price, size, pnl, timestamp, status), `events` table (id, type, message, timestamp)
- Logging: loguru with console + file rotation (10MB, keep 7)

Done when: bot connects to testnet, streams live 15m candles, stores/retrieves trades, logs cleanly.

## Sprint 1 — Strategy Engine (~2 weeks)

**Goal:** Generate BUY/SELL signals. Validate against historical data.

Tasks:
- Indicator functions: `ema(candles, period)`, `highest_high(candles, period)`, `lowest_low(candles, period)` — raw numpy math, no TA-Lib
- Signal generator: `evaluate(symbol, candles_15m, candles_1h)` → `Signal` or `None`. LONG: price > EMA50 + breakout above 20-period high + 1h confirmation. SHORT: inverse.
- Non-trade filters: chop filter (3+ EMA crosses in 20 candles), volatility filter (ATR < threshold), spread filter (>0.05%). Each returns `(bool, reason)`.
- Validation script: run strategy over 30 days, output signal list, eyeball on chart.

Done when: signals match chart visually, no look-ahead bias, filters reject choppy periods.

## Sprint 2 — Risk Engine (~1.5 weeks)

**Goal:** Position sizing and all kill switches work.

Tasks:
- Position size calculator: `(balance * risk_pct) / abs(entry - stop)`, clamped to Binance step size (round DOWN)
- Daily loss tracker: 3% of starting balance (UTC day) → HALT
- Consecutive loss tracker: 3 in a row → 4-hour cooldown pause
- Concurrent position limiter: default 1, reconcile with exchange before check
- Leverage manager: `client.change_leverage()`, `client.change_margin_type("ISOLATED")`, reduce to 2x if ATR elevated, never exceed 5x
- Unified risk gate: `risk_check(signal) → (approved, reason)`, checks all above in order, rejects on first failure

Done when: all limits fire correctly in simulated scenarios, every rejection logged with reason.

## Sprint 3 — Execution Engine (~2 weeks)

**Goal:** Orders land on Binance testnet. Stops attach. No orphan positions.

Tasks:
- Order placement: `client.new_order()` MARKET orders, return fill details
- Stop loss: STOP_MARKET with `closePosition="true"` immediately after fill. If SL fails → emergency close.
- Trailing stop: track candle closes, move stop on new swing structure, cancel + replace STOP_MARKET. Never loosen stop.
- Retry logic: retry on timeout/429/-1003/5xx, fail immediately on -2019/-1102/-4003, max 3 attempts with exponential backoff
- Reconciliation: `client.get_position_risk()` on startup + every 5 min, sync mismatches
- Emergency close: `cancel_open_orders()` then market close, log as CRITICAL

Done when: testnet orders execute, every position has a stop, failed SL triggers emergency close.

## Sprint 4 — Lifecycle + Psychology (~2 weeks)

**Goal:** Full state machine. All safety rails. Autonomous testnet operation.

Tasks:
- State machine: IDLE → SCANNING → SIGNAL_DETECTED → VALIDATING → POSITION_OPEN → MANAGING_TRADE → EXITED → SCANNING. Log every transition. Invalid transition = error.
- Main loop: async, wakes on candle close, calls strategy → risk → execution in order, graceful shutdown on SIGINT/SIGTERM
- Anti-revenge trading: minimum 2-candle (30 min) wait after any loss
- Overtrading prevention: max 6 trades per day, reset midnight UTC
- System deviation guard: every order must trace to a Signal object, no manual override
- Daily report: starting/ending balance, PnL, trades, rejections with reasons, max drawdown

Done when: bot runs 24h+ on testnet, all safety rails fire correctly, daily report generates.

## Sprint 5 — Backtest + Paper + Deploy (~3 weeks)

**Goal:** Prove the system on historical data. Paper trade 2+ weeks. Go live with micro capital.

Tasks:
- Backtest engine: feed historical candles through full pipeline, simulate fills at close + 0.02% slippage + 0.04% fees
- Metrics: win rate, avg win/loss, profit factor, max drawdown, equity curve, monthly breakdown
- Sanity check: win rate 30-55% is normal. Profit factor > 1.2. Max DD < 15%. If it looks too good, it's a bug.
- Paper trading mode: `MODE=paper` config flag, everything live except orders simulated locally, run 2+ weeks minimum
- Paper review checklist: 24/7 stability, risk limits fired, PnL within bounds, trailing stops correct
- Live deployment: prod API keys, start with $25-50, `RISK_PER_TRADE=0.005`, monitor first 10 trades manually

Scaling: Week 1-2 $25-50 → Week 3-4 $100-200 → Month 2 $500 → Month 3+ at own discretion.

Done when: 2 weeks paper clean, first 10 live trades execute correctly with proper risk management.