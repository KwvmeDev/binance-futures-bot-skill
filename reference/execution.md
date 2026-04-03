# Execution Engine Reference

## Binance Futures API Setup

### Client Initialization
```python
from binance.um_futures import UMFutures

# Testnet
client = UMFutures(
    key=os.getenv("BINANCE_API_KEY"),
    secret=os.getenv("BINANCE_SECRET"),
    base_url="https://testnet.binancefuture.com"
)

# Production (only after Sprint 5 validation)
client = UMFutures(
    key=os.getenv("BINANCE_API_KEY"),
    secret=os.getenv("BINANCE_SECRET")
    # default base_url is production
)
```

### Testnet Toggle
One environment variable controls the toggle. Never use code comments or conditional imports.
```python
TESTNET = os.getenv("TESTNET", "true").lower() == "true"
base_url = "https://testnet.binancefuture.com" if TESTNET else None
```

## Order Placement

### Market Entry
```python
entry_order = client.new_order(
    symbol="BTCUSDT",
    side="BUY",        # or "SELL"
    type="MARKET",
    quantity=0.02       # calculated by risk engine
)
# Response includes: orderId, avgPrice, executedQty, status
```

### Stop Loss (Mandatory)
Place immediately after entry fill. If this fails, emergency close.
```python
sl_order = client.new_order(
    symbol="BTCUSDT",
    side="SELL",         # opposite of entry side
    type="STOP_MARKET",
    stopPrice=str(64500),
    closePosition="true"  # closes entire position at stop price
)
```

### Full Entry Flow
```python
async def open_position(signal: Signal, size: float) -> Order:
    # 1. Place entry
    entry = await execute_with_retry(
        lambda: client.new_order(
            symbol=signal.symbol,
            side="BUY" if signal.side == "LONG" else "SELL",
            type="MARKET",
            quantity=size
        )
    )
    
    if float(entry["executedQty"]) == 0:
        raise ExecutionError("Entry order not filled")
    
    # 2. Immediately attach stop loss
    try:
        sl = await execute_with_retry(
            lambda: client.new_order(
                symbol=signal.symbol,
                side="SELL" if signal.side == "LONG" else "BUY",
                type="STOP_MARKET",
                stopPrice=str(signal.stop_loss),
                closePosition="true"
            )
        )
    except Exception as e:
        logger.critical(f"SL PLACEMENT FAILED: {e}. Emergency closing.")
        await emergency_close(signal.symbol)
        raise
    
    return Order(entry=entry, stop_loss=sl)
```

## Trailing Stop

### Logic
- Track candle closes while in a trade
- LONG: identify new higher lows → move stop below them (minus 0.1% buffer)
- SHORT: identify new lower highs → move stop above them (plus 0.1% buffer)
- Stop can only move in favor of trade (tighten), never loosen

### Implementation (Cancel + Replace)
```python
async def update_trailing_stop(position, new_stop_price: float):
    # Safety check: stop can only tighten
    if position.side == "LONG" and new_stop_price <= position.current_stop:
        return  # would loosen stop — reject
    if position.side == "SHORT" and new_stop_price >= position.current_stop:
        return  # would loosen stop — reject
    
    # Cancel existing stop
    client.cancel_order(symbol=position.symbol, orderId=position.sl_order_id)
    
    # Place new stop
    sl = client.new_order(
        symbol=position.symbol,
        side="SELL" if position.side == "LONG" else "BUY",
        type="STOP_MARKET",
        stopPrice=str(new_stop_price),
        closePosition="true"
    )
    position.sl_order_id = sl["orderId"]
    position.current_stop = new_stop_price
```

### Alternative: Binance Native Trailing Stop
```python
# Simpler but less control over trail behavior
client.new_order(
    symbol="BTCUSDT",
    side="SELL",
    type="TRAILING_STOP_MARKET",
    callbackRate=1.0,  # 1% trailing distance
    quantity=size
)
```
Use the manual cancel+replace approach for structure-based trailing. Use native trailing stop only if you want a fixed percentage trail.

## Retry Logic

```python
async def execute_with_retry(fn, max_retries=3, backoff=1.0):
    for attempt in range(max_retries):
        try:
            return fn()
        except ClientError as e:
            # Don't retry client errors (bad params, insufficient balance)
            if e.error_code in (-1102, -2019, -4003):
                logger.error(f"Non-retryable error: {e.error_code} {e.error_message}")
                raise
            # Retry on rate limit
            if e.error_code == -1003:
                wait = backoff * (2 ** attempt)
                logger.warning(f"Rate limited, retrying in {wait}s")
                await asyncio.sleep(wait)
                continue
            raise
        except Exception as e:
            # Retry on network/timeout errors
            if attempt < max_retries - 1:
                wait = backoff * (2 ** attempt)
                logger.warning(f"Attempt {attempt+1} failed: {e}, retrying in {wait}s")
                await asyncio.sleep(wait)
            else:
                raise
```

### Binance Error Codes to Know
| Code | Meaning | Retry? |
|------|---------|--------|
| -1003 | Too many requests (rate limit) | Yes, with backoff |
| -1015 | Too many orders | Yes, with longer backoff |
| -1102 | Invalid parameter | No |
| -2019 | Insufficient margin | No |
| -4003 | Quantity too small | No |
| -5xx | Server error | Yes |

## Position Reconciliation

```python
async def reconcile():
    """Sync local state with exchange truth."""
    positions = client.get_position_risk()
    open_on_exchange = [
        p for p in positions
        if float(p["positionAmt"]) != 0
    ]
    
    open_orders = client.get_orders(symbol=symbol)
    
    # Compare with local state and log mismatches
    for pos in open_on_exchange:
        if pos["symbol"] not in local_state.positions:
            logger.warning(f"RECONCILE: Found orphan position {pos['symbol']} on exchange")
            # Adopt into local state or close it
    
    for symbol in local_state.positions:
        if symbol not in [p["symbol"] for p in open_on_exchange]:
            logger.warning(f"RECONCILE: Local position {symbol} not found on exchange")
            # Remove from local state
```

Run on startup and every 5 minutes during operation.

## Emergency Close

```python
async def emergency_close(symbol: str):
    """Flatten everything for a symbol. Called on critical failures."""
    logger.critical(f"EMERGENCY CLOSE: {symbol}")
    
    # 1. Cancel all open orders
    try:
        client.cancel_open_orders(symbol=symbol)
    except Exception as e:
        logger.error(f"Failed to cancel orders: {e}")
    
    # 2. Close position
    positions = client.get_position_risk()
    for pos in positions:
        if pos["symbol"] == symbol and float(pos["positionAmt"]) != 0:
            amt = abs(float(pos["positionAmt"]))
            side = "SELL" if float(pos["positionAmt"]) > 0 else "BUY"
            client.new_order(
                symbol=symbol, side=side, type="MARKET", quantity=amt
            )
    
    logger.critical(f"EMERGENCY CLOSE COMPLETE: {symbol}")
```

## User Data Stream (Real-Time Updates)

Subscribe to order and position updates instead of polling:
```python
# Get listen key
response = client.new_listen_key()
listen_key = response["listenKey"]

# Connect to user data stream
ws_url = f"wss://fstream.binance.com/ws/{listen_key}"

# Renew listen key every 30 minutes
# client.renew_listen_key(listenKey=listen_key)
```

Events you'll receive: ORDER_TRADE_UPDATE (fills, cancels), ACCOUNT_UPDATE (balance changes).