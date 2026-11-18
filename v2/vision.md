# Vision

This page sets out what The Gavel Protocol is for, the commitments it is built to, and where it stops. The rest of these docs show how it works and how to build on it.

## A public record of promises

Every commercial order rests on a few questions: who holds what, who owes whom, what makes a promise credible, what things are worth, and who can be trusted. Today each institution keeps its own books of these, and they are reconciled at cost after the fact.

Gavel puts them on one public record. Each question has a building block:

| The question | The block | The institution it records |
|---|---|---|
| Who holds what? | The vault | Property and possession |
| Who owes what to whom, and when? | Obligation | Contract and debt |
| What makes a promise credible, and what happens if it fails? | The Lien | The pledge and the security interest |
| What is a thing worth? | The spot market and the secondary market | The market and the fair |
| Who can be trusted? | BOND | Reputation, the merchant's good name |

Assets are held in a shared vault. Promises are recorded as obligations. Collateral is pledged under liens that enforce by rules everyone agreed to in advance. Prices and rates come from open auctions, not from price feeds. A token, BOND, is earned by keeping promises: apart from a fixed amount issued at its launch, it is minted only when interest on a secured loan is paid, and a stake of it is lost when a promise it backs is not kept.

## Building blocks, not one product

Gavel is not a lending product with features around it. It is a minimal set of building blocks from which **any debt or derivative that does not need a price feed can be composed**:

- a **bullet loan** is one obligation and one lien;
- an **instalment loan** or a **note** is a schedule of obligations under one lien;
- **insurance** is an underwriter's obligation that comes into force when someone else's loan is not repaid;
- a **cash-settled future** is two obligations whose amounts are fixed from recorded spot prices, each secured by margin;
- a **bill** is a token packaging insured claims that fall due in the same week.

None of these needs a contract of its own at the bottom of the stack. Each is a thin product that arranges the same blocks, so each inherits their rules. The bullet loan, the cash-settled future, insurance and the insured bill, and liquidation cover are built this way on the testnet, beside the venues they trade on (the spot market and the secondary market) and BOND, which reads the record rather than writing to it. See the [concepts overview](concepts/overview.md).

Equity is the one thing deliberately not composed from the blocks. Shares are issued by their issuers' own contracts, under the issuers' own rules, and the blocks take them as assets: collateral to borrow against, or something to sell.

## Secured credit in the shape the law knows

Traditional finance has the legal half of secured credit. The law says how a security interest is created, how others learn of it, how it ranks, when the creditor may enforce, how the collateral is sold and who receives the proceeds. What it lacks is a shared record: each lender, custodian and servicer keeps its own books.

Decentralised finance has the other half, one open record anyone can read and settle against. But most on-chain lending looks nothing like the law of secured credit: a position is liquidated when a price feed crosses a line, and a liquidator keeps a bonus.

Gavel's liens are built in the shape of secured credit:

| The law of secured credit | Gavel's building blocks |
|---|---|
| A security interest attaches to identified collateral for an identified debt | A lien binds one vault lock to one set of obligations |
| Third parties can learn of it | The lock, the lien and the obligations are public on chain |
| Priority between creditors | A locked asset cannot secure a second set |
| Default, then enforcement | An obligation is breached only after its due date and a grace period fixed at creation; enforcement then follows by a rule anyone can trigger |
| Application of proceeds | Every creditor in the set shares pro rata, by what each is owed at enforcement, and interest stops at that moment |

The deployed Lien has one enforcement mode, TAKE: on breach the collateral passes **in kind** to the creditors. If the collateral holds enough of the loan's own currency to cover the debt, only what is owed is taken and the rest is kept for the borrower's side. Otherwise all of it passes, whatever it is worth, with no sale and no surplus returned. The deployed Lien has no sale mode.

The claim is narrow: the mechanical core of secured credit can be carried out by code that does what the law already sets out, so that the outcome on chain and the outcome a court would recognise can coincide. Where the law depends on judgement rather than mechanism, such as a stay in insolvency or the protections owed to consumers, the protocol does not pretend to supply it. That belongs to the legal agreements that refer to the on-chain record, and to the products built above the core.

**Parties need not know each other.** Terms are set by auction, the creditor's right is a transferable claim, the borrower's side is a transferable position, and the collateral, held under a rule, stands where a central counterparty's balance sheet would otherwise stand.

**Compliance lives at the edges, not in the core.** Counterparty-agnostic is not compliance-agnostic. A product can admit only verified participants to its own market. Those products write to the same record as fully open ones, and the core stays neutral.

## The commitments

Five commitments define Gavel. Most are properties of the deployed code that you can check; where one rests on a rule or on a power someone holds, this section says so.

**No protocol fees.** The building blocks have no fee parameter, and neither do the products built on them by the protocol's author. There is no treasury. The one payment to anyone other than the counterparties is the spot market's reward to whoever clears a batch. It comes out of the gap between a buyer's limit and the clearing price, so a buyer may pay more than the clearing price but never more than the limit they set. Any fee in the wider ecosystem belongs to an application built above the core, and is that application's choice.

**Non-custodial.** User funds live in the vault and move only by rules in its code. A contract can act on your balance only if you have approved it for your own account. Revoking that approval stops anything new; reservations and locks already made run to their end. No contract is upgradeable, so no one can change their logic. (Liquidation cover is the one product whose collateral sits outside the vault: it stays in the lending market it covers, in an account kept by the cover's contract that no one can manage at that market while the cover can still pay.)

**Permissionless use.** Anyone can call any contract, through any frontend or none, and anyone can read the record, run an indexer or clear a spot batch, without approval. A new product that only reads the record, or that creates obligations and liens with its own funds, needs no one's approval either. **A product that reserves or locks users' balances in the vault must first be admitted by the vault's owner**, as well as approved by each user. See [the base layer and what you trust](concepts/base-layer-and-trust.md).

**Oracle-free.** Prices and rates come from Gavel's own auctions. The blocks never read a price feed, and amounts and triggers in the products built by the protocol's author come from the protocol's own clearings, obligations and time. One product reads another venue's oracle: liquidation cover reads the covered lending market's own price, which is the price that market liquidates by, to refuse cover on a position already too close to liquidation. Where an instrument reads another protocol's on-chain state, the read is named when the obligation is created and published with it. The chain cannot tell a rate index from a price feed, so this is a rule the protocol's author keeps and anyone can check from the disclosures, not something the blocks enforce. Anyone may build products that use a price feed; that choice affects only the parties who make it.

**Open composition.** The products the protocol's author publishes are reference implementations, not the only ones. Others can build alternatives on the same blocks, and the protocol charges them nothing.

The emergency powers that exist beside these commitments, who holds them, and what they cannot do are set out in [the base layer and what you trust](concepts/base-layer-and-trust.md).

## Where it stops

Gavel does only what agreement can do. Facts it cannot witness come from parties you choose to trust. Judgement stays with courts and with the agreements above the record. It enforces only against what was pledged.

It is deliberately not:

- **a currency issuer.** Users bring existing tokens; Gavel records what they do with them.
- **a dispute-resolution system.** Contracts execute as written. No one can reverse a transaction or force a party to do what it did not agree to.
- **a prediction platform.** The protocol witnesses no real-world outcomes itself. An instrument can take one only from a source its parties chose to trust.
- **a DAO with a governance token.** BOND carries no governance, fee or revenue right, and there is no voting and no treasury.
- **an identity system.** To the protocol, an address is the whole of identity. Identity-aware products can be built above it.

## Why it is built this way

Infrastructure that rivals can share must be trustworthy without trusting its operator. A record with no fees, rules that cannot be changed under its users, and powers held over it that are narrow, public and unable to move anyone's funds is the kind that two competing institutions, or an institution and an open market, can both adopt. That is the design commitment: the infrastructure on which people make and keep promises should be extracted from human discretion, and should be honest about where agreement ends.

## Next

- [Concepts overview](concepts/overview.md): the layers, the blocks and one loan from start to finish.
- [Deployments](deployments.md): what is on the testnet, and where.
