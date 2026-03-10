# Polymarket API Skills

This repository provides an AI agent skill for interacting with the Polymarket API. It is designed to act as a verified, production-grade reference for AI assistants, allowing them to accurately understand and write code for Polymarket prediction markets.

## Installation

To add this skill to your AI coding agent or environment, run the following command:

```bash
npx skills add Rishabjs03/polymarket-api-skills
```

## How It Works

This skill injects deep context into your AI agent regarding the Polymarket ecosystem. By referencing this skill, the AI learns exactly how to interact with Polymarket's three main APIs, helping you avoid common bugs and undocumented gotchas.

### The Three APIs

The skill teaches your AI to pick the right API for the job:

1. **Gamma API (`https://gamma-api.polymarket.com`)**
   **Purpose**: Market discovery.
   Use this for searching events, finding markets, reading questions, and getting `token_id`s. No authentication is required.

2. **CLOB API (`https://clob.polymarket.com`)**
   **Purpose**: Trading and live prices.
   Use this for fetching the orderbook, getting best bid/ask, and placing/canceling orders. Read operations are public, while trading requires L2 authentication.

3. **Data API (`https://data-api.polymarket.com`)**
   **Purpose**: User analytics.
   Use this for analyzing user positions, trade history, portfolio value, and leaderboards. No authentication is required.

### Key Capabilities the AI Leans

When this skill is installed, your agent will automatically know:

- **Authentication Quirks**: It knows how to manage L1 (EIP-712) and L2 (HMAC-SHA256) auth. It understands the critical differences between signing for an EOA wallet, an Email/Magic wallet, or a Proxy/Safe wallet.
- **Data Anomalies**: It knows exactly how to parse Polymarket's unique data structures (such as `outcomePrices` being stringified JSON arrays).
- **SDK Integrations**: It contains verified code snippets to help initialize and use the official Python (`py-clob-client`) and TypeScript (`@polymarket/clob-client`) SDKs efficiently.
- **Trading Workflows**: Complete workflows for checking prices, placing different order types (Limit, Market/FOK, GTC), and streaming real-time data via specific WebSockets endpoints.
- **Smart Contract Details**: It has verified Polygon Mainnet contract addresses (USDC, CTF, Exchange) and understands when and how to perform crucial token approvals before starting to trade.
- **Rate Limit Best Practices**: It is aware of Polymarket's Cloudflare rate constraints and will write robust code using WebSockets and batch queries rather than naive polling.

Save time and prevent AI hallucinations by using this skill as your agent's single source of truth for all Polymarket integrations.
