# Strategy Engine Reference

## Market Scope

- Symbols: BTCUSDT, ETHUSDT
- Primary timeframe: 15m (signal generation)
- Confirmation timeframe: 1h (trend filter)
- Candle cache: last 500 candles per symbol per timeframe

## Indicators

### EMA 50 (Trend Filter)
- Calculate on close prices only
- Use on both 15m and 1h timeframes
- Purpose: determine if market is trending up or down

### 20-Period High/Low (Breakout Levels)
- `highest_high_20`: highest high of last 20 closed candles
- `lowest_low_20`: lowest low of last 20 closed candles
- Purpose: define breakout levels for entry triggers

### ATR 14 (Volatility Measure)
- Used by non-trade filters and risk engine
- Not used for entry signals directly

## Entry Rules

### LONG Entry
All three conditions must be true simultaneously:
1. 15m close > 50 EMA (15m)
2. 15m close > 20-period high (breakout confirmation)
3. 1h close > 50 EMA (1h) — higher timeframe alignment

### SHORT Entry
All three conditions must be true simultaneously:
1. 15m close < 50 EMA (15m)
2. 15m close < 20-period low (breakdown confirmation)
3. 1h close < 50 EMA (1h) — higher timeframe alignment

### Signal Data Model
```python
@dataclass
class Signal:
    symbol: str           # e.g. "BTCUSDT"
    side: str             # "LONG" or "SHORT"
    entry_price: float    # price at signal (candle close)
    stop_loss: float      # initial stop level
    timestamp: datetime
    timeframe: str        # "15m"
    reason: str           # human-readable signal description
```

## Exit Rules

### Initial Stop Loss
- LONG: below the most recent swing low (lowest low of last 5 candles minus 0.1% buffer)
- SHORT: above the most recent swing high (highest high of last 5 candles plus 0.1% buffer)
- Placed immediately as STOP_MARKET order after entry fill

### Trailing Stop (Structure-Based)
- LONG: when a new higher low forms (higher than previous swing low), move stop to just below it
- SHORT: when a new lower high forms (lower than previous swing high), move stop to just above it
- Buffer: 0.1% of price to avoid getting stopped by noise
- Stop can only tighten (move in favor of trade) — never move it against the trade
- Implementation: cancel existing STOP_MARKET, place new one at updated level

### Take Profit
- No fixed take profit target. Let the trailing stop handle exits.
- The system is designed to let winners run.

## Non-Trade Filters

Each filter returns `(should_skip: bool, reason: str)`. Log every rejection.

### Chop Filter
- If price has crossed EMA 50 more than 3 times in the last 20 candles → skip
- Indicates sideways/ranging market where breakout signals are unreliable

### Volatility Filter
- If ATR(14) < configurable threshold → skip
- Low volatility means the breakout is unlikely to have follow-through
- Suggested thresholds: BTC 0.3% of price, ETH 0.4% of price

### Spread Filter
- If current bid-ask spread > 0.05% → skip
- Wide spreads eat into edge and signal poor liquidity

## Indicator Implementation Notes

- Build indicators from raw math using numpy. Do not use TA-Lib.
- EMA formula: `EMA_today = (close - EMA_yesterday) * multiplier + EMA_yesterday` where `multiplier = 2 / (period + 1)`
- All indicators use CLOSED candles only. The current unclosed candle must never be used.
- Signals fire on candle close events, not intrabar.