# Concepts overview

This page maps Gavel V2: the layers, the building blocks, the engines and products built on them, and what is deployed. It ends by following one loan from start to finish. Each block has its own page; this one shows how they fit together.

Everything here describes the contracts as deployed on Arbitrum Sepolia. They are **not audited**.

## Three ideas

**Debt, security and custody are kept apart, as the law keeps them.** A debt is a promise to pay (Obligation). Security is a right over collateral that backs one or more debts, with rules for enforcing it (the Lien). Custody is where the collateral actually is (the vault). Every product is an arrangement of the three.

**Products are thin.** A product discovers terms, usually by auction, then creates obligations and liens with the result. Its escrow is kept in the vault, under the product's key and each user's account, except where a product must hold an item itself (the secondary market holds what it is selling, for example); each product's page says which. Its rules are its own; the blocks' rules are the same for every product.

**Contracts find each other by naming.** When a set of obligations is created, it names its vault and its lien contract, and those choices never change. Nothing is rewired after deployment, so a faulty contract can harm only what named it.

## The four layers

| Layer | Name | What it holds |
|---|---|---|
| **0** | **Primitives** | **The core:** the vault, Obligation (with its modules) and the Lien. **Venues and a service:** the spot market, the secondary market, and BOND. |
| **1** | **Mechanisms** | Contracts that originate, protect, package and settle instruments: the bullet loan's two products, the cover orchestrator, the future orchestrator, the bill packager. Also **feature contracts** attached at creation, such as resolvers and adapters. |
| **2** | **Instruments** | What people hold: loans, protection, futures, bills. An instrument has no contract of its own; it is the obligations and liens a mechanism created. |
| **3** | **Applications** | Frontends, agents, indexers, strategies, integrations with other venues. |

Between the core and the products sit **the engines**: five auction contracts that run bidding and nothing else, and one helper, `SealedFront`, that takes sealed bids in front of an engine but finds no price itself. They never touch the vault. A product opens an auction on an engine, holds the escrow, and settles the result on the core.

**The base layer** is layers 0 to 2 as published by the protocol's author, under one rule: no price feed, no admin and no fee beyond the emergency powers (pausing, and admitting contracts to the vault), every external read disclosed, and every lien resolving within bounds fixed at creation. Not everything the author publishes meets it: BOND stands outside the base layer while its owner holds a power to change which loans and prices it counts, a power that ends at a date fixed in the contract, or earlier if renounced. Anything built by anyone else, including products that use price feeds or charge fees, is a third-party product. See [the base layer and what you trust](base-layer-and-trust.md), which also states the exceptions.

## The building blocks

| Block | What it is | Page |
|---|---|---|
| **The vault** | One shared contract holding assets as user accounts. Products reserve and lock balances in it rather than holding them. A product must be admitted by the vault's owner, and approved by you for your own account, before it can act on your balance. | [The vault](vault.md) |
| **Obligation** | The register of debts. A *set* is one or more obligations created together, owed by one obligor and ending together. An obligation can have a fixed amount, or an amount resolved later, or come into force on a trigger, through modules named at creation. | [Obligation](obligation.md) |
| **Claim units and the obligor position** | The two tokens a set creates. Each obligation's claim is an ERC-1155 id with a fixed supply of units; whoever holds the units is paid. The obligor position is an ERC-721, one per set; whoever holds it owes the set and gets the collateral back on release. | [Claims and obligor positions](claims-and-positions.md) |
| **The Lien** | Binds one vault lock to one set. Released when the set is paid; enforced when it is breached, passing the collateral in kind to the claim holders. | [The Lien](lien.md) |
| **The engines** | `SingleAuction`, `GroupAuction`, `PostedOffer`, `BatchAuction` and `ClockAuction`: bidding, with no custody. | [The engines](engines.md) |
| **BOND** | A credit record: minted to whoever registered a secured loan as its interest is paid, staked on obligations, and burned when a staked obligation is not paid in time. | [BOND](../products/bond.md) |

## What is deployed

As deployed on Arbitrum Sepolia, at 14 November 2026. Addresses and verified source are on the [deployments](../deployments.md) page, which is kept current.

| Group | Contracts | Built on |
|---|---|---|
| **The core** | `Vault`, `Obligation` (with the linked libraries `ObligationLogic` and `ModuleLogic`), `ObligorPosition`, `Lien` | — |
| **The engines** | `SingleAuction`, `GroupAuction`, `PostedOffer`, `BatchAuction`, `ClockAuction`; the helper `SealedFront` | — |
| **[Bullet loan](../products/bullet-loan.md)** | `BulletLoanRequest`, `BulletLoanOffer` (with the libraries `BulletOpening` and `BulletSettlement`) | `SingleAuction`, `GroupAuction`; the core |
| **[Spot market](../products/spot-market.md)** | One `SpotMarket` per pair | `BatchAuction`; the vault |
| **[Secondary market](../products/secondary-market.md)** | `ClaimLotMarket`, `ObligorPositionMarket` (with the library `LotCreditIndex`) | `SingleAuction`; the vault and the core's tokens |
| **[Cash-settled future](../products/cash-settled-future.md)** | `FutureOrchestrator`, `SpotReferenceResolver` | `PostedOffer`; the core; a spot market's clearings |
| **[Insurance and bills](../products/insurance-and-bills.md)** | `CoverOrchestrator` (with `CoverIssuance` and `CoverSettlement`), `ContingentBondAdapter`, `BillPackager`, a `SpotMarket` for each bill series, and a `BillToken` for each series, created when it first takes a deposit | `PostedOffer`; the core |
| **[Liquidation cover](../products/liquidation-cover.md)** | `MorphoCoverAdapter`, `CoverAccount` | `CoverOrchestrator`; a test copy of Morpho Blue |
| **[BOND](../products/bond.md)** | `BondToken`, `BondProtocol`, `BondListing`, `BondCurve` | Reads the bullet loan and the spot markets |

The assets are test tokens (`mWBTC`, `mUSDC`, `mUSDT`, `mWETH`, `mcbBTC`), worth nothing. Anyone can mint them, and anyone can burn any holder's balance, the vault's included, so on the testnet the vault can come to hold less than it records. Real tokens do not behave this way.

Some contracts deployed earlier are still on chain but **superseded**: a first `Obligation`, `Lien`, `ObligorPosition` and `AuctionOrchestrator`, and a first BOND package. The first Obligation, Lien and `AuctionOrchestrator` remain admitted to the vault. All are marked as superseded in the address list. Build against the current ones.

## One loan, from start to finish

A borrower asks for a loan of mUSDC against mWBTC, from one lender, through `BulletLoanRequest` on `SingleAuction`.

1. **Deposit and approve.** The borrower deposits mWBTC into the vault and approves `BulletLoanRequest` as a consumer of their own account. Lenders do the same with mUSDC.
2. **Open.** The borrower opens a request: the amount, the term, and the most they will repay. The product locks the collateral in the vault under its own key and opens an auction on `SingleAuction`, ending between one and seven days later (a pause of the product pushes the end later by its length).
3. **Bid.** Each lender bids the repayment they would accept. The product reserves the lender's principal in the vault. The engine keeps the best bid; when a bid is beaten, the product releases that lender's reservation in the same transaction.
4. **Settle.** After the end, anyone may call `finalise`. The product closes the auction, checks the result and settles on the core, in one transaction:
   - Obligation creates a set of one obligation: the borrower owes the winning repayment at the due date. The borrower receives the set's **obligor position**; the lender receives the obligation's **claim units**.
   - The collateral lock is handed to the **Lien**, which binds it to the set.
   - The lender's principal moves to the borrower's free balance in the vault.
5. **While it runs,** either side can sell: the lender sells claim units on `ClaimLotMarket`, and the borrower can sell the obligor position, with the collateral subject to the debt, on `ObligorPositionMarket`.
6. **Repay.** The borrower, or anyone, pays the obligation, at the latest before the grace period ends. Paying early is allowed, but the full repayment is owed whenever it is paid. The claim holders collect what was paid. Anyone can then release the lien, and the collateral is credited for the obligor position's holder to claim.
7. **Or breach.** If the obligation is still unpaid seven days after the due date (the product's grace period), anyone may call `markBreached`, and the set is breached. Until the lien is enforced, anyone may still pay the set off in full, which credits the collateral to the obligor position's holder to claim. Once anyone enforces it, the collateral passes in kind to the claim holders, shared by what each is owed.

Each step is an ordinary call with a public rule. No step needs a price feed, a keeper with special rights, or anyone's permission. The same pattern, discovery on an engine and settlement on the core, runs every other product.

## Terms used throughout

- **Set:** obligations created together by one call, owed by one obligor and ending together. A lien secures a whole set, and a breach of any obligation breaches the set.
- **Obligor:** whoever owes a set; in practice, whoever holds its obligor position.
- **Claim holder:** whoever holds an obligation's claim units, and so is paid.
- **Consumer:** a contract the vault admits to reserve, lock and move balances, only on the accounts of users who approved it.
- **Grace:** time after the due date before an unpaid obligation is breached, fixed when its set is created.
- **TAKE:** the Lien's enforcement: the collateral passes in kind to the claim holders. If it holds enough of the loan's own currency to cover the debt, only what is owed is taken.
- **Pause-adjusted:** deadlines move later by the time a contract spends paused, so that a pause does not cause defaults. Engines cap how far an auction's end can move (30 days); each product's page states what a longer pause does.

## Next

- [The vault](vault.md), then [Obligation](obligation.md) and [the Lien](lien.md).
- To build at once: [writing](../build/writing.md) and the [contracts reference](../build/contracts-reference.md).
