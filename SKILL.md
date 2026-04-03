---
name: crypto-futures-bot
description: Build a disciplined, risk-first crypto futures trading bot for Binance. Use this skill whenever someone asks to build a trading bot, crypto bot, futures bot, Binance bot, automated trading system, or any algorithmic trading project targeting crypto derivatives. Also trigger when users mention position sizing, trailing stops, trading state machines, backtesting crypto strategies, or connecting to Binance Futures API — even if they don't say "bot" explicitly. Covers the full lifecycle from project setup through live deployment with micro capital.
---

# Crypto Futures Trading Bot — Build Skill

This skill guides Claude through building a complete, risk-first Binance Futures trading bot in Python. It's opinionated by design — the decisions here come from real experience shipping bots that survive live markets.

## Core Philosophy

Capital preservation beats profit chasing. The bot follows rules, not intuition. Losses are expected and budgeted for. The system only trades when a defined edge exists.

Hard constraints that apply to every decision:
- Never risk account survival for short-term gain
- Every position gets a stop loss — no exceptions
- No revenge trading, no overtrading, no doubling down

## Architecture (5 Layers)

The bot is built as 5 tightly coupled layers, each with a clear responsibility:

1. **Strategy Engine** — generates BUY/SELL signals from indicators
2. **Risk Engine** — sizes positions, enforces daily/consecutive loss limits
3. **Execution Engine** — places orders on Binance, attaches stops, handles retries
4. **Lifecycle Manager** — state machine that orchestrates the trading flow
5. **Psychology Layer** — hard-coded rules preventing destructive behavior

Build them in this order. Each layer depends on the ones before it.

## Tech Stack (Locked)

- Python 3.11+ with asyncio
- `binance-futures-connector` — REST API for USDT-M futures
- `binance-connector-python` — WebSocket streams for market data + User Data Stream
- `websockets` — fallback for raw WS connections
- `pydantic` + `pydantic-settings` — config validation, data models
- `loguru` — logging with rotation
- `aiosqlite` — async trade log (SQLite, not Postgres — upgrade when needed)
- `numpy` — indicator math
- `pytest` + `pytest-asyncio` — testing

Packages explicitly NOT used: ccxt (direct Binance SDK only), pandas, TA-Lib, SQLAlchemy, Redis, Celery.

## Strategy Specification

Read `references/strategy.md` for the full strategy rules including:
- Symbols (BTCUSDT, ETHUSDT), timeframes (15m primary, 1h confirmation)
- Entry rules for LONG and SHORT (EMA 50 + 20-period breakout)
- Exit rules (initial stop + trailing stop on structure)
- Non-trade filters (chop, volatility, spread)

## Risk Specification

Read `references/risk.md` for the full risk engine rules including:
- Position sizing formula
- Daily loss limit (3%), per-trade risk (0.5-1%)
- Consecutive loss pause (3 in a row → cooldown)
- Leverage management (2-5x max, volatility-adjusted)
- The unified risk gate function

## Execution Specification

Read `references/execution.md` for the full execution engine including:
- Binance API order placement with real code examples
- Stop loss attachment (STOP_MARKET with closePosition)
- Trailing stop logic (cancel + replace)
- Retry logic with Binance-specific error codes
- Position reconciliation
- Emergency close procedure
- User Data Stream for real-time fill updates

## Build Sequence

Read `references/sprints.md` for the sprint-by-sprint build plan. The sequence is:

1. **Foundation** — connect to Binance testnet, stream prices, store candles
2. **Strategy** — implement indicators and signal generation
3. **Risk** — position sizing, kill switches, all safety limits
4. **Execution** — orders on testnet, stop attachment, trailing stops
5. **Lifecycle + Psychology** — state machine, anti-revenge, overtrading prevention
6. **Validate + Deploy** — backtest, 2+ weeks paper trading, micro capital go-live

Each sprint has explicit acceptance criteria. Don't skip ahead.

## Project Structure

```
bot/
├── config/
│   ├── settings.py          # pydantic config model
│   └── pairs.yaml           # symbol configs
├── strategy/
│   ├── signals.py           # signal generation
│   ├── indicators.py        # EMA, breakout levels
│   └── filters.py           # non-trade conditions
├── risk/
│   ├── sizing.py            # position size calculator
│   ├── limits.py            # daily loss, consecutive loss tracking
│   └── rules.py             # kill switches
├── execution/
│   ├── orders.py            # order placement + retry
│   ├── trailing.py          # trailing stop logic
│   └── reconcile.py         # position reconciliation
├── lifecycle/
│   ├── state_machine.py     # state transitions
│   └── manager.py           # orchestrator
├── data/
│   ├── feed.py              # websocket price stream
│   ├── candles.py           # OHLCV fetch + cache
│   └── store.py             # SQLite trade log
├── backtest/
│   ├── engine.py            # backtest runner
│   └── report.py            # metrics + equity curve
├── tests/
├── main.py
├── requirements.txt
└── .env.example
```

## Critical Rules for Claude

When building any part of this bot:

1. **Testnet first.** Always use `base_url="https://testnet.binancefuture.com"` until explicitly deploying.
2. **Stop loss is mandatory.** If SL placement fails after entry, immediately close the position. No orphan trades.
3. **risk_check() before every order.** No bypasses, no "just this once."
4. **Log every decision.** Entries, exits, rejections (with reason), errors, daily PnL.
5. **No look-ahead bias in indicators.** Only use closed candles. Never use the current unclosed candle.
6. **Round position sizes DOWN to Binance step size.** Rounding up = order rejection.
7. **Fetch fresh balance before sizing.** Never use cached balance for position size calculation.
8. **Secrets in .env only.** Never hardcode API keys. Never commit .env to git.

## When the User Asks to Customize

Users will want to change the strategy, symbols, or parameters. That's fine — but enforce these guardrails:

- If they want to add more indicators: warn that complexity ≠ edge. Ship simple first, add later.
- If they want higher leverage: warn explicitly. Above 5x on crypto futures is how accounts blow up.
- If they want to remove risk limits: refuse politely. The risk engine is not optional.
- If they want more symbols: fine, but start with 1-2 and scale after proving the system works.
- If they want to skip paper trading: push back hard. 2 weeks minimum.