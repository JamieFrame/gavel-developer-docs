# Architecture

This document explains how The Gavel Protocol is put together: the contracts, how they relate, the paths by which users reach them, and the properties a builder can rely on. It is the foundation for the rest of these docs — the contract reference, integration guide, and frontend guide all assume the model described here.

For deployed addresses and verified source, see the [The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol) repository.

## Design philosophy

The Gavel Protocol is a fixed-term, collateralised lending protocol with four defining choices:

- **Oracle-free.** The protocol never asks an external price feed what collateral is worth. Interest rates are discovered by a competitive auction, and collateral is valued by the lenders who choose to bid. There is no price oracle to manipulate, and no oracle dependency to break.
- **Permissionless.** The core contracts accept any ERC-20 (or, for the NFT stack, any ERC-721). There is no whitelist in the core. Anyone can create an auction, bid, or trade a position by calling the contracts directly.
- **Immutable.** The deployed contracts are non-upgradeable. There is no proxy admin and no upgrade path; the logic cannot change after deployment. New functionality ships as new, separately deployed contracts (a "redeploy-as-v2" model), never as an in-place upgrade.
- **No liquidation.** Loans are fixed-term. There is no liquidation engine and no margin call. A loan runs to maturity; if it isn't repaid, the lender claims the collateral after a grace period.

## The layered model

The system is organised in three layers, and exists as two parallel stacks — one for ERC-20 collateral, one for ERC-721 (NFT) collateral.

```
                    ┌──────────────────────────────────────────┐
                    │                  USERS                    │
                    │           (any EOA or contract)           │
                    └───────────────────┬──────────────────────┘
                                        │
            ┌───────────────────────────┼───────────────────────────┐
            │  curated path                                direct path│
            ▼                                                         ▼
 ┌──────────────────────────┐                          ┌──────────────────────────┐
 │      CURATION LAYER       │                          │                          │
 │  ListingService /         │  whitelist, optional     │                          │
 │  NFTListingService        │  fee, then delegates ──► │                          │
 │  (optional, operator)     │                          │                          │
 └──────────────────────────┘                          │   direct, any token,     │
                                                        │   no whitelist           │
            ┌───────────────────────────────────────────┘                          │
            ▼                                                                       ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │                              CORE PROTOCOL LAYER                                  │
 │                     LoanProtocol  /  NFTLoanProtocol                              │
 │   collateral custody · auctions · loan lifecycle · secondary marketplace         │
 └────────────────────────────────────┬─────────────────────────────────────────────┘
                                      │  mints / burns / protocol-transfers
                                      ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │                               POSITION LAYER                                     │
 │                      PositionNFT  /  NFTPositionNFT                               │
 │            borrower and lender positions as transferable ERC-721s                │
 └────────────────────────────────────────────────────────────────────────────────┘
```

### Core Protocol Layer — `LoanProtocol` / `NFTLoanProtocol`

The heart of the system. A single core contract per stack owns the whole lifecycle: it takes custody of collateral, runs auctions, creates and settles loans, and hosts the secondary marketplace for positions. It accepts any token, enforces no whitelist, and is the contract you call for direct, permissionless access. `NFTLoanProtocol` mirrors `LoanProtocol` for ERC-721 collateral.

### Position Layer — `PositionNFT` / `NFTPositionNFT`

Every loan has two sides, and each side is an ERC-721. When a loan is created, the core mints a **borrower position** and a **lender position**. These NFTs *are* the positions: holding the lender NFT is what entitles you to repayment; holding the borrower NFT is what entitles you to reclaim collateral on repayment. Because they're standard ERC-721s, positions can be transferred and sold. Only the core protocol can mint, burn, or perform protocol transfers on them — the link to the core is fixed at initialisation and cannot be changed.

The token-ID scheme is deterministic: for a given `loanId`, the **borrower** position is `tokenId = loanId × 2` and the **lender** position is `tokenId = loanId × 2 + 1`.

### Curation Layer — `ListingService` / `NFTListingService`

An **optional** layer in front of the core. It maintains a whitelist of accepted collateral and loan tokens (and, for NFTs, accepted collections), can apply a fee, and then delegates auction creation to the core on the user's behalf. It is what the hosted interface routes through, so that interface users transact against a vetted token set. It is not privileged: it reaches the core through the same public operator mechanism any contract could use, and using it is never required.

## Access paths

There are two ways to reach the core, and understanding the difference is the key architectural point for a builder.

**Direct (permissionless).** Call `LoanProtocol` / `NFTLoanProtocol` yourself. You choose the tokens; no whitelist applies; no fee is taken by a curation layer. This is the raw protocol, and it is fully supported — see the [Direct Access Guide](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/direct-access-guide.md) in the protocol repo.

**Curated (via a Curation Layer).** Call `ListingService` / `NFTListingService`, which checks the token against its whitelist, applies any configured fee, and then creates the auction in the core. To act on your behalf, a curation layer must be an **approved operator** of yours.

### The operator-approval model

The core lets a user delegate specific actions to another address. A user calls `setOperatorApproval(operator, true)`; thereafter the approved operator may call the `…For` variants (e.g. `createAuctionFor`) on that user's behalf. This is how a curation layer — or any custom layer you build — acts for a user without ever taking custody of their assets. Approval is explicit, granted only by the user, and revocable. It is also the extension point for your own products: **anyone can build an alternative curation layer or routing contract** on exactly this mechanism, which is why the curated path is non-exclusive.

## The auction mechanism

A borrower opens an auction stating the collateral they're posting, the token and amount they want to borrow, and the maximum total repayment they'll accept. Lenders then compete by bidding the **repayment amount** they require to fund the loan. A lower repayment is more favourable to the borrower, so lenders bid each other down, each improving on the current best by at least the auction's bid step, bounded by the borrower's maximum. When the auction ends it is finalised, the loan is created against the winning (lowest-repayment) bid, and the loan tokens are released to the borrower. The spread between the loan amount and the agreed repayment is the lender's return — discovered by the market, with no oracle in the loop.

## Loan lifecycle

```
  AUCTION ──finalise──► LOAN (active) ──repay────► closed (collateral returned)
     │                       │
     │                       └──mature + grace──► DEFAULT (lender claims collateral)
     ├──cancel (no bids)──► cancelled
     └──no bids, window passes──► expired
```

On finalisation the core mints the borrower and lender position NFTs. Repayment by the borrower returns the collateral and closes the loan. If the loan is not repaid by maturity, then after a fixed **grace period** the lender may claim the collateral. The grace period is a compile-time constant — it is not an administrative parameter.

## Secondary market

Both positions in a loan are transferable, so either side can exit early by selling its NFT through the marketplace built into the core. A holder lists a position (by `loanId` and side); buyers can purchase at the asking price or negotiate via offers and counter-offers. Purchases are MEV-protected: a buy specifies the maximum price and the expected payment token, and reverts if the listing has shifted underneath the transaction. Marketplace functions operate on the position **`tokenId`** (using the `loanId × 2` / `loanId × 2 + 1` scheme above), with the listing functions taking the `loanId` and side instead.

This makes a borrower position mechanically similar to an option: a buyer of the borrower NFT pays a price (the premium) for the right to repay (the strike) before maturity (the expiry) and reclaim the collateral.

## Properties a builder can rely on

- **Immutability.** Addresses and logic are fixed. Integrate against them with confidence that they will not change beneath you; new versions arrive at new addresses.
- **Any token.** The core imposes no token whitelist. If you build a curated experience, the whitelist is yours to define — it is not enforced by the core.
- **No custody by intermediaries.** The operator model lets layers act for users without holding assets. A curation layer or router never takes custody; the core does, only for the duration of an auction or loan.
- **Deterministic position identity.** The `loanId`-to-`tokenId` mapping is fixed arithmetic, so you can derive position token IDs without a lookup.
- **Oracle independence.** There is no price feed to integrate, monitor, or guard against. Collateral valuation is a market activity performed by bidders.

## Where to go next

- **Contract reference** — function-level interface for each contract *(in this repo)*.
- **Integration guide** — building on the protocol, the operator pattern in practice *(in this repo)*.
- **Frontend guide** — building a custom interface against the protocol and the public API *(in this repo)*.
- **Direct Access Guide** — calling the core directly, with worked flows ([protocol repo](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/direct-access-guide.md)).
- **Deployed contracts** — addresses and verified source ([protocol repo](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md)).
