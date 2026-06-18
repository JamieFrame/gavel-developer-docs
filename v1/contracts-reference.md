# Contract Reference

A function-level reference for the deployed contracts. Read [Architecture](architecture.md) first for the model; see the protocol repo for [deployed addresses](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md) and verified source, and the [Direct Access Guide](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/direct-access-guide.md) for worked end-to-end flows.

The system is two parallel stacks. This reference documents the **ERC-20 stack** (`LoanProtocol`, `PositionNFT`, `ListingService`) in full, then describes the **NFT stack** (`NFTLoanProtocol`, `NFTPositionNFT`, `NFTListingService`) as its differences. All amounts are in token base units. Position token IDs follow `borrower = loanId × 2`, `lender = loanId × 2 + 1`.

---

# LoanProtocol

The core contract: collateral custody, auctions, the loan lifecycle, and the position marketplace. Permissionless and immutable; accepts any ERC-20.

## Enums

- `AuctionStatus { OPEN, FINALIZED, CANCELLED, EXPIRED }`
- `LoanStatus { ACTIVE, REPAID, DEFAULTED }`
- `MarketplaceOfferStatus { PENDING, ACCEPTED, REJECTED, COUNTERED, CANCELLED, EXPIRED }`

## Structs

**`Auction`** — `borrower`, `collateralToken`, `collateralAmount`, `loanToken`, `loanAmount`, `maxRepayment`, `loanDuration`, `auctionEnd`, `currentBidder`, `currentBid`, `bidCount`, `status`, `bidStep`.

**`Loan`** — `borrower`, `collateralToken`, `collateralAmount`, `loanToken`, `loanAmount`, `repaymentAmount`, `maturityTimestamp`, `lender`, `status`.

**`MarketplaceListing`** — `seller`, `loanId`, `positionType`, `paymentToken`, `askingPrice`, `minOfferAmount`, `listedAt`, `active`.

**`MarketplaceOffer`** — `buyer`, `amount`, `escrowedAmount`, `status`, `counterAmount`, `createdAt`, `expiresAt`, `paymentToken`.

## Operator approval

| Function | Description |
|---|---|
| `setOperatorApproval(address operator, bool approved)` | Grant or revoke an operator to act for `msg.sender` via the `…For` functions. |
| `operatorApprovals(address owner, address operator) → bool` | Whether `operator` is approved for `owner`. |

## Core functions

| Function | Description |
|---|---|
| `depositCollateral(address token, uint256 amount)` | Deposit ERC-20 collateral into your protocol balance. Requires prior `approve`. |
| `withdrawCollateral(address token, uint256 amount)` | Withdraw unused (unlocked) collateral. |
| `createAuction(collateralToken, collateralAmount, loanToken, loanAmount, maxRepayment, loanDuration, auctionDuration, bidStep) → uint256 auctionId` | Open an auction. `maxRepayment` caps total repayment; durations in seconds; `bidStep = 0` uses the protocol default. |
| `createAuctionFor(borrower, collateralFrom, collateralToken, collateralAmount, loanToken, loanAmount, maxRepayment, loanDuration, auctionDuration, bidStep) → uint256 auctionId` | Operator variant. `msg.sender` must be an approved operator of `collateralFrom`. |
| `cancelAuction(uint256 auctionId)` | Cancel an auction with no bids (borrower only). |
| `placeBid(uint256 auctionId, uint256 repaymentAmount)` | Bid the total repayment you require; lower beats the current bid. Requires `approve` of the loan token for `loanAmount`. |
| `claimRefund(address token)` | Pull-based refund of outbid deposits. |
| `finalizeAuction(uint256 auctionId)` | Finalise an ended auction: creates the loan from the winning bid, mints positions, disburses the loan. Callable by anyone. |
| `claimExpiredAuction(uint256 auctionId)` | Reclaim assets from an auction that ended but was never finalised. |
| `repayLoan(uint256 loanId)` | Repay in full and reclaim collateral (borrower-position holder). Requires `approve` of the loan token for `repaymentAmount`. |
| `claimCollateral(uint256 loanId)` | Seize collateral on a defaulted loan (lender-position holder, after the grace period). |
| `markDefault(uint256 loanId)` | Permissionlessly mark a past-grace loan as defaulted. Emits an event; transfers nothing. |

## Marketplace functions

| Function | Description |
|---|---|
| `listPosition(loanId, positionType, paymentToken, askingPrice, minOfferAmount)` | List a position for sale. `positionType` is `"borrower"` or `"lender"`; `minOfferAmount = 0` means no floor. |
| `listPositionFor(loanId, seller, positionType, paymentToken, askingPrice, minOfferAmount)` | Operator variant of `listPosition`. |
| `unlistPosition(uint256 tokenId)` | Remove your listing. |
| `updateListingPrice(uint256 tokenId, uint256 newPrice)` | Change the asking price. |
| `cleanStaleListing(uint256 tokenId)` | Permissionlessly clear a listing whose underlying loan state makes it stale. |
| `buyPosition(uint256 tokenId, uint256 maxPrice, address expectedPaymentToken)` | Buy at the asking price. MEV-protected: reverts if price `> maxPrice` or the payment token differs. |
| `makeMarketplaceOffer(tokenId, offerAmount, offerDuration, expectedPaymentToken) → uint256 offerId` | Make an offer below asking; funds are escrowed. |
| `cancelMarketplaceOffer(tokenId, offerId)` | Cancel your pending offer and reclaim escrow. |
| `rejectMarketplaceOffer(tokenId, offerId)` | Seller rejects an offer (returns escrow). |
| `expireMarketplaceOffer(tokenId, offerId)` | Mark an expired offer and release escrow. |
| `counterMarketplaceOffer(tokenId, offerId, counterAmount, counterDuration)` | Seller counters with a new price. |
| `acceptMarketplaceOffer(tokenId, offerId)` | Seller accepts an offer; position transfers, escrow releases. |
| `acceptMarketplaceCounterOffer(tokenId, offerId)` | Buyer accepts the seller's counter. |

## View functions

Core: `getAuction(auctionId) → Auction`, `getLoan(loanId) → Loan`, `getCurrentBid(auctionId) → (address bidder, uint256 amount)`, `getBidCount(auctionId) → uint256`, `getCollateralBalance(user, token) → uint256`, `getPendingRefund(user, token) → uint256`, `getBorrowerPositionOwner(loanId) → address`, `getLenderPositionOwner(loanId) → address`, `getAuctionTimeRemaining(auctionId) → uint256`, `getLoanTimeRemaining(loanId) → uint256`, `isInGracePeriod(loanId) → bool`, `canClaimCollateral(loanId) → bool`, `getFinalizationTimeRemaining(auctionId) → uint256`, `canFinalize(auctionId) → bool`, `canClaimExpiredAuction(auctionId, claimant) → (bool canClaim, bool isLender)`, `getMinimumBid(auctionId) → uint256`, `getMaximumBid(auctionId) → uint256`, `getAuctionBidStep(auctionId) → uint256`, `loanNonce() → uint256`.

Marketplace: `getMarketplaceListing(tokenId) → MarketplaceListing`, `getMarketplaceOffer(tokenId, offerId) → MarketplaceOffer`, `getMarketplaceOfferCount(tokenId) → uint256`, `isPositionListed(tokenId) → bool`.

## Constants (read on-chain)

`MIN_AUCTION_DURATION`, `MAX_AUCTION_DURATION`, `MIN_LOAN_DURATION`, `MAX_LOAN_DURATION` (loans can run up to ~30 years, for a full yield curve), `GRACE_PERIOD` (post-maturity window before seizure), `FINALIZATION_WINDOW`, `MIN_BID_STEP`, `MIN_OFFER_DURATION`, `MATURITY_BUFFER` (freezes marketplace ops near maturity), `MAX_OFFERS_PER_LISTING`. All are compile-time constants — read them from the contract; none is owner-settable.

## Events

For indexing/subgraphs, the lifecycle events are: `CollateralDeposited`, `CollateralWithdrawn`, `AuctionCreated`, `AuctionCancelled`, `BidPlaced`, `AuctionFloorReached`, `RefundAvailable`, `RefundClaimed`, `AuctionFinalized`, `AuctionExpiredNoBids`, `AuctionExpiredNotFinalized`, `ExpiredAuctionClaimed`, `LoanRepaid`, `LoanDefaulted`, `LoanDefaultMarked`, `BorrowerPositionTransferred`, `LenderPositionTransferred`, `OperatorApprovalSet`. Marketplace: `PositionListed`, `PositionUnlisted`, `ListingPriceUpdated`, `MarketplaceOfferMade`, `MarketplaceOfferCancelled`, `MarketplaceOfferRejected`, `MarketplaceCounterOffer`, `MarketplaceOfferAccepted`, `PositionSold`, `MarketplaceOfferExpired`, `MarketplaceOfferRefundQueued`, `MarketplaceRefundClaimed`.

## Custom errors

The contract reverts with typed errors. Common ones by category:

- **Validation:** `ZeroAddress`, `InvalidToken`, `SameToken`, `InvalidAmount`, `InvalidRepayment`, `InvalidDuration`, `InvalidPrice`, `InvalidPositionType`.
- **Auction:** `AuctionNotFound`, `AuctionNotOpen`, `AuctionEnded`, `AuctionStillOpen`, `AuctionNotExpired`, `BidTooHigh`, `BidTooLow`, `BidStepTooHigh`, `NoBids`, `HasBids`, `FinalizationWindowExpired`, `FinalizationWindowActive`.
- **Authorisation:** `NotBorrower`, `Unauthorized`, `NotSeller`, `NotBuyer`, `NotPositionOwner`, `CannotBuyOwnPosition`.
- **Loan:** `LoanNotFound`, `LoanNotActive`, `LoanNotMatured`, `GracePeriodNotEnded`, `GracePeriodExpired`, `LoanExpired`, `AlreadyClaimed`, `InsufficientCollateral`, `InsufficientBalance`, `NoRefundAvailable`.
- **Marketplace:** `NotListed`, `AlreadyListed`, `ListingNotStale`, `InvalidOffer`, `OfferNotFound`, `InvalidOfferStatus`, `OfferNotExpired`, `OfferExpired`, `OfferDurationTooShort`, `OfferDurationTooLong`, `OfferDurationExceedsLoanMaturity`, `MarketplaceFrozen`, `TooManyOffers`, `PriceExceedsMaximum`, `PaymentTokenMismatch`, `OfferBelowMinimum`.

---

# PositionNFT

The ERC-721 representing loan positions. It extends standard `IERC721`, so all ordinary NFT calls (`ownerOf`, `transferFrom`, `approve`, etc.) work. The protocol-only functions cannot be called by anyone except the core.

`PositionType { BORROWER, LENDER }`

**Protocol-only (callable only by `LoanProtocol`):** `mintBorrowerPosition(loanId, to) → uint256`, `mintLenderPosition(loanId, to) → uint256`, `setLoanMetadata(loanId, collateralToken, collateralAmount, loanToken, loanAmount, repaymentAmount, maturityTimestamp)`, `burn(tokenId)`, `protocolTransfer(tokenId, from, to)`.

**Pure / view helpers (useful to builders):**

| Function | Description |
|---|---|
| `getBorrowerTokenId(loanId) → uint256` | Borrower token ID (`loanId × 2`). |
| `getLenderTokenId(loanId) → uint256` | Lender token ID (`loanId × 2 + 1`). |
| `getLoanId(tokenId) → uint256` | Loan ID from a token ID. |
| `getPositionType(tokenId) → PositionType` | Borrower or lender. |
| `exists(tokenId) → bool` | Whether the token exists. |

The `loanProtocol` reference is fixed at initialisation and cannot be changed.

---

# ListingService (Curation Layer)

Optional layer that fronts the core with a token whitelist and an optional fee. To use it, a user first approves it as an operator on `LoanProtocol` (`setOperatorApproval(listingService, true)`), then calls `createListedAuction` — the service checks the whitelist, applies any fee, and delegates to `LoanProtocol.createAuctionFor`.

**User entrypoint:**

| Function | Description |
|---|---|
| `createListedAuction(collateralToken, collateralAmount, loanToken, loanAmount, maxRepayment, loanDuration, auctionDuration, bidStep) → uint256 auctionId` | Create an auction through the curated layer. Reverts (`TokenNotWhitelisted`) if either token is not whitelisted. |

**Views:** `isListedAuction(auctionId) → bool`, `isCollateralWhitelisted(token) → bool`, `isLoanTokenWhitelisted(token) → bool`.

**Admin (owner-only — the curation layer administrator):** `setCollateralWhitelist(token, whitelisted)`, `setLoanTokenWhitelist(token, whitelisted, minBidStep)` (a non-zero `minBidStep` is required when whitelisting), `batchSetCollateralWhitelist(tokens[], whitelisted)`, `batchSetLoanTokenWhitelist(...)`, `pause()`, `unpause()`. These define the curated set; they do not affect the permissionless core.

**Events:** `ListedAuctionCreated`, `CollateralTokenUpdated(token, whitelisted)`, `LoanTokenUpdated(token, whitelisted)`.

---

# NFT stack — differences from the ERC-20 stack

`NFTLoanProtocol`, `NFTPositionNFT`, and `NFTListingService` mirror the ERC-20 stack — same lifecycle, same auction and marketplace mechanics, same operator model, analogous events and errors. The differences are:

**Collateral is an ERC-721, not a token amount.** `NFTLoanProtocol.createAuction` takes the collateral NFT and its token ID instead of a token + amount:

```
createAuction(
    address collateralNFT,
    uint256 collateralTokenId,
    address loanToken,
    uint256 loanAmount,
    uint256 maxRepayment,
    uint256 loanDuration,
    uint256 auctionDuration,
    uint256 bidStep
) → uint256 auctionId
```

`createAuctionFor` takes the same parameters plus the borrower/`collateralFrom` addresses, with the same operator-approval requirement.

**Curation uses a collection whitelist.** `NFTListingService` whitelists collections rather than collateral tokens: `setCollectionWhitelist(collection, whitelisted)` (owner-only), with events `CollectionWhitelistUpdated`, `LoanTokenWhitelistUpdated`, and `CollectionInfoUpdated`. At launch no collection is whitelisted, so the curated NFT path is inactive until one is added; the direct core path remains open.

**Position metadata.** `NFTPositionNFT` additionally caches a collateral image reference for richer position metadata. Its position-ID scheme, mint/burn/protocol-transfer access control, and pure helpers match `PositionNFT`.

Everything else — bidding, finalisation, repayment, default and grace, the full marketplace (`listPosition`/`buyPosition`/offers), the view functions, and the operator model — behaves as documented for the ERC-20 stack.
