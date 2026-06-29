# Aletheia API

The Aletheia API serves the market data derived from The Gavel Protocol — the BTC yield curve, the credit surface, and a family of credit and on-chain indicators — over a simple REST/JSON interface. It is the value-added product layer: a ready-to-consume API so you don't have to index the chain and compute these yourself. If you want the raw protocol data straight from chain instead, see [On-Chain Data](on-chain-data.md).

This page documents how to **consume** the API. It does not describe how the indicators are produced; indicator definitions live on the data product's own indicator pages.

## Base URL

```
Production (Arbitrum One):  https://api.thegavel.io/v1
Testnet:                    https://api-testnet.thegavel.io/v1
```

## Access and rate limits

The API is currently **public and open** — no API key is required to call it. A tiered model is planned: a free tier available to everyone up to a request limit, with an upgraded API key for higher limits. Pricing is not yet finalised. Until then, please be considerate with request volume; once the tiered model is live, this page will document keys and limits.

## Conventions

- All endpoints are `GET` and return JSON.
- Parameters are passed as query strings. Common ones: `pair` (e.g. `WBTC/USDC`), `points` (include fitted curve points), and for `/history` endpoints, time-range/limit parameters.
- Any endpoint with a `…/history` counterpart returns the current value at the base path and a time series at `/history`.
- Errors return a non-2xx HTTP status with a body of the form:

```json
{ "error": { "code": "string", "message": "human-readable detail" } }
```

## Endpoints

### Core

| Endpoint | Returns |
|---|---|
| `GET /health` | Service health. |
| `GET /yield-curve` | The current market-discovered BTC yield curve. Pass `pair` and `points=true` for fitted curve points. |
| `GET /yield-curve/history` | Historical yield-curve snapshots. |
| `GET /credit-surface` | The current credit surface for a `pair`. |
| `GET /credit-surface/at` | The credit surface **as it stood on a historical date** — the aggregate `grid` (same shape as `/credit-surface`) plus a per-loan `loans` dataset, and the resolved `asOf` snapshot. Params: `date` (ISO calendar date, UTC end-of-day; omit for the live surface), `pair`, and `loans=false` to return the grid only. As-of semantics: the protocol state at the last snapshot on or before the requested date. Future dates clamp to now; dates before any data return an honest-empty `200`. Each `loans` row carries `loanId`, `durationDays`, `ltv`, `apr`, `principal`, `collateralAmount`, `lta` (liquidation price = `principal`/`collateralAmount`), `repaymentOwed`, `status`, `maturity`, and the marketplace state `listed`/`relisted`/`listingPrice`. |

### Protocol data

| Endpoint | Returns |
|---|---|
| `GET /auctions` | List of protocol auctions (supports filter params). |
| `GET /auctions/{id}` | A single auction by ID. |
| `GET /stats` | Protocol-wide statistics. |
| `GET /analytics/daily` | Daily analytics series. |
| `GET /protocol-state` | Current protocol state summary. |

### Marketplace (secondary market)

Current-state listings and the full event log for borrower/lender position resales, indexed directly from on-chain marketplace events. Read-only.

| Endpoint | Returns |
|---|---|
| `GET /marketplace/listings` | Current-state listings. Optional filters: `status` (`LISTED`/`UNLISTED`/`SOLD`), `loan_id`, `token_id`, `position_type` (`borrower`/`lender`), `relisted` (`true`/`false`). Pagination: `limit` (default 50, max 200), `offset`. |
| `GET /marketplace/history` | Event log for a position. **One of `loan_id` or `token_id` is required.** Optional: `event_type`, `as_of` (a block number or ISO date — replays the log up to and including that point). Pagination: `limit` (default 100, max 500), `offset`. Events are returned in `(block_number, log_index)` order. |

Prices are returned both raw (on-chain base units, as a string — `asking_price`, `sale_price`, `price`) and pre-formatted (`asking_price_formatted`, etc.) using the payment token's decimals, alongside `payment_token_symbol` and `payment_token_decimals`.

> **Mainnet note.** The secondary marketplace is new on mainnet, so mainnet listings/history are currently **sparse** — these endpoints return well-formed empty results (`200` with empty arrays) there until activity accrues. The testnet base URL has live data.

### Prices and market context

| Endpoint | Returns |
|---|---|
| `GET /prices/btc` · `/prices/btc/history` | BTC price, current and historical. |
| `GET /market/btc` | BTC market context. |
| `GET /macro/current` | Current macro context. |
| `GET /defi-rates/current` | Current comparable DeFi lending rates. |
| `GET /stablecoins/current` | Current stablecoin context. |
| `GET /rates/comparison` | Cross-venue rate comparison. |
| `GET /rates/latest` | Latest rate for a `reference` (e.g. `gavel`, `aave`, `compound`, `treasuries`, `btc_funding`) and `tenor` (e.g. `3m`). |
| `GET /rates/history` | Historical rate series. |
| `GET /benchmark-curves` | Term-structure benchmark curve points. |
| `GET /btc/chain-state` | Bitcoin chain state. |
| `GET /arbitrum/state` | Arbitrum network state. |

### Gavel-native indicators

Each returns the current value; the `/history` variant returns the series. Definitions are on the indicator reference pages.

`GET /tsr` · `/tci` · `/implied-price` · `/regime` · `/cdr` (each with `…/history`).

### Credit-complex indicators

`GET /credit-complex` returns the batched set in one call; `GET /capital-stack` returns the capital-stack view. Individual indicators (with `/history` where applicable): `GET /lci`, `/vrb`, `/lpi`, `/drp`, `/sli`, `/sdr`, `/gls`, `/coc`, `/ccpi`.

### Bitcoin on-chain indicators

`GET /onchain/hrcs` · `/mrys` · `/onchain/rpid` (each with `…/history`). UTXO-derived indicators: `GET /indicators/latest`, `GET /indicators/history`, and `GET /indicators/hodl-waves`.

## Examples

```bash
# Current yield curve with fitted points
curl "https://api.thegavel.io/v1/yield-curve?pair=WBTC/USDC&points=true"

# A single auction
curl "https://api.thegavel.io/v1/auctions/42"

# Latest Gavel 3-month rate
curl "https://api.thegavel.io/v1/rates/latest?reference=gavel&tenor=3m"

# Batched credit-complex indicators
curl "https://api.thegavel.io/v1/credit-complex"

# Current marketplace listings for one loan
curl "https://api.thegavel.io/v1/marketplace/listings?loan_id=308"

# Replay a position's marketplace event log as of a given block
curl "https://api.thegavel.io/v1/marketplace/history?token_id=617&as_of=282399131"
```

Each returns a JSON document; shape varies by endpoint. Handle non-2xx responses by reading `error.code` and `error.message` from the body.

## Notes

- The data is derived from on-chain protocol activity. For the underlying raw data — auctions, loans, and events read directly from the contracts — see [On-Chain Data](on-chain-data.md).
- Endpoint availability evolves as new indicators ship; treat this list as the current surface, and check `/health` and the live responses for what's available.
