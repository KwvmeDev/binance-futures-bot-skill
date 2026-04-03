# Risk Engine Reference

## Position Sizing

### Formula
```
position_size = (account_balance * risk_per_trade) / abs(entry_price - stop_loss_price)
```

### Constraints
- `risk_per_trade`: 0.5% to 1% of account balance (configurable, default 1%)
- Always fetch fresh balance from Binance before calculating (`client.account()`)
- Round position size DOWN to Binance's step size for the symbol (use `exchangeInfo` filters)
- Clamp to exchange minimum/maximum order quantity
- If calculated size < minimum order size → reject trade (don't round up)

### Example
```
balance = $1000, risk = 1%, entry = $65,000, stop = $64,500
size = (1000 * 0.01) / (65000 - 64500) = 10 / 500 = 0.02 BTC
```

## Daily Loss Limit

- Maximum daily loss: 3% of starting balance
- Starting balance: snapshot at 00:00 UTC each day
- Track realized PnL only (unrealized losses don't count)
- When cumulative daily realized loss >= 3% → HALT all trading
- Reset at midnight UTC
- Log HALT events as WARNING level

## Consecutive Loss Tracker

- Track results of last N trades (win/loss)
- If 3 consecutive losses → PAUSE for configurable cooldown (default: 4 hours)
- A winning trade resets the counter to 0
- A breakeven trade (PnL within ±0.1%) counts as neutral, does not reset or increment

## Concurrent Position Limiter

- Max open positions: configurable (default: 1)
- Before every new order: check local state AND reconcile with exchange
- Use `client.get_position_risk()` to verify — don't trust local state alone
- If max reached → reject signal with log

## Leverage Management

- Default leverage: 3x
- Set on startup per symbol: `client.change_leverage(symbol, leverage)`
- Set margin type to ISOLATED: `client.change_margin_type(symbol, "ISOLATED")`
- If ATR(14) > 1.5x its 50-period average → reduce leverage to 2x (high volatility regime)
- Never exceed 5x under any condition, regardless of configuration
- If user requests >5x, warn explicitly about account blow-up risk

## Unified Risk Gate

Single function that checks everything in order. Rejects on first failure.

```python
def risk_check(signal: Signal, account_state: AccountState) -> tuple[bool, str]:
    """Returns (approved, reason). Reason is empty string if approved."""
    
    # 1. Is trading halted? (daily loss limit)
    if account_state.daily_loss_pct >= MAX_DAILY_LOSS:
        return False, f"DAILY_LOSS_HALT: {account_state.daily_loss_pct:.1%} >= {MAX_DAILY_LOSS:.1%}"
    
    # 2. Are we in consecutive loss cooldown?
    if account_state.consecutive_losses >= 3 and not cooldown_expired():
        return False, f"CONSECUTIVE_LOSS_PAUSE: {account_state.consecutive_losses} losses, cooldown active"
    
    # 3. Max concurrent positions?
    if account_state.open_positions >= MAX_CONCURRENT:
        return False, f"MAX_POSITIONS: {account_state.open_positions} >= {MAX_CONCURRENT}"
    
    # 4. Can we size a valid position?
    size = calculate_position_size(signal, account_state.balance)
    if size < MIN_ORDER_SIZE[signal.symbol]:
        return False, f"SIZE_TOO_SMALL: {size} < {MIN_ORDER_SIZE[signal.symbol]}"
    
    # 5. Is leverage appropriate?
    if current_atr_elevated(signal.symbol):
        adjust_leverage(signal.symbol, 2)
    
    return True, ""
```

Every rejected trade must be logged with the rejection reason. This data is critical for system analysis.

## Anti-Doubling-Down Rule

- Never add to a losing position
- Never increase position size after a loss
- If a position is open and in drawdown, the only allowed actions are: hold, tighten stop, or close