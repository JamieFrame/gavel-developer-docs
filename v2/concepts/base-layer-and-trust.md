# The base layer and what you trust

This page sets out what you rely on when you use Gavel V2: the rule that defines the base layer, the four promises its contracts keep, the powers their owners hold, and where those powers stop. Everything here describes the contracts as deployed on Arbitrum Sepolia, which are **not audited**.

## The base layer

The **base layer** is the core, the venues and the products published by the protocol's author (layers 0 to 2 in the [overview](overview.md)) that meet one rule. A contract belongs to it only if:

1. it is built only from the core and other base-layer contracts;
2. it uses **no price feed**: amounts and triggers come from Gavel's own auctions, obligation states, time and holders' choices, and any read of another protocol's on-chain state is **disclosed per obligation, by target**, when the obligation is created;
3. it has **no admin and no fee** beyond the emergency surface below;
4. it follows the core's rules: anyone may call it, payments are pulled rather than pushed, a pause never blocks an exit, and it keeps the four promises;
5. every lien it uses resolves within bounds fixed at creation.

Whatever is in the base layer therefore carries a guarantee, not just a label. Anything else, including any product that reads a price feed, charges a fee or has an operator, is a third-party product: it may be built on the same blocks, but the guarantee does not extend to it.

The rule on outside reads is kept by the author and checked through the disclosures; the chain cannot tell a rate index from a price feed, so the blocks cannot enforce it. Two current cases, stated plainly:

- **Liquidation cover reads a price.** `MorphoCoverAdapter` reads the covered lending market's own oracle, the price that market liquidates by, to refuse cover on a position already too close to liquidation. The read is disclosed with each cover. See [liquidation cover](../products/liquidation-cover.md).
- **BOND is outside the base layer while its listing power lives** (below). It enters when the power ends.

## The four promises

Every base-layer contract keeps these for every user, whatever token, counterparty or contract anyone else has chosen.

1. **Conservation.** The vault records an asset only as it measures it entering, and no record can overflow. Only the token itself can make the vault hold less than it records (by its issuer's powers, or by misreporting its balance), and then only for that token.
2. **Isolation.** No position, lien or account can draw on another's value. A choice of token, contract or counterparty harms only those who made it.
3. **Exits.** You can always take back what is free to you, subject to the token itself, and every reservation and lock can always be released by whoever holds it. No pause, de-authorisation or other party's choice removes an exit. A running pause can **delay** an auction's exits (an auction with bids is held while its product is paused, and its deadlines move later by the pause), but it never removes them.
4. **Resolution.** Every lien resolves within bounds fixed at creation, extended by any time spent paused. Pauses have no maximum length, so a pause can delay a resolution for as long as it lasts, but never redirect one.

**One exception to bounded resolution.** Cover that pays on another set's breach (a `STATE` trigger) cannot lapse while the contract it reads fails to answer, so a target that always failed would hold the cover pending and its collateral locked. The targets in use on the testnet (Obligation itself, and an adapter that only reads storage) cannot fail this way.

The core is as open as it can be without breaking these promises. It admits any token, any counterparty and any contract that names it, and refuses something only where leaving it open would break a promise. Risks that harm only whoever chose them are left open and disclosed, for products and frontends to curate:

- a token with unusual behaviour (rebasing, fee-on-transfer, an issuer that can move or freeze balances);
- an ERC-721 deposited under the wrong kind of key, which can then never be withdrawn;
- a token sent to the vault outside `deposit`, or a credit to the vault's own address, neither of which can be recovered.

## The emergency surface

The contracts' owners hold a narrow set of powers. **None of them can move a user's funds, change an obligation's terms, or redirect a claim or a position.** No contract is upgradeable, and `renounceOwnership` is disabled on the core, so no core contract can be left with no one able to pause it.

**Pauses.** A pause stops new activity; exits and settlement continue, and deadlines move later by the time paused.

| Contract | A pause stops | Still works |
|---|---|---|
| `Obligation` | Creating sets | Paying, breach, collection, redemption, release, the Lien's calls, every module call |
| `Lien` | Creating liens | Release, payoff, enforcement, claim |
| `BulletLoanRequest`, `BulletLoanOffer` | Openings, bids, cancellation, closing and settlement | Releasing escrow and expiry |
| Each `SpotMarket` | Placing orders, clearing, expiring an empty slot, abandoning a stalled clearing | Cancelling orders, draining |
| `ClaimLotMarket`, `ObligorPositionMarket` | New sales and bids | Settlement, cancellation, refunds, claiming units |
| `FutureOrchestrator` | New offers and takes | Cancelling and expiring offers, settlement, firing, payment, release |
| `CoverOrchestrator` | New offers and fills | Cancelling and closing offers, settlement, claiming units, release |

The vault, `ObligorPosition`, the engines, `SealedFront`, the resolvers and adapters, `BillPackager`, liquidation cover's contracts, and BOND's `BondToken`, `BondProtocol` and `BondCurve` have **no pause**. While a product is paused, escrow already committed to an open auction is held until the unpause; escrow the product has already let go stays releasable throughout. Each product's page gives its pause in full.

**The vault's whitelist.** The vault's owner decides which contracts may reserve, lock and move balances (`setAuthorisedConsumer`), and can empty the list at once (`emergencyDeauthoriseAll`). Removing a contract stops it starting anything new; it moves no balance, and withdrawals, releases and settlement in progress continue. Payment from a vault balance to a de-authorised Obligation or Lien stops too; payment from a wallet continues. See [the vault](vault.md#exits-what-keeps-working).

Two consequences to know:

- **A new product needs the owner's admission** before it can reserve or lock users' balances, as well as each user's approval. A product that only reads the record, or that creates obligations and liens with its own funds, needs no admission.
- **De-authorising, or pausing, a spot market near a reference time** can affect which clearings a cash-settled future counts. See [cash-settled future](../products/cash-settled-future.md).

**One-shot wiring.** `ObligorPosition`'s link to Obligation was set once at deployment and cannot be changed. Everything else is named per set at creation: each set names its vault and lien contract, and they never change.

## BOND's listing power

Beside the emergency surface sits one economic power. `BondListing`'s owner decides, within fixed bounds, which loan tokens, priced assets and sources **future** BOND registrations count:

- additions are announced and take effect 7 days later, and can be cancelled before then;
- removals are immediate and final, and apply to new registrations only;
- mint caps and price ceilings can only be lowered directly; raising one goes through the 7-day queue;
- the power ends at `LISTING_END` (on the testnet, 14 November 2027), or earlier if the owner renounces it; after that, the lists are fixed.

It moves no funds and changes no existing registration, except that a lowered cap applies to that registration's later mints. Because it is an admin power, BOND stands outside the base layer until it ends. See [BOND](../products/bond.md).

## Who holds the powers on the testnet

On Arbitrum Sepolia, **every owned contract is owned by the account that deployed it**, an externally owned account, not a multisig. That includes the vault (and so its whitelist), Obligation, the Lien, every pausable product, and `BondListing`. You can check any contract's owner with `owner()`, the whitelist with `authorisedConsumerCount()` and `authorisedConsumerAt(i)` on the vault, and `BondListing`'s end with `LISTING_END()`.

The superseded contracts listed on the [deployments](../deployments.md) page are still on chain, and the first Obligation, Lien and `AuctionOrchestrator` remain on the vault's whitelist. They can act only on the accounts of users who approved them.

## What you trust, and what you do not

**You trust:**

- **the contracts as deployed.** Their behaviour is fixed; you trust that the bytecode does what its verified source says. The source is on Arbiscan, linked from the [deployments](../deployments.md) page. The contracts are not audited.
- **the chain.** Arbitrum Sepolia's sequencer and block times. Deadlines are measured against the block's timestamp.
- **the tokens you use.** A token's own behaviour (an issuer's freeze, a blocklist, a pause) is outside the protocol. A loan token whose issuer pauses it can stop a borrower repaying while the grace period runs.
- **the owners, within the emergency surface.** That they will not pause without cause or admit a harmful contract to the vault. Each user's approval still limits an admitted contract to the accounts that approved it.
- **any module or outside contract your obligation names**, for exactly what it was named to answer. Each is disclosed at creation and harms only the obligations that named it.

**You do not need to trust:**

- **any frontend, indexer, API or agent.** Every contract can be called and read directly; the outcome is the same whoever submits the transaction.
- **keepers.** Anyone may clear a spot batch or finalise an auction; a keeper chooses nothing, and if no one acts, the only effect is delay.
- **an operator you approve,** beyond its approval. An operator can act for you only where you approved it, and cannot redirect your funds: `withdrawFor`, for example, pays only you.

## Next

- [Writing](../build/writing.md): approvals and calls in practice.
- [Composing a product](../build/composing-a-product.md): building on the core, and admission to the vault.
