# Deployments

Gavel V2 is deployed on one network, for testing.

| | |
|---|---|
| Network | **Arbitrum Sepolia** (testnet) |
| Chain id | 421614 |
| Status | Deployed for testing. **Not audited.** No contract is upgradeable |
| Tokens | Test tokens with no value |
| Owners | Every owned contract is owned by the account that deployed it, an externally owned account; `owner()` returns it |

## Addresses

The address list is kept in the companion repository, generated from the deployment records rather than typed:

- **[V2 testnet addresses](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.md)**: every contract, grouped as on these pages, with each spot pair and bill series, the test tokens, and the superseded contracts.
- **[`v2-testnet-addresses.json`](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.json)**: the same list, machine-readable, for scripts and frontends.

These docs never type an address; use the list.

## Verified source

Each address in the list links to Arbiscan, where every contract's source, the test tokens aside, is verified. That is the source to read: the bytecode on chain is what runs, and the verified source is what it was compiled from. Libraries linked into a contract (`ObligationLogic` and `ModuleLogic` into `Obligation`, for example) are deployed and verified separately.

The full ABI of each contract is on its Arbiscan page. The [contracts reference](build/contracts-reference.md) lists the functions, events and errors you are likely to use.

## What is deployed

| Group | Contracts | Pages |
|---|---|---|
| The core | `Vault`, `Obligation` (with `ObligationLogic` and `ModuleLogic`), `ObligorPosition`, `Lien` | [Concepts](concepts/overview.md) |
| The engines | `SingleAuction`, `GroupAuction`, `PostedOffer`, `BatchAuction`, `ClockAuction`; the helper `SealedFront` | [The engines](concepts/engines.md) |
| Bullet loan | `BulletLoanRequest`, `BulletLoanOffer` (with `BulletOpening`, `BulletSettlement`) | [Bullet loan](products/bullet-loan.md) |
| Spot market | One `SpotMarket` per pair | [Spot market](products/spot-market.md) |
| Secondary market | `ClaimLotMarket`, `ObligorPositionMarket` (with `LotCreditIndex`) | [Secondary market](products/secondary-market.md) |
| Cash-settled future | `FutureOrchestrator`, `SpotReferenceResolver` | [Cash-settled future](products/cash-settled-future.md) |
| Insurance and bills | `CoverOrchestrator` (with `CoverIssuance`, `CoverSettlement`), `ContingentBondAdapter`, `BillPackager`; a `SpotMarket` and a `BillToken` for each bill series | [Insurance and bills](products/insurance-and-bills.md) |
| Liquidation cover | `MorphoCoverAdapter` (three, one per ceiling), `CoverAccount` | [Liquidation cover](products/liquidation-cover.md) |
| BOND | `BondToken`, `BondProtocol`, `BondListing`, `BondCurve` | [BOND](products/bond.md) |

**Test contracts.** Morpho Blue is not deployed on Arbitrum Sepolia, so liquidation cover runs against a test copy (`MockMorpho`, with `MockOracle` and `MockIrm`) whose price and rate are set by a testnet key. A test cover adapter (`MockCoverAdapter`) is also deployed. None of these is part of the protocol.

**Test tokens.** `mWBTC`, `mUSDC`, `mUSDT`, `mWETH` and `mcbBTC` are the ones the products use; a few more mock tokens are deployed but unused. Anyone can `mint` them, and anyone can `burn` any holder's balance, the vault's included, so on the testnet the vault can come to hold less than it records. Real tokens do not behave this way.

**Bill tokens.** A series' `BillToken` address is fixed in advance and listed, but the token is created only when the series takes its first deposit, so it may not exist on chain yet.

## Superseded contracts

A first `Obligation`, `Lien`, `ObligorPosition` and `AuctionOrchestrator`, and a first BOND package, are still on chain. They are marked as superseded in the address list. The first `Obligation`, `Lien` and `AuctionOrchestrator` remain on the vault's consumer whitelist, so they can act on the accounts of users who approved them; approving them is not needed for anything on these pages. The current contracts are the ones these pages describe.

## v1

v1, the auction-based lending protocol, is live on Arbitrum One and audited. Its addresses are in [The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md), and its docs are under [`v1/`](../v1/).

## Next

- [Writing](build/writing.md): your first transactions on the testnet.
- [Reading data](build/reading-data.md): from the chain or from the testnet API.
