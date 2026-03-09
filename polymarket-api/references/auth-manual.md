# Manual Authentication Reference

For implementing auth without the SDK. Prefer the SDK whenever possible.
All signing details verified from Polymarket's py-clob-client source code.

---

## L1 — EIP-712 Wallet Signing

Used to generate L2 credentials. Called once per wallet.

**Endpoint:** `POST https://clob.polymarket.com/auth/api-key`

**EIP-712 domain and struct (from py_clob_client/signing/eip712.py):**
```json
{
  "domain": {
    "name": "ClobAuthDomain",
    "version": "1",
    "chainId": 137
  },
  "types": {
    "ClobAuth": [
      {"name": "address",   "type": "address"},
      {"name": "timestamp", "type": "string"},
      {"name": "nonce",     "type": "uint256"},
      {"name": "message",   "type": "string"}
    ]
  },
  "message": {
    "address":   "0xYourAddress",
    "timestamp": "1700000000",
    "nonce":     0,
    "message":   "This message attests that I control the given wallet"
  }
}
```

Note: `timestamp` in the EIP-712 struct is a **string** representation of Unix seconds.

**L1 request headers:**
```
POLY_ADDRESS      0xYourAddress
POLY_SIGNATURE    <EIP-712 signature from wallet>
POLY_TIMESTAMP    1700000000    (Unix SECONDS — not milliseconds)
POLY_NONCE        0
```

**Response:**
```json
{
  "apiKey":     "550e8400-e29b-41d4-a716-446655440000",
  "secret":     "base64EncodedSecretString==",
  "passphrase": "randomPassphraseString"
}
```
Store all three. Secret and passphrase are shown only once.

---

## L2 — HMAC-SHA256 Signing

Used for every authenticated trading request.

**Signature construction:**
```
message   =  timestamp + method + requestPath + body
signature =  HMAC-SHA256(base64Decode(secret), message.encode("utf-8"))
encoded   =  base64Encode(signature)
```

Where:
- `timestamp` = Unix **milliseconds** as a string (e.g. `"1700000000000"`) — NOT seconds
- `method` = uppercase HTTP method: `GET`, `POST`, `DELETE`
- `requestPath` = full path including query string: `/orders?status=live`
- `body` = JSON-serialized request body. Use `""` for GET requests.

**Python implementation:**
```python
import hmac, hashlib, base64, time

def make_l2_headers(method: str, path: str, body: str,
                    api_key: str, secret: str, passphrase: str,
                    address: str) -> dict:
    ts = str(int(time.time() * 1000))          # milliseconds
    message = ts + method.upper() + path + body
    secret_bytes = base64.b64decode(secret)
    sig = hmac.new(secret_bytes, message.encode("utf-8"), hashlib.sha256).digest()
    sig_b64 = base64.b64encode(sig).decode("utf-8")
    return {
        "POLY_ADDRESS":    address,
        "POLY_SIGNATURE":  sig_b64,
        "POLY_TIMESTAMP":  ts,
        "POLY_API_KEY":    api_key,
        "POLY_PASSPHRASE": passphrase,
    }
```

**L2 request headers:**
```
POLY_ADDRESS      0xYourAddress
POLY_API_KEY      your-api-key-uuid
POLY_PASSPHRASE   your-passphrase
POLY_SIGNATURE    base64(HMAC-SHA256(base64Decode(secret), ts+method+path+body))
POLY_TIMESTAMP    1700000000000   (Unix MILLISECONDS)
```

---

## Managing API Keys

```
GET    /auth/api-keys        list all existing L2 keys (L1 auth required)
POST   /auth/api-key         create a new L2 key (L1 auth required)
DELETE /auth/api-key         revoke a key, body: {"apiKey": "..."}
```

---

## Common Auth Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Unauthorized / Invalid api key` | Stale or wrong L2 creds | Re-run `create_or_derive_api_creds()` |
| `Invalid signature` | Timestamp in seconds not ms, or wrong secret | Check timestamp unit |
| `Nonce error` | L1 nonce reused (rare) | Increment nonce or use 0 every time |
| `403 / geoblock` | IP from restricted region | Not an auth error — IP is blocked |
| `not enough balance / allowance` | EOA wallet missing token approval | Run allowance setup once |
| Silent auth failure / orders rejected | Wrong `signature_type` for wallet | Match sig type to wallet type |

---

## Builder Authentication

Builder credentials attribute orders to your app in the Builders Program.
Separate from user trading credentials — do not substitute one for the other.

**TypeScript:**
```typescript
import { BuilderConfig, BuilderApiKeyCreds } from "@polymarket/builder-signing-sdk";

const builderCreds: BuilderApiKeyCreds = {
  key:        process.env.POLY_BUILDER_API_KEY!,
  secret:     process.env.POLY_BUILDER_SECRET!,
  passphrase: process.env.POLY_BUILDER_PASSPHRASE!,
};
const builderConfig: BuilderConfig = { localBuilderCreds: builderCreds };

// Pass as last argument to ClobClient
const client = new ClobClient(
    host, chainId, signer, userCreds, sigType, funder,
    undefined, false, builderConfig
);
```

**Python:**
```python
# Builder signing is handled via SDK internals when POLY_BUILDER_* env vars are set
# See py-builder-signing-sdk or Polymarket builder docs for details
```

Get builder API keys at: `polymarket.com/settings?tab=builder`