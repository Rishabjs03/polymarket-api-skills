---
name: polymarket-api
description: >
  Verified, production-grade reference for building on the Polymarket prediction markets API.
  Use this skill for ANY task touching Polymarket — fetching markets, reading live prices,
  placing or cancelling orders, streaming WebSocket data, setting up wallets, querying positions,
  understanding the on-chain settlement layer, or debugging auth failures.
  Also triggers for: gamma-api.polymarket.com, clob.polymarket.com, data-api.polymarket.com,
  py-clob-client, @polymarket/clob-client, CLOB trading, CTF tokens, prediction market bots,
  trading scripts, and market data dashboards. Always read this skill before writing any
  Polymarket code — it contains verified contract addresses, real rate limits, and SDK
  gotchas sourced directly from Polymarket's GitHub and official docs.
---

Polymarket is the world's largest prediction market, running on Polygon (chain ID 137).
Markets are binary — YES/NO outcome tokens priced 0.0–1.0 USDC representing implied probability.
Collateral is USDC.e on Polygon PoS. All trading is off-chain matched, on-chain settled via
CTF (Conditional Token Framework) smart contracts.

---

## The Three APIs — Pick the Right One First

| API | Base URL | Auth | What it does |
|-----|----------|------|--------------|
| **Gamma** | `https://gamma-api.polymarket.com` | None | Market discovery — events, markets, tags, search, comments, sports, profiles |
| **CLOB** | `https://clob.polymarket.com` | None for reads; L2 for trading | Orderbook, live prices, order placement and cancellation |
| **Data** | `https://data-api.polymarket.com` | None | User positions, trade history, portfolio value, leaderboards |

**The rule**: Gamma to discover. CLOB to trade and read live prices. Data to analyze user activity.
The Gamma and Data APIs are 100% public. CLOB read endpoints (prices, orderbook) are also public.
Only order placement, cancellation, and trade history require L2 auth.

---

## Data Model — Know This Before Writing Anything

```
Series
  └── Event  (e.g. "2024 US Presidential Election")
        └── Market  (e.g. "Will Trump win the 2024 election?")
              ├── YES token  →  clobTokenIds[0]   ← use this for all CLOB calls
              └── NO token   →  clobTokenIds[1]
```

The `token_id` (from `clobTokenIds`) is the primary key for everything in the CLOB — prices,
orderbooks, order placement, price history. Get it from the Gamma API first.

**Critical fields on a Gamma market object:**
```
id                  numeric Gamma market ID
question            human-readable question
conditionId         on-chain CTF condition ID (hex) — use for Data API calls
clobTokenIds        ["YES_token_id", "NO_token_id"] — use for all CLOB calls
outcomePrices       ⚠️ STRINGIFIED JSON — must parse: json.loads(market["outcomePrices"])
active / closed     always filter active=true, closed=false for live markets
negRisk             bool — if true, order construction differs; pass neg_risk=True in SDK
```

---

## SDK Installation

### Python (py-clob-client v0.34.6+)
```bash
pip install py-clob-client   # requires Python 3.9+
```

### TypeScript (@polymarket/clob-client v5.2.1+)
```bash
npm install @polymarket/clob-client ethers
# Works with both ethers v5 (@ethersproject/wallet) and viem
```

---

## Authentication — Two Levels

**L1 (private key / EIP-712):** Used once to generate L2 credentials. The SDK handles this automatically via `create_or_derive_api_creds()` / `createOrDeriveApiKey()`. Never call it in a hot path — run once and persist the credentials.

**L2 (HMAC-SHA256):** Used for every trading request. Three values: `apiKey`, `secret`, `passphrase`. Always use the SDK for signing — manual HMAC is error-prone.

**Signature types — must match your wallet or auth silently fails:**

| Value | Wallet type |
|-------|-------------|
| `0` | Standard EOA: MetaMask, hardware wallet, any wallet where you control the private key directly |
| `1` | Email/Magic wallet (delegated signing via Magic Link) |
| `2` | Browser wallet proxy / Gnosis Safe multisig |

**Funder address:** Only required for signature types 1 and 2. It is the address that actually holds USDC on Polymarket, which differs from the signing key. For type 0 EOA, the signing key and the funder are the same — omit `funder`.

**Getting your private key from Polymarket:** Log in → Cash → three dots menu → "Export Private Key" → remove the `0x` prefix before using.

---

## Client Setup

### Python

```python
from py_clob_client.client import ClobClient

HOST = "https://clob.polymarket.com"
CHAIN_ID = 137

# Read-only — no credentials needed
client = ClobClient(HOST, chain_id=CHAIN_ID)

# EOA/MetaMask wallet — full trading
client = ClobClient(
    HOST,
    key=os.getenv("PRIVATE_KEY"),      # without 0x prefix
    chain_id=CHAIN_ID,
    signature_type=0,                   # EOA
)
client.set_api_creds(client.create_or_derive_api_creds())

# Email/Magic wallet — full trading
client = ClobClient(
    HOST,
    key=os.getenv("PRIVATE_KEY"),
    chain_id=CHAIN_ID,
    signature_type=1,                   # Magic/email
    funder=os.getenv("FUNDER_ADDRESS"), # the address holding USDC
)
client.set_api_creds(client.create_or_derive_api_creds())
```

### TypeScript

```typescript
import { ClobClient, OrderType, Side } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const HOST = "https://clob.polymarket.com";
const CHAIN_ID = 137;

// Read-only
const client = new ClobClient(HOST, CHAIN_ID);

// EOA full trading
const signer = new Wallet(process.env.PRIVATE_KEY!);
const tempClient = new ClobClient(HOST, CHAIN_ID, signer);
const creds = await tempClient.createOrDeriveApiKey();
const client = new ClobClient(HOST, CHAIN_ID, signer, creds, 0); // 0=EOA

// Viem alternative (works with clob-client v5+)
import { createWalletClient, http } from "viem";
import { polygon } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(`0x${process.env.PRIVATE_KEY}`);
const walletClient = createWalletClient({ account, chain: polygon, transport: http() });
const client = new ClobClient(HOST, CHAIN_ID, walletClient);
```

---

## Core Workflows

### 1. Find a market and get its token_id
```python
import requests, json

markets = requests.get("https://gamma-api.polymarket.com/markets", params={
    "active": True, "closed": False, "q": "bitcoin", "limit": 20,
}).json()

for m in markets:
    token_ids  = m["clobTokenIds"]              # [YES_id, NO_id]
    prices     = json.loads(m["outcomePrices"]) # ["0.65", "0.35"] — MUST parse
    print(m["question"], prices)
```

### 2. Get live prices (no auth)
```python
token_id = "71321045679252212594626385532706912750332728571942532289631379312455583992563"

mid   = client.get_midpoint(token_id)           # float — best proxy for probability
bid   = client.get_price(token_id, side="SELL") # best bid
ask   = client.get_price(token_id, side="BUY")  # best ask
book  = client.get_order_book(token_id)         # book.bids / book.asks → [{price, size}]

# Multiple books in one call
from py_clob_client.clob_types import BookParams
books = client.get_order_books([BookParams(token_id=t) for t in token_ids])

# Price history
history = requests.get("https://clob.polymarket.com/prices-history", params={
    "market": token_id,
    "interval": "1d",   # 1m | 5m | 1h | 6h | 1d | 1w
    "fidelity": 100,
}).json()["history"]    # [{"t": unix_ts, "p": 0.65}, ...]
```

### 3. Place a limit order (L2 auth required)
```python
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY, SELL

order  = OrderArgs(token_id=token_id, price=0.65, size=10.0, side=BUY)
signed = client.create_order(order)
resp   = client.post_order(signed, OrderType.GTC)
# resp: {"success": True, "orderID": "0x...", "status": "live"|"matched"|"unmatched"}
```

### 4. Place a market order / FOK (L2 auth required)
```python
from py_clob_client.clob_types import MarketOrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY

mo     = MarketOrderArgs(token_id=token_id, amount=25.0, side=BUY)
signed = client.create_market_order(mo)
resp   = client.post_order(signed, OrderType.FOK)
```

TypeScript convenience method (create + post in one call):
```typescript
const resp = await client.createAndPostOrder(
    { tokenID: token_id, price: 0.65, side: Side.BUY, size: 10 },
    { tickSize: "0.01", negRisk: false },
    OrderType.GTC
);
```

### 5. Manage orders
```python
from py_clob_client.clob_types import OpenOrderParams

orders = client.get_orders(OpenOrderParams())              # all open orders
client.cancel(order_id="0x...")                            # single
client.cancel_all()                                        # all open orders
client.cancel_market_orders(asset_id=token_id)            # all in a market
```

### 6. User positions and portfolio (no auth — works for any public address)
```python
wallet = "0xYourWalletAddress"

positions  = requests.get("https://data-api.polymarket.com/positions",
                          params={"user": wallet}).json()
portfolio  = requests.get("https://data-api.polymarket.com/value",
                          params={"user": wallet}).json()  # {"value": "1234.56"}
activity   = requests.get("https://data-api.polymarket.com/activity",
                          params={"user": wallet}).json()
```

### 7. Real-time prices via WebSocket (no auth)
```python
import asyncio, websockets, json

async def stream_market(token_ids: list[str]):
    uri = "wss://ws-subscriptions-clob.polymarket.com/ws/market"
    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps({
            "auth": {},
            "type": "Market",
            "markets": [],
            "assets_ids": token_ids
        }))
        async for msg in ws:
            event = json.loads(msg)
            # event["type"]: "book" | "price_change" | "last_trade_price"

asyncio.run(stream_market([token_id]))
```

For your own order/trade events, use `wss://ws-subscriptions-clob.polymarket.com/ws/user`
with L2 credentials in the `auth` field of the subscribe message.

---

## EOA Wallet Token Allowances — Required Before First Trade

Email/Magic wallets: no action needed — allowances are set automatically.

MetaMask / EOA wallets: must approve two tokens for three contracts once per wallet.
Without this, orders fail with "not enough balance / allowance". No clear error message.

**USDC contract:** `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`
**Conditional Tokens (CTF):** `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`

Approve both tokens for all three exchange contracts:
```
0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E  ← CTF Exchange (main, binary markets)
0xC5d563A36AE78145C45a50134d48A1215220f80a  ← NegRisk CTF Exchange
0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296  ← NegRisk Adapter
```

Full working code: https://gist.github.com/poly-rodr/44313920481de58d5a3f6d1f8226bd5e
Run once. After that, you can trade indefinitely without repeating this step.

---

## Settlement / Redeeming Winning Positions

After a market resolves, winning tokens can be redeemed for USDC by calling `redeemPositions`
on the CTF contract. The function burns winning tokens and returns collateral per the payout vector.

**The major gotcha**: most Polymarket accounts use a Safe (proxy multisig) wallet internally.
For these accounts, `redeemPositions` cannot be called directly — the Safe must execute it via
`execTransaction`, which requires managing Safe signatures. The py-clob-client SDK has no `redeem`
method as of v0.34.6, and there is no official CLOB API endpoint for redemption.

**Practical options:**
- Sell your position on the open market before resolution (cleanest, no redemption needed)
- For EOA wallets: call `redeemPositions` directly on the CTF contract via web3.py / ethers
- For proxy wallet accounts: redemption is handled automatically by Polymarket's UI over time;
  no programmatic path is currently available via the official SDK

CTF contract on Polygon: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`
Function signature: `redeemPositions(collateralToken, parentCollectionId, conditionId, indexSets)`
- `collateralToken`: USDC address
- `parentCollectionId`: `bytes32(0)` (null for Polymarket)
- `conditionId`: from `market.conditionId`
- `indexSets`: `[1, 2]` for binary markets (YES=1, NO=2)

---

## On-Chain Contract Addresses (Polygon Mainnet)

These are verified from Polymarket's official GitHub:

```
USDC (collateral):           0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174
Conditional Tokens (CTF):   0x4D97DCd97eC945f40cF65F87097ACe5EA0476045
CTF Exchange (binary):      0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E
NegRisk CTF Exchange:       0xC5d563A36AE78145C45a50134d48A1215220f80a
NegRisk Adapter:            0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296
```

---

## Rate Limits

Enforced via Cloudflare throttling — requests are delayed/queued, not hard-rejected.
Returns HTTP 429 when exceeded.

| Endpoint | Limit |
|----------|-------|
| `POST /order` (order placement) | 3,000 per 10 minutes per API key |
| `GET /books` (batch orderbooks) | 300 per 10 seconds |
| Public read endpoints (general) | ~60 requests/minute |
| WebSocket connections | Effectively unlimited |

Best practices: use WebSocket instead of polling for live prices; use `/books` (batch) over
individual `/book` calls; implement exponential backoff on 429 responses.

---

## Order Types

| Type | Behavior | Use when |
|------|----------|----------|
| `GTC` | Rests on book until filled or cancelled | Limit orders, market making |
| `FOK` | Fill entire order now or cancel completely | Market orders, certainty required |
| `FAK` | Fill available now, cancel remainder | Partial fills acceptable |
| `GTD` | Like GTC but with an expiry timestamp | Time-bounded exposure |

Valid tick sizes: `"0.1"`, `"0.01"`, `"0.001"`, `"0.0001"`. Fetch per-market — it changes dynamically.
The Python SDK resolves tick size automatically. In TypeScript, pass `{ tickSize, negRisk }` to
`createAndPostOrder()` — get both values from the Gamma market object.

---

## Critical Gotchas — Verified from GitHub Issues and Source Code

**`outcomePrices` is a stringified JSON array.** Accessing it raw gives a string like `'["0.65","0.35"]'`. Always call `json.loads(market["outcomePrices"])` in Python or `JSON.parse(market.outcomePrices)` in JS.

**EOA wallets need token allowances before any trade.** Email/Magic wallets are handled automatically. EOA users will hit "not enough allowance" on their very first order. See allowance section above.

**Signature type must exactly match your wallet type.** Using type 0 with a Magic wallet, or type 1 with an EOA, will cause orders to fail or auth to silently reject. Confirm before building.

**`create_or_derive_api_creds()` should be called once and persisted.** Calling it on every run works but is slow and wasteful. Store the `apiKey`, `secret`, and `passphrase` in environment variables after the first run.

**The `funder` address is your Polymarket profile address, not your signing key.** For proxy wallets (type 1/2), the funder is the address you fund on Polymarket and where your USDC lives. The signing key is separate. Confusing these causes "insufficient balance" errors even with funds present.

**`negRisk` markets need special handling.** Verify `market.negRisk` before placing orders. In TypeScript's `createAndPostOrder`, pass `{ negRisk: true }`. In Python, `create_order` accepts a `neg_risk` kwarg. Skipping this on a negRisk market produces incorrect order construction.

**Redemption is not supported programmatically for most users.** As of py-clob-client v0.34.6, there is no `redeem()` method. The CLOB API has no redeem endpoint. For proxy wallet users (most users), on-chain redemption requires Safe transaction infrastructure. Sell before resolution if you need liquidity.

**`POLY_TIMESTAMP` is milliseconds for L2, seconds for L1.** The SDK handles this. Manual implementations constantly get this wrong, causing `Invalid signature` errors.

**Geographic restrictions are IP-level, not credential errors.** A 403 with no auth error message is likely a geoblock. US IPs and certain other regions are blocked on the international platform. Polymarket US (CFTC-regulated) has a separate API endpoint.

---

## Environment Variables

```env
# Wallet
PRIVATE_KEY=your_private_key_without_0x_prefix
FUNDER_ADDRESS=0xYourPolymarketProfileAddress   # only for signature types 1 and 2

# L2 credentials — generate once via create_or_derive_api_creds(), then store these
API_KEY=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
SECRET=base64EncodedSecretString==
PASSPHRASE=yourPassphraseString

# Builder Program (optional — separate from trading credentials)
POLY_BUILDER_API_KEY=...
POLY_BUILDER_SECRET=...
POLY_BUILDER_PASSPHRASE=...
```

---

## Reference Files

Read these when you need full endpoint/parameter schemas:

- `references/gamma-api.md` — Every Gamma API endpoint with all params and response fields
- `references/clob-api.md` — Every CLOB REST and WebSocket endpoint
- `references/data-api.md` — Every Data API endpoint
- `references/auth-manual.md` — Manual L1/L2 HMAC signing without SDK + builder auth
- `references/sdk-methods.md` — Full method reference for Python and TypeScript SDKs

---

The four most common Polymarket bugs: wrong API (Gamma vs CLOB vs Data), missing token allowances
(EOA wallets), unparsed `outcomePrices` string, and wrong `signature_type` for the wallet.
Catch these before debugging anything else.