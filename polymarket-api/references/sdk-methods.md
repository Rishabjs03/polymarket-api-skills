# SDK Methods Reference

Sourced from official Polymarket GitHub repos and docs.
Python: py-clob-client v0.34.6 | TypeScript: @polymarket/clob-client v5.2.1

---

## Python SDK — Complete Method List

### Public Methods (no credentials needed)

```python
client.get_ok()                            # health check → "OK"
client.get_server_time()                   # server Unix timestamp

# Prices
client.get_price(token_id, side)           # side = "BUY" or "SELL" → {"price": "0.65"}
client.get_prices([token_id], side)        # batch prices
client.get_midpoint(token_id)              # mid-market price → {"mid": "0.645"}
client.get_midpoints([token_id])           # batch midpoints
client.get_spread(token_id)               # bid-ask spread
client.get_spreads([token_id])             # batch spreads
client.get_last_trade_price(token_id)      # last matched price
client.get_last_trade_prices([token_id])   # batch last trade prices

# Orderbook
client.get_order_book(token_id)            # OrderBookSummary object with .bids, .asks
client.get_order_books([BookParams(...)])  # batch orderbooks, BookParams from clob_types

# Market listings
client.get_simplified_markets(next_cursor="")  # paginated → {"data": [...], "next_cursor": "..."}
client.get_sampling_markets(next_cursor="")
client.get_sampling_simplified_markets(next_cursor="")
client.get_markets(next_cursor="")

# Market data
client.get_tick_size(token_id)             # → {"minimum_tick_size": "0.01"}
client.get_fee_rate(token_id)              # trading fee rate

# Allowances (EOA only)
client.get_balance_allowance(params)       # check current USDC/CTF allowance
client.update_balance_allowance(params)    # set allowance (requires private key)
```

### L1 Methods (requires private key, no API creds)

```python
client.create_api_key()                    # create new L2 credentials
client.derive_api_key()                    # derive deterministic L2 creds from private key
client.create_or_derive_api_creds()        # derive if exists, else create — use this
client.get_api_keys()                      # list all L2 keys for this wallet
client.delete_api_key()                    # revoke current L2 key
```

### L2 Methods (requires private key + API credentials)

```python
# Order creation
client.create_order(order_args, options)   # OrderArgs → signed order object
client.create_market_order(market_order_args)  # MarketOrderArgs → signed order
client.post_order(order, order_type)       # submit signed order → {"success", "orderID", "status"}
client.create_and_post_order(order_args)   # convenience: create + post in one call

# Order management
client.get_order(order_id)                 # single order by ID
client.get_orders(params)                  # list orders, OpenOrderParams() for all open
client.cancel(order_id)                    # cancel single order
client.cancel_orders([order_id, ...])      # cancel multiple orders
client.cancel_all()                        # cancel all open orders
client.cancel_market_orders(asset_id)      # cancel all orders for a token

# Trades
client.get_trades(params)                  # trade history, TradeParams for filtering

# Heartbeat (for liquidity reward programs)
client.send_heartbeat()                    # signal liveness

# Scoring
client.get_order_scoring(order_id)         # reward/scoring status for an order
```

---

## Key Data Classes (py_clob_client.clob_types)

```python
from py_clob_client.clob_types import (
    OrderArgs,
    MarketOrderArgs,
    OrderType,
    OpenOrderParams,
    TradeParams,
    BookParams,
    ApiCreds,
)
from py_clob_client.order_builder.constants import BUY, SELL

# Limit order
OrderArgs(
    token_id: str,     # outcome token ID
    price: float,      # 0.0–1.0
    size: float,       # USDC amount (shares × price)
    side: BUY | SELL,
    # optional:
    expiration: int,   # Unix timestamp, only for GTD orders
)

# Market order
MarketOrderArgs(
    token_id: str,
    amount: float,     # USDC to spend (for BUY) or shares to sell (for SELL)
    side: BUY | SELL,
    order_type: OrderType.FOK,
)

# Order types
OrderType.GTC   # Good Till Cancelled
OrderType.FOK   # Fill or Kill
OrderType.FAK   # Fill and Kill (partial fill allowed)
OrderType.GTD   # Good Till Date

# Query open orders
OpenOrderParams(
    market: str = None,   # token_id to filter
)

# Query trades
TradeParams(
    market: str = None,
    maker_address: str = None,
    taker_address: str = None,
)

# Batch orderbooks
BookParams(token_id: str)
```

---

## TypeScript SDK — Key Methods

```typescript
// Public
await client.getMarkets()
await client.getOrderBook(tokenId)              // → OrderBookSummary
await client.getOrderBooks([{tokenID}])         // batch
await client.getPrice(tokenId, "BUY")           // → {price: "0.65"}
await client.getMidpoint(tokenId)               // → {mid: "0.645"}
await client.getSpread(tokenId)
await client.getLastTradePrice(tokenId)
await client.getTickSize(tokenId)               // → {minimum_tick_size: "0.01"}
await client.getServerTime()

// L1 (signer required)
await client.createApiKey()
await client.deriveApiKey()
await client.createOrDeriveApiKey()             // use this

// L2 (signer + creds required)
await client.createOrder({
    tokenID: string,
    price: number,                              // 0.0–1.0
    side: Side.BUY | Side.SELL,
    size: number,
})
await client.postOrder(order, OrderType.GTC)
await client.createAndPostOrder(orderParams, {tickSize, negRisk}, orderType)

await client.getOrder(orderId)
await client.getOrders({ market?, asset_id?, status? })
await client.cancelOrder({ orderID })
await client.cancelOrders({ orderIDs: [] })
await client.cancelAll()
await client.cancelMarketOrders({ asset_id: tokenId })

await client.getTrades({ market?, maker_address?, taker_address? })
await client.getBuilderTrades()                 // builder program only
```

**Error handling in TypeScript — opt-in strict mode:**
```typescript
const client = new ClobClient(
    host, chainId, signer, creds, sigType, funder,
    undefined, undefined, undefined, undefined, undefined, undefined,
    true   // throwOnError — throws ApiError instead of returning {error}
);

try {
    const book = await client.getOrderBook(tokenId);
} catch (e) {
    if (e instanceof ApiError) {
        console.log(e.message);  // "No orderbook exists for the requested token id"
        console.log(e.status);   // 404
        console.log(e.data);     // full error response from API
    }
}
```

By default (without `throwOnError`), the SDK returns `{ error: "...", status: ... }` objects
on failure rather than throwing.

---

## Response Shapes

### Order response
```json
{
  "success": true,
  "orderID": "0x1234...",
  "status": "live",
  "takingAmount": "65.0",
  "makingAmount": "100.0",
  "errorMsg": ""
}
```
Status values: `"live"` (on book), `"matched"` (filled), `"unmatched"` (rejected/no liquidity)

### OrderBookSummary
```json
{
  "market": "token_id",
  "asset_id": "token_id",
  "bids": [{"price": "0.64", "size": "200.0"}],
  "asks": [{"price": "0.65", "size": "150.0"}],
  "hash": "..."
}
```
Bids sorted descending, asks sorted ascending.

### Trade record
```json
{
  "id": "...",
  "tradeID": "...",
  "market": "conditionId",
  "asset_id": "token_id",
  "outcome": "YES",
  "price": "0.65",
  "size": "100.0",
  "side": "BUY",
  "status": "TRADE_STATUS_CONFIRMED",
  "match_time": "1700000000",
  "transaction_hash": "0x...",
  "maker_address": "0x...",
  "taker_address": "0x...",
  "fee_rate_bps": "20"
}
```