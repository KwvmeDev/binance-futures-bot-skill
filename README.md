# Binance Futures Trading Bot — Claude Skill

A Claude skill that builds you a disciplined, risk-first crypto futures trading bot for Binance. Tell Claude what you want, and it scaffolds the entire system — strategy engine, risk management, execution layer, state machine, and deployment pipeline.

No boilerplate tutorials. No "hello world" trading bots that skip risk management. This is the full thing.

## What It Does

Install the skill, then tell Claude something like:

> "Build me a Binance futures trading bot"

Claude will build a complete Python bot across 6 sprints:

| Sprint | What You Get |
|--------|-------------|
| 0 — Foundation | Binance testnet connection, live price streaming, candle caching, trade logging |
| 1 — Strategy | EMA 50 trend filter + 20-period breakout signals with chop/volatility filters |
| 2 — Risk | Position sizing, 3% daily loss halt, consecutive loss cooldown, leverage management |
| 3 — Execution | Market orders, auto stop-loss attachment, trailing stops, retry logic, emergency close |
| 4 — Lifecycle | State machine, anti-revenge trading, overtrading prevention, daily reports |
| 5 — Deploy | Backtesting, 2+ weeks paper trading, micro-capital live deployment ($25-50) |

## Why This Exists

Most trading bot tutorials teach you how to place an order. Then you lose money because there's no risk management, no state tracking, and no kill switches.

This skill forces the boring stuff first: capital preservation, position sizing, loss limits, and rule enforcement. The strategy is intentionally simple (EMA + breakout). The risk engine is not.

## The Stack

- **Python 3.11+** with asyncio
- **binance-futures-connector** — REST API for USDT-M futures
- **binance-connector-python** — WebSocket streams
- **pydantic** — config and data validation
- **loguru** — logging
- **aiosqlite** — trade storage
- **numpy** — indicator math

No ccxt, no pandas, no TA-Lib, no SQLAlchemy, no Redis. Lean on purpose.

## Install

Download the `.skill` file from [Releases](../../releases) and upload it to Claude:

1. Open Claude (claude.ai or Claude app)
2. Go to **Settings**
3. Upload `crypto-futures-bot.skill`
4. Start a new conversation and ask Claude to build your bot

## What's Inside the Skill

```
crypto-futures-bot/
├── SKILL.md                      # Main skill file — architecture, rules, guardrails
└── references/
    ├── strategy.md               # Entry/exit rules, indicators, filters
    ├── risk.md                   # Position sizing, loss limits, risk gate
    ├── execution.md              # Binance API code, stops, retries, emergency close
    └── sprints.md                # 6-sprint build sequence with acceptance criteria
```

Claude reads `SKILL.md` first, then dives into the relevant reference file based on what you're working on. You don't need to read these yourself — but you can if you want to understand or customize the system.

## Strategy at a Glance

**Symbols:** BTCUSDT, ETHUSDT
**Timeframes:** 15m (signals), 1h (confirmation)

**LONG:** Price > 50 EMA, breaks above 20-period high, 1h confirms uptrend
**SHORT:** Price < 50 EMA, breaks below 20-period low, 1h confirms downtrend

**Exit:** Initial stop at recent swing + trailing stop that follows market structure. No fixed take profit — let winners run.

**Filters:** Skip choppy markets, low volatility, wide spreads.

This is a starting point. The skill supports customization, but pushes back if you try to remove risk limits or crank leverage past 5x.

## Risk Rules (Non-Negotiable)

These are hardcoded into the skill. Claude will enforce them even if you ask it not to.

- **Max risk per trade:** 0.5–1% of account
- **Max daily loss:** 3% → trading halts until next day
- **3 consecutive losses** → 4-hour cooldown
- **Max leverage:** 5x (reduced to 2x in high volatility)
- **Every position gets a stop loss.** If SL placement fails, position closes immediately.
- **No revenge trading.** Minimum 30-minute cooldown after any loss.
- **No overtrading.** Max 6 entries per day.

## Customization

You can modify the strategy, symbols, and parameters. Claude will help. But it will push back on:

- Removing risk limits ("just skip the daily loss check") → **No.**
- Leverage above 5x → **Warning about account blow-up risk.**
- Skipping paper trading → **Hard no. 2 weeks minimum.**
- Adding 15 indicators → **Ships simple first, add complexity later.**

## Before You Start

You'll need:

1. **Binance Futures Testnet account** — https://testnet.binancefuture.com (sign in with GitHub)
2. **API key + secret** from testnet
3. **Python 3.11+**

You do NOT need real money until Sprint 5 is complete and validated.

## Disclaimer

This is a tool for building trading systems. It is not financial advice. Crypto futures trading involves significant risk of loss. The strategy included is a starting point — not a guaranteed edge. Past backtest performance does not predict future results. Start with testnet, then paper trading, then micro capital. Never risk money you can't afford to lose.

## License

MIT

## Contributing

Found a bug in the skill? Want to add a strategy variant? PRs welcome. Keep it lean — the whole point is disciplined simplicity.
