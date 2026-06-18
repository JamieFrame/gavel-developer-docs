# Integration Guide

This guide shows how to integrate with The Gavel Protocol in code: the common flows (borrow, lend, trade a position), the operator pattern that lets you build on top without taking custody, and how to read state and handle errors. Read [Architecture](architecture.md) for the model and [Contract Reference](contracts-reference.md) for exact signatures.

Examples use **ethers v6**; the same calls work with viem or any other library. Addresses are in the protocol repo's [deployed contracts](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md); ABIs come from the verified source on Arbiscan or by compiling the contracts in `src/`.

## Setup

```js
import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider(ARBITRUM_RPC_URL);
const signer   = new ethers.Wallet(PRIVATE_KEY, provider);

const loanProtocol = new ethers.Contract(LOAN_PROTOCOL_ADDR, LOAN_PROTOCOL_ABI, signer);
const positionNFT  = new ethers.Contract(POSITION_NFT_ADDR,  POSITION_NFT_ABI,  signer);
const usdc         = new ethers.Contract(USDC_ADDR, ERC20_ABI, signer);
```

A note on decimals: amounts are token base units. USDC and USDT use 6 decimals, WBTC uses 8, most ERC-20s use 18. Use `ethers.parseUnits("100", 6)` for 100 USDC.

Every flow that moves a token starts with an ERC-20 `approve` to the core contract.

---

## Flow 1 — Borrow (direct)

```js
// 1. Approve and deposit collateral (e.g. 0.5 WBTC, 8 decimals)
const collateralAmount = ethers.parseUnits("0.5", 8);
await (await wbtc.approve(loanProtocol.target, collateralAmount)).wait();
await (await loanProtocol.depositCollateral(WBTC_ADDR, collateralAmount)).wait();

// 2. Open an auction: borrow 10,000 USDC, repay at most 10,500, 90-day term, 24h auction
const tx = await loanProtocol.createAuction(
  WBTC_ADDR, collateralAmount,
  USDC_ADDR, ethers.parseUnits("10000", 6),
  ethers.parseUnits("10500", 6),     // maxRepayment (interest cap)
  90n * 24n * 60n * 60n,             // loanDuration (seconds)
  24n * 60n * 60n,                   // auctionDuration (seconds)
  0n                                 // bidStep: 0 = protocol default
);
const receipt = await tx.wait();

// 3. Recover the auctionId from the AuctionCreated event
const ev = receipt.logs
  .map(l => { try { return loanProtocol.interface.parseLog(l); } catch { return null; } })
  .find(p => p && p.name === "AuctionCreated");
const auctionId = ev.args.auctionId;
```

Once the auction ends, anyone can finalise it (finalisation is permissionless — your UI, a keeper, or the winning lender can call it):

```js
if (await loanProtocol.canFinalize(auctionId)) {
  await (await loanProtocol.finalizeAuction(auctionId)).wait();
}
```

Finalisation creates the loan and mints the position NFTs. The `loanId` equals the `auctionId` for that auction. To repay:

```js
const loan = await loanProtocol.getLoan(loanId);
await (await usdc.approve(loanProtocol.target, loan.repaymentAmount)).wait();
await (await loanProtocol.repayLoan(loanId)).wait(); // returns your collateral
```

---

## Flow 2 — Lend (bid on an auction)

```js
const auction = await loanProtocol.getAuction(auctionId);

// Approve the loan token for the principal you'd fund
await (await usdc.approve(loanProtocol.target, auction.loanAmount)).wait();

// Bid the total repayment you require. Lower beats the current bid.
// Stay within the valid range:
const maxBid = await loanProtocol.getMaximumBid(auctionId);
const minBid = await loanProtocol.getMinimumBid(auctionId);
const myBid  = minBid; // most competitive

await (await loanProtocol.placeBid(auctionId, myBid)).wait();
```

If you're outbid, your deposit is held for pull-based refund:

```js
const owed = await loanProtocol.getPendingRefund(myAddress, USDC_ADDR);
if (owed > 0n) await (await loanProtocol.claimRefund(USDC_ADDR)).wait();
```

If you win, finalisation funds the loan and you hold the lender position NFT. At maturity you receive the repayment. If the borrower defaults, claim the collateral after the grace period:

```js
if (await loanProtocol.canClaimCollateral(loanId)) {
  await (await loanProtocol.claimCollateral(loanId)).wait();
}
```

---

## Flow 3 — Trade a position

Derive the token ID from the loan ID and the side:

```js
const lenderTokenId   = await positionNFT.getLenderTokenId(loanId);   // loanId*2 + 1
const borrowerTokenId = await positionNFT.getBorrowerTokenId(loanId); // loanId*2

// List the lender position for 9,800 USDC
await (await loanProtocol.listPosition(
  loanId, "lender", USDC_ADDR,
  ethers.parseUnits("9800", 6),  // askingPrice
  0n                             // minOfferAmount (0 = no floor)
)).wait();
```

Buying is MEV-protected — pass the maximum you'll pay and the expected payment token, and approve first:

```js
const listing = await loanProtocol.getMarketplaceListing(lenderTokenId);
await (await usdc.approve(loanProtocol.target, listing.askingPrice)).wait();
await (await loanProtocol.buyPosition(
  lenderTokenId,
  listing.askingPrice,  // maxPrice — reverts if the listing moved up
  USDC_ADDR             // expectedPaymentToken — reverts if it changed
)).wait();
```

Or negotiate with an offer (`makeMarketplaceOffer` → seller `accept`/`counter`, buyer `acceptMarketplaceCounterOffer`). Note marketplace operations freeze in the `MATURITY_BUFFER` window before maturity — a `MarketplaceFrozen` revert near maturity is expected.

---

## The operator pattern (building on top)

This is how you build products on the protocol without ever holding user assets. A user grants your contract operator rights once; your contract then acts for them through the `…For` functions, while custody stays in the core.

```js
// User approves your contract as an operator (one-time)
await (await loanProtocol.setOperatorApproval(YOUR_ROUTER_ADDR, true)).wait();
```

Your router/curation/strategy contract then calls the `…For` variant:

```solidity
// Inside YourRouter.sol — create an auction for an approved user
function route(
    address user,
    address collateralToken, uint256 collateralAmount,
    address loanToken, uint256 loanAmount, uint256 maxRepayment,
    uint256 loanDuration, uint256 auctionDuration, uint256 bidStep
) external returns (uint256 auctionId) {
    // ... your own logic: whitelist checks, fees, accounting ...
    // The core verifies that address(this) is an approved operator of `user`.
    auctionId = loanProtocol.createAuctionFor(
        user,            // borrower
        user,            // collateralFrom
        collateralToken, collateralAmount,
        loanToken, loanAmount, maxRepayment,
        loanDuration, auctionDuration, bidStep
    );
}
```

The core enforces the authorisation: `createAuctionFor` reverts with `Unauthorized` unless `msg.sender` is the user or an approved operator of `collateralFrom`. The same pattern applies to `listPositionFor`. This is exactly how the reference `ListingService` works — and exactly how you can build your own curation layer or router. See [Build Your Own Curation Layer](build-your-own-curation-layer.md).

### Using the reference Curation Layer

If instead of building your own you want to route through the reference `ListingService` (the vetted-token path the hosted interface uses), the user approves it as an operator and calls `createListedAuction`:

```js
await (await loanProtocol.setOperatorApproval(LISTING_SERVICE_ADDR, true)).wait();
await (await listingService.createListedAuction(/* same params as createAuction */)).wait();
// reverts with TokenNotWhitelisted if a token isn't in the curated set
```

---

## Reading state and indexing

For UIs, poll the view functions (`getAuction`, `getLoan`, `getCurrentBid`, `getAuctionTimeRemaining`, `canFinalize`, `canClaimCollateral`, `isPositionListed`, …). For an index or subgraph, watch the events:

- Auctions: `AuctionCreated`, `BidPlaced`, `AuctionFinalized`, `AuctionCancelled`, `AuctionExpiredNoBids`, `AuctionExpiredNotFinalized`.
- Loans: `LoanRepaid`, `LoanDefaulted`, `LoanDefaultMarked`.
- Positions/marketplace: `PositionListed`, `PositionSold`, `MarketplaceOfferMade`, `MarketplaceOfferAccepted`, `BorrowerPositionTransferred`, `LenderPositionTransferred`.

```js
loanProtocol.on(loanProtocol.filters.AuctionCreated(), (auctionId, /* ... */ ) => {
  // index the new auction
});
```

For market-wide data (yield curve, credit-surface indicators) you don't need to build your own indexer — see the [Aletheia API](../data/aletheia-api.md) and [on-chain data access](../data/on-chain-data.md).

---

## Error handling

The contracts revert with typed custom errors (listed in the [Contract Reference](contracts-reference.md)). Decode them from the revert data:

```js
try {
  await loanProtocol.placeBid(auctionId, myBid);
} catch (e) {
  const decoded = loanProtocol.interface.parseError(e.data ?? e.error?.data);
  // decoded.name e.g. "BidTooHigh", "AuctionEnded"
}
```

Common reverts to expect:

- `TokenNotWhitelisted` — using `ListingService` with a non-curated token (the direct core path has no whitelist).
- `BidTooHigh` / `BidTooLow` / `BidStepTooHigh` — bid outside the current valid range; read `getMinimumBid`/`getMaximumBid` first.
- `PriceExceedsMaximum` / `PaymentTokenMismatch` — your `buyPosition` MEV guards fired because the listing changed.
- `MarketplaceFrozen` — within the maturity buffer; marketplace ops are paused for that loan.
- `Unauthorized` — calling a `…For` function without operator approval.
- `GracePeriodNotEnded` / `LoanNotMatured` — claiming collateral too early.

## Practical notes

- **Decimals** differ per token; never hardcode 18.
- **`bidStep = 0`** uses the protocol default; pass a value only to widen the step.
- **Finalisation is permissionless** — don't rely on a counterparty to finalise; your integration can call `finalizeAuction` itself.
- **Refunds are pull-based** — outbid lenders must call `claimRefund`; it isn't pushed.
- **`loanId == auctionId`** for the loan created from that auction.
- **Immutability** — these addresses and behaviours won't change; new versions ship at new addresses.

## Next

- [Build Your Own Curation Layer](build-your-own-curation-layer.md) — an independent whitelist/router on the operator model.
- [Frontend Guide](frontend.md) — building a UI against the protocol and the public API.
- [Direct Access Guide](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/direct-access-guide.md) — the no-code path via Arbiscan.
