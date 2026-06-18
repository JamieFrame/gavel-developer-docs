# On-Chain Data

Everything the protocol knows is on-chain and public. You can read auctions, loans, bids, and positions straight from the contracts with nothing but an RPC endpoint — no API, no key, no permission. This is the most fundamental data path, and it's free.

The [Aletheia API](aletheia-api.md) sits on top of this raw data and adds computed products (the fitted yield curve, the credit surface, indicators). Use the API when you want those ready-made; read on-chain directly when you want the source data or want to compute your own views.

## Reading state directly

Every `view` function in the [Contract Reference](contracts-reference.md) is a free call. Point an RPC provider at the deployed [contract addresses](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md) and read:

```js
import { ethers } from "ethers";
const provider = new ethers.JsonRpcProvider(ARBITRUM_RPC_URL);
const loanProtocol = new ethers.Contract(LOAN_PROTOCOL_ADDR, LOAN_PROTOCOL_ABI, provider);

// Walk auctions/loans by nonce
const nextId = await loanProtocol.loanNonce();
for (let id = 0n; id < nextId; id++) {
  const auction = await loanProtocol.getAuction(id);   // Auction struct
  const loan    = await loanProtocol.getLoan(id);      // Loan struct (if finalised)
  // ... your own aggregation
}

// Or read a single position's owner / state
const lenderOwner = await loanProtocol.getLenderPositionOwner(loanId);
```

Useful read functions: `getAuction`, `getLoan`, `getCurrentBid`, `getBidCount`, `getMarketplaceListing`, `isPositionListed`, `getCollateralBalance`, `getPendingRefund`, `loanNonce`, plus the timing/status helpers (`canFinalize`, `isInGracePeriod`, `canClaimCollateral`). The NFT stack mirrors these on `NFTLoanProtocol`.

## Indexing via events

For a historical record or a live feed, index the contract events rather than polling. The lifecycle events (full list in the [Contract Reference](contracts-reference.md)) are emitted for every state change:

- **Auctions:** `AuctionCreated`, `BidPlaced`, `AuctionFinalized`, `AuctionCancelled`, `AuctionExpiredNoBids`, `AuctionExpiredNotFinalized`.
- **Loans:** `LoanRepaid`, `LoanDefaulted`, `LoanDefaultMarked`.
- **Positions & marketplace:** `PositionListed`, `PositionSold`, `MarketplaceOfferMade`, `MarketplaceOfferAccepted`, `BorrowerPositionTransferred`, `LenderPositionTransferred`.

```js
// Backfill historical auctions, then subscribe to new ones
const created = await loanProtocol.queryFilter(
  loanProtocol.filters.AuctionCreated(), fromBlock, "latest"
);
loanProtocol.on(loanProtocol.filters.AuctionCreated(), (auctionId /* , ... */) => {
  // index new auctions in real time
});
```

This is enough to build your own subgraph, dashboard, or analytics from first principles — the same source the Aletheia API is built from.

## Programmatic access for AI agents (public MCP)

The protocol also exposes its data through a public **MCP** (Model Context Protocol) server, so AI agents can read protocol state through tools rather than raw RPC:

```
https://mcp.thegavel.io
```

It is **read-only**. The available tools cover the protocol's market data and reference state — for example, retrieving the yield curve, fetching protocol reference data, finding auctions matching criteria, and listing the available on-chain indicators — alongside a few helper tools (wallet readiness checks, compatible wallet options, fiat on-ramp suggestions). Connect any MCP-capable client to that URL to discover the current tool set and schemas.

## Which path to use

- **Direct RPC reads / events** — source data, fully permissionless, you compute your own views. Free.
- **Public MCP** — the same protocol data exposed as read tools for agents. Free, read-only.
- **[Aletheia API](aletheia-api.md)** — computed products (fitted curve, credit surface, indicators) ready to consume, so you don't index or compute them yourself.
