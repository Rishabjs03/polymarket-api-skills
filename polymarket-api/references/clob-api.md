# CLOB API Reference

Base: `https://clob.polymarket.com` | Auth: None for reads; L2 HMAC for trading

---

## Pricing (Public — no auth)

### GET /price
Best price for a token on one side.
Params: `token_id` (required), `side` (`BUY` or `SELL`)
Response: `{"price": "0.65"}`

### GET /prices
Batch prices for multiple tokens.
Params: `token_ids` (comma-separated), `side`

### POST /prices
Batch prices via request body.
Body: `{"params": [{"token_id": "...", "side": "BUY"}]}`

### GET /midpoint
Mid-market price — best proxy for implied probability.
Params: `token_id`
Response: `{"mid": "0.645"}`

### GET /midpoints
Params: `token_ids` (comma-separated)

### POST /midpoints
Body: `{"params": [{"token_id": "..."}]}`

### GET /spread
Bid-ask spread.
Params: `token_id`
Response: `{"spread": "0.01"}`

### POST /spreads
Body: `{"params": [{"token_id": "..."}]}`

### GET /last-trade-price
Most recent matched trade price.
Params: `token_id`
Response: `{"price": "0.64"}`

### GET /last-trade-prices
Params: `token_ids` (comma-separated)

### POST /last-trade-prices
Body: `{"params": [{"token_id": "..."}]}`

---

## Orderbook (Public)

### GET /book
Full orderbook for a token.
Params: `token_id`
Response:
```json
{
  "market": "token_id",
  "asset_id": "token_id",
  "bids": [{"price": "0.64", "size": "200.0"}],
  "asks": [{"price": "0.65", "size": "150.0"}],
  "hash": "..."
}
```
Bids descending, asks ascending.
Rate limit: 300 requests/10 seconds for the batch endpoint.

### POST /books
Batch orderbooks — preferred over looping GET /book.
Body: `{"params": [{"token_id": "..."}]}`

---

## Price History (Public)

### GET /prices-history
Historical price timeseries.
Params:
- `market` — token_id (required)
- `interval` — `1m`, `5m`, `1h`, `6h`, `1d`, `1w`
- `fidelity` — number of data points
- `startTs`, `endTs` — Unix timestamp range

Response: `{"history": [{"t": 1700000000, "p": 0.65}, ...]}`

---

## Market Metadata (Public)

### GET /tick-size
Minimum price increment for a market. Always fetch before placing orders.
Params: `token_id`
Response: `{"minimum_tick_size": "0.01"}`
Valid values: `"0.1"`, `"0.01"`, `"0.001"`, `"0.0001"`

### GET /tick-size/{token_id}
Same via path parameter.

### GET /fee-rate
Trading fee for a market.
Params: `token_id`

### GET /fee-rate/{token_id}
Same via path parameter.

### GET /time
Server time (Unix milliseconds).
Response: `{"time": 1700000000000}`

### GET /neg-risk
Whether a market uses negRisk logic.
Params: `token_id`

---

## Market Listings (Public)

### GET /markets
Simplified CLOB market listings.
Params: `next_cursor` (base64 cursor for pagination, start with `""` or `"MA=="`)
Response: `{"data": [...], "next_cursor": "..."}`
Note: cursor `"LTE="` signals no more results.

### GET /sampling-markets
Markets in the CLOB liquidity reward sampling program.

### GET /sampling-simplified
Simplified sampling markets.

---

## Order Management (L2 Auth Required)

All order endpoints require `POLY_ADDRESS`, `POLY_API_KEY`, `POLY_PASSPHRASE`, `POLY_SIGNATURE`, `POLY_TIMESTAMP` headers.

### POST /order
Submit a single signed order.
Body: signed order object constructed by SDK.
Response:
```json
{
  "success": true,
  "orderID": "0x...",
  "status": "live" | "matched" | "unmatched",
  "takingAmount": "65.0",
  "makingAmount": "100.0",
  "errorMsg": ""
}
```

Key order fields (constructed by SDK):
```
tokenID         outcome token ID
price           0.0–1.0
size            USDC amount
side            "BUY" | "SELL"
orderType       "GTC" | "FOK" | "FAK" | "GTD"
expiration      Unix timestamp (GTD only)
negRisk         bool
```

Rate limit: 3,000 per 10 minutes per API key.

### POST /orders
Submit multiple orders at once.

### GET /data/orders
Open and historical orders for the authenticated user.
Params: `market` (token_id), `asset_id`, `status` (`live`|`matched`|`cancelled`|`unmatched`), `limit`, `offset`

### GET /data/orders/{order_id}
Single order by ID.

### DELETE /order
Cancel a single order.
Body: `{"orderID": "0x..."}`

### DELETE /orders
Cancel multiple orders.
Body: `{"orderIDs": ["0x...", ...]}`

### DELETE /orders/all
Cancel all open orders for the authenticated user.

### DELETE /orders/market
Cancel all orders in a specific market.
Body: `{"asset_id": "token_id"}`

### GET /order-scoring
Reward/scoring status for an order. Used in liquidity programs.
Params: `order_id`

### POST /heartbeat
Signal liveness. Used by active market makers to maintain reward eligibility.

---

## Trades (L2 Auth Required)

### GET /data/trades
Trade history.
Params: `market` (token_id), `maker_address`, `taker_address`, `limit`, `offset`

Trade fields:
```
id, tradeID
market          conditionId
asset_id        token_id
outcome         "YES" | "NO"
price           matched price
size            USDC amount
side            "BUY" | "SELL"
status          "TRADE_STATUS_CONFIRMED"
match_time      Unix timestamp string
transaction_hash  Polygon tx hash
maker_address, taker_address
fee_rate_bps    fee in basis points (default 20 = 0.20%)
```

### GET /data/builder-trades
Trades attributed to the authenticated builder account.

---

## WebSocket — `wss://ws-subscriptions-clob.polymarket.com/ws`

### Market Channel (`/ws/market`) — Public

Subscribe:
```json
{
  "auth": {},
  "type": "Market",
  "markets": [],
  "assets_ids": ["token_id_1", "token_id_2"]
}
```

Event types received:
- `book` — full orderbook snapshot `{"asset_id", "bids": [{price, size}], "asks": [{price, size}]}`
- `price_change` — delta `{"asset_id", "changes": [{price, size, side}]}`
- `last_trade_price` — `{"asset_id", "price"}`
- `tick_size_change`

### User Channel (`/ws/user`) — L2 credentials required

Subscribe:
```json
{
  "auth": {
    "apiKey": "your-api-key-uuid",
    "secret": "your-api-secret",
    "passphrase": "your-passphrase"
  },
  "type": "User"
}
```

Then subscribe to markets:
```json
{"operation": "subscribe", "markets": ["conditionId"]}
```

Event types: `order` (order status changes), `trade` (fill confirmations), `balance`

### Sports Channel (`/ws/sports`)
Real-time sports market events.