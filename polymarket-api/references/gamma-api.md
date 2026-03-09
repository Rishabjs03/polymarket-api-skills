# Gamma API Reference

Base: `https://gamma-api.polymarket.com` | Auth: None

---

## GET /markets

List, search, and filter markets. This is the primary discovery endpoint.

**Query params:**
```
active          bool   — filter to active markets
closed          bool   — filter to closed/resolved markets
archived        bool
id              str    — exact numeric market ID
slug            str    — exact market slug
condition_id    str    — on-chain conditionId (hex)
clob_token_id   str    — filter by outcome token ID
q               str    — full-text search across question/description
tag_id          int    — filter by tag ID
tag_slug        str    — filter by tag slug
limit           int    — max 100, default 100
offset          int    — pagination offset
order           str    — sort field: "volume", "liquidity", "startDate", "endDate"
ascending       bool   — sort direction
related_tags    bool   — include related tag objects in response
```

**Full response fields per market:**
```
id                  numeric Gamma market ID
question            human-readable question text
slug                URL slug (e.g. "will-trump-win-2024")
description         full market description
conditionId         on-chain CTF condition ID (hex string)
clobTokenIds        [YES_token_id, NO_token_id]
active              bool
closed              bool
archived            bool
startDate           ISO timestamp
endDate             ISO timestamp
creationDate        ISO timestamp
volume              total lifetime volume, string USDC
volumeNum           float version of volume
volume24hr          24-hour volume float
liquidity           current liquidity string USDC
liquidityNum        float version of liquidity
outcomes            ["Yes", "No"]
outcomePrices       ⚠️ STRINGIFIED JSON — '["0.65", "0.35"]' — must be parsed
bestBid             current best bid price string
bestAsk             current best ask price string
lastTradePrice      last matched price string
spread              bid-ask spread string
negRisk             bool — true for negRisk markets (multi-outcome)
groupItemTitle      for grouped markets
groupItemThreshold
resolvedBy          resolution source name
resolutionSource    resolution URL or description
image               image URL
icon                icon URL
tags                [{id, label, slug}]
events              nested event objects (if queried via /events endpoint)
clobRewards         liquidity reward info object
acceptingOrders     bool
acceptingOrdersTimestamp
```

## GET /markets/{id}
Single market by numeric ID.

## GET /markets?slug={slug}
Market by slug. Returns array — take first element.

## GET /markets/{id}/tags
Tags attached to a specific market.

---

## GET /events

Events are groups of related markets (e.g. an election event containing multiple candidate markets).

**Query params:** same pattern as /markets — `active`, `closed`, `archived`, `slug`, `id`,
`tag_id`, `tag_slug`, `limit`, `offset`, `order`, `ascending`

**Key response fields:**
```
id, slug, title, description
startDate, endDate, creationDate
active, archived, closed
volume, liquidity
tags[]
markets[]               nested market objects
seriesId                parent series ID if grouped
```

## GET /events/{id}
## GET /events?slug={slug}
## GET /events/tags

---

## GET /search

Search markets, events, and profiles in one call.

**Params:**
```
q                   str    required — search query
limit_per_type      int    results per type
page                int
events_tag          str[]  filter events by tag
keep_closed_markets int    0 or 1
sort                str    sort field
ascending           bool
search_tags         bool   include tags in results
search_profiles     bool   include profiles in results
```

**Response:** `{ "markets": [...], "events": [...], "profiles": [...] }`

---

## GET /tags

List all tags.
**Params:** `q` (search), `limit`, `offset`
**Tag object:** `{ id, label, slug, forceShow, publishedAt }`

## GET /tags/{id}
## GET /tags?slug={slug}
## GET /tags/{id}/related — related tags by ID
## GET /tags/{id}/related-tags — tags that relate to this tag

---

## GET /series

Parent groupings of events (e.g. "US Elections" containing multiple election events).
## GET /series/{id}

---

## GET /comments?market_id={id}

Comments on a market.
**Params:** `market_id`, `limit`, `offset`
Comment fields: `id`, `content`, `author` (wallet), `createdAt`, `marketId`

## GET /comments/{id}
## GET /comments?user_address={address}

---

## GET /sports/metadata
## GET /sports/market-types
## GET /sports/teams — params: `sport`, `league`

---

## GET /profiles?wallet_address={address}

Public user profile.
Fields: `name`, `username`, `bio`, `profileImage`, `address`, `twitterHandle`, `website`