# Frontend Guide

This guide is for building a user interface on The Gavel Protocol — your own frontend, an embedded widget, or an alternative to the reference app at thegavel.io. A Gavel frontend draws on two sources, and the main job of this guide is to show how they fit together:

- **The contracts** — for user actions (borrow, lend, trade) and live on-chain state. Covered in depth in the [Integration Guide](integration.md).
- **The public API** — for the computed market data (yield curve, credit surface, indicators, auction history) you'll want to display. Covered in [Aletheia API](../data/aletheia-api.md).

The reference frontend is a React app deployed on Vercel; nothing here is framework-specific, and any stack works.

## Two layers, two jobs

```
        ┌──────────────────────────────────────────────┐
        │                 your frontend                 │
        ├───────────────────────┬──────────────────────┤
        │   action layer        │   data layer         │
        │   wallet + contracts  │   public API (read)  │
        │   (writes + state)    │   (curve, indicators)│
        └───────────┬───────────┴──────────┬───────────┘
                    ▼                       ▼
            Arbitrum One RPC         api.thegavel.io
            (LoanProtocol, …)        (yield-curve, …)
```

Keep them separate. The action layer needs a wallet and signs transactions; the data layer is read-only HTTP and needs no wallet. Most of your screen is data-layer (charts, tables, rates); the action layer activates when a user borrows, bids, or trades.

## The data layer

A thin fetch client over the API is all you need. This mirrors the reference app's own client:

```js
const API_BASE = import.meta.env.VITE_API_BASE_URL || "https://api.thegavel.io/v1";

async function api(path, params = {}) {
  const url = new URL(`${API_BASE}${path}`);
  for (const [k, v] of Object.entries(params)) {
    if (v != null) url.searchParams.set(k, String(v));
  }
  const res = await fetch(url, { headers: { Accept: "application/json" } });
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new Error(body?.error?.message || `API ${path}: ${res.status}`);
  }
  return res.json();
}

// e.g. the yield curve for a chart, the latest auctions for a table
const curve    = await api("/yield-curve", { pair: "WBTC/USDC", points: true });
const auctions = await api("/auctions");
```

The full endpoint list is in [Aletheia API](../data/aletheia-api.md). The API is public and open today (a tiered model is planned), so the data layer needs no key. For a dashboard, fetch the snapshots you need in parallel and render each panel as its result arrives, isolating failures so one slow endpoint doesn't blank the page.

## The action layer

Connect a wallet, instantiate the contracts, and drive the flows from the [Integration Guide](integration.md). With ethers v6 and an injected provider:

```js
import { ethers } from "ethers";

const provider = new ethers.BrowserProvider(window.ethereum);
await provider.send("eth_requestAccounts", []);
const signer = await provider.getSigner();

// Ensure the wallet is on Arbitrum One (chainId 42161)
await provider.send("wallet_switchEthereumChain", [{ chainId: "0xa4b1" }]);

const loanProtocol = new ethers.Contract(LOAN_PROTOCOL_ADDR, LOAN_PROTOCOL_ABI, signer);
```

From there the borrow/lend/trade flows are exactly as documented in the integration guide. Three frontend-specific points:

- **Read with the provider, write with the signer.** Use cheap `view` calls (`getAuction`, `getLoan`, `canFinalize`, `isInGracePeriod`) to render state and gate buttons; only request a signature when the user acts.
- **Surface typed errors as readable messages.** Decode custom errors (`loanProtocol.interface.parseError(...)`) and map them — `BidTooHigh` → "Your bid is above the current best," `MarketplaceFrozen` → "Trading is paused near maturity." See the integration guide's error section.
- **Handle the approval steps in the UI.** ERC-20 `approve` before deposit/bid/repay, and — if you route through a curated path — the one-time operator approval. Show these as distinct steps so users understand the two-transaction pattern.

## Curation is your frontend's choice

A frontend decides which tokens, collections, and auctions it surfaces — that *is* curation. You can present only a vetted set (e.g. WBTC collateral, USDC/USDT loans) by filtering what you show and constructing transactions with sensible parameters, while the underlying contracts remain open to anything. On v1 you can also route through the reference `ListingService` for its on-chain whitelist; in v2 curation is purely this frontend-level concern (see the [v2 Roadmap](../vision/v2-roadmap.md)). Either way, your curation is a recommendation in your UI, not a restriction on the protocol.

## Live updates

For a responsive UI, subscribe to contract events for the action layer (new auctions, bids, finalisations, sales) and poll the API for the data layer (curve and indicators update on their own cadence). The event list and indexing patterns are in [On-Chain Data](../data/on-chain-data.md).

```js
loanProtocol.on(loanProtocol.filters.BidPlaced(), (auctionId) => {
  // refresh the auction row
});
```

## Configuration and networks

Drive everything from environment variables so one build targets mainnet or testnet:

| Variable | Mainnet | Testnet |
|---|---|---|
| `VITE_API_BASE_URL` | `https://api.thegavel.io/v1` | `https://api-testnet.thegavel.io/v1` |
| `VITE_NETWORK` | `mainnet` (Arbitrum One) | `testnet` (Arbitrum Sepolia) |
| contract addresses | from [deployed contracts](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md) (Arbitrum One) | Arbitrum Sepolia addresses |

Keep the contract addresses and the API base in sync with the network — pointing the action layer at one network and the data layer at another is a common and confusing bug.

## Putting it together

A minimal page is: fetch the yield curve and recent auctions from the API on load and render them; offer a "Connect wallet" button that activates the action layer; and when a user borrows or bids, run the integration-guide flow, showing the approval and confirmation steps. Start read-only (data layer only — it needs no wallet), then layer the actions on top.

## Next

- [Integration Guide](integration.md) — the contract flows your action layer drives.
- [Aletheia API](../data/aletheia-api.md) — every endpoint your data layer can read.
- [On-Chain Data](../data/on-chain-data.md) — events for live updates.
