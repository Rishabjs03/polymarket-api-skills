# Data API Reference

Base: `https://data-api.polymarket.com` | Auth: None

All endpoints are public — works for any wallet address.
Great for building dashboards without requiring users to connect wallets.

---

## GET /positions

Current and historical positions for a user.

**Params:**
```
user            wallet address (required)
market          conditionId filter
sizeClosedMin   float — include positions where at least this much has been sold/redeemed
                set to 0.01 to include closed/resolved positions
limit, offset   pagination
```

**Response fields per position:**
```
market          conditionId
asset           token_id
outcome         "Yes" | "No"
size            current open shares
avgPrice        average entry price
currentValue    current USDC value at mid price
initialValue    USDC value when position was opened
cashPnl         realized + unrealized P&L in USDC
percentPnl      P&L as percentage
curPrice        current mid price
startDate       when position was opened
title           market question text (convenience field)
endDate         market end date
conditionId     matches market field
```

---

## GET /value

Total portfolio value for a user.
**Params:** `user`
**Response:** `{"value": "1234.56"}` — sum of all open position values in USDC

---

## GET /activity

Trade history and platform activity.

**Params:** `user`, `market` (conditionId), `type`, `limit`, `offset`
**Type filter values:** `trade`, `split`, `merge`, `redeem`, `reward`, `conversion`, `maker_rebate`, `yield`

**Response fields per activity record:**
```
id
type            activity type
market          conditionId
asset           token_id
outcome         "Yes" | "No"
price
size
side            "BUY" | "SELL"
timestamp       Unix timestamp
transactionHash Polygon tx hash
proxyWallet
```

---

## GET /trades

Structured trade records.
**Params:** `user`, `market` (conditionId), `maker_address`, `taker_address`, `limit`, `offset`

---

## GET /open-interest

Open interest for a market.
**Params:** `market` (conditionId)
**Response:** `{"openInterest": "50000.00"}`

---

## GET /positions (for a market — all users)

Get all positions held by all users in a specific market.
**Params:** `market` (conditionId)
Useful for building top holder leaderboards per market.

---

## GET /holders

Top token holders for a market outcome.
**Params:** `market` (conditionId), `outcome` (`Yes`|`No`), `limit`

---

## GET /live-volume

Live trading volume for an event.
**Params:** `event_id`

---

## GET /leaderboard

Trader leaderboard rankings.

**Params:**
```
window      "daily" | "weekly" | "monthly" | "alltime"
limit, offset
```

**Response fields per entry:**
```
proxyWallet     wallet address
name, username  display info
profileImage
pnl             total profit/loss USDC
volume          total traded volume USDC
rank
```

---

## GET /markets-traded

Total number of markets a user has traded.
**Params:** `user`
**Response:** `{"marketsTraded": 47}`

---

## GET /snapshot

Download full accounting history as ZIP of CSVs.
**Params:** `user`
Good for tax reporting and complete trade history export.

---

## Builder Endpoints

### GET /builders/leaderboard
Aggregated builder leaderboard ranked by attributed volume.

### GET /builders/volume
Daily volume timeseries for a builder.
**Params:** `builder` (wallet address), `startDate`, `endDate`
**Response:** `[{"date": "2024-01-01", "volume": "12345.67"}, ...]`