# Composing a product

Every Gavel V2 product is a thin contract over the same blocks, and anyone can build one. This page describes the pattern the deployed products follow, what the core checks and what it leaves to the product, how to write a module, and admission to the shared vault.

The contracts are deployed on Arbitrum Sepolia and are **not audited**. Signatures are in the [contracts reference](contracts-reference.md).

## The pattern

A product does three things.

1. **Discover terms on an engine.** It opens an auction on `SingleAuction`, `GroupAuction`, `PostedOffer`, `BatchAuction` or `ClockAuction`, as its **controller**, naming the principal it runs the auction for. Bidders bid through the product, never the engine directly. See [the engines](../concepts/engines.md).
2. **Escrow in the vault under its own consumer key.** Each bid's backing is a reservation (`vault.reserve(user, asset, amount, contextId)`) or a lock (`vault.lockCollateral(params)`) held by the product, under a context id the product chooses. Nothing leaves the vault while terms are found.
3. **Settle on the core in one transaction.** When the result is in, the product creates the debt, binds the security and pays out, in this order, with nothing called between the hand-over and the lien:

| Step | Call | What the core checks |
|---|---|---|
| 1 | `obligation.createSet(obligor, grace, vault, lienContract, releaseGate, obligations[])` | The terms' bounds (grace from 1 second to 365 days, at most 25 obligations and 200 holder-obligation pairs, dates within 30 years); the caller becomes the set's **creator** |
| 2 | `vault.transferLoanOwnership(lockId, lien)` | The caller owns the lock; the caller and the Lien are both on the vault's whitelist. The vault records the caller as the lock's previous owner |
| 3 | `lien.createLien({vault, lockId, setId}, "")` | The caller created the set and handed the lock over; the set names this vault and this Lien; the lock is not already used; its depositor is the set's obligor and still holds the obligor position; the set is not yet bound, paid or mixed in assets |
| 4 | `vault.settleReservation(lender, borrower, asset, contextId, amount)` | The caller holds the reservation and is on the whitelist |

The bullet loan does exactly this, and also checks that the principal disbursed equals the debt created. An unsecured product skips steps 2 and 3.

## Counterparts are named, never wired

No counterpart is rewired after deployment. Each choice is made once, by naming:

- **The product's own counterparts** are fixed in its constructor: the engines, the vault, Obligation and the Lien, as immutables. The deployed bullet loan refuses to deploy unless `lien.obligation()` is the Obligation it was given.
- **Each set's counterparts** are fixed at creation: its `vault`, its `lienContract`, its `releaseGate` (an address that may forgive a share of what is owed, or zero) and its `grace`. None can change.
- **Each contingent obligation's modules** are fixed at creation by `createContingentSet`: its resolver, its trigger target or exercise module, their data, and the bounds they answer within.

A faulty contract therefore harms only what named it. There is no registry and no setter; a replacement is a new deployment.

## Admission to the vault

The vault's owner keeps a whitelist of **consumers**: contracts that may reserve, lock and move users' balances. A product that does any of these must be on it, and each user must also approve it for their own account (`approveConsumer`). On the testnet the owner is the account that deployed the contracts; `owner()` returns it, and `setAuthorisedConsumer(consumer, true)` is the call that admits a contract.

| Needs admission | Needs none |
|---|---|
| `reserve`, `lockCollateral`, `enlargeLockFor` | Reading anything |
| `settleReservation`, `transferLoanOwnership` (both the caller and the new owner) | `createSet` and `createContingentSet`: anyone may create a set |
| `transfer` from another account's balance | `obligation.pay` and `lien.payoff` from a wallet |
| | `deposit`, `withdraw` and `enlargeLock` from the caller's own balance |

So **a product that only reads, or that creates sets with its own funds** (an unsecured set, paid from its own wallet), needs no admission. A product that escrows users' balances or locks collateral for a lien does.

Removal from the whitelist stops only new entries; see [the vault](../concepts/vault.md#exits-what-keeps-working).

## What the core leaves to the product

The core keeps its four promises for every user whatever a product does (see [the base layer and what you trust](../concepts/base-layer-and-trust.md)). Some checks it leaves to its callers, since an error harms only that product's users. The core does not check the following; a consumer of it does:

- **Release amounts.** `releaseReservation` returns the amount released, and returns zero without reverting when nothing was held under that key. A product reads the amount rather than assuming one.
- **Recipients.** A product never names the vault's own address as a recipient: a credit there can never be withdrawn.
- **Hand-over and lien in one contract.** The Lien accepts `createLien` only from the lock's previous owner, so the contract that calls `transferLoanOwnership` calls `createLien` itself; a router between the two breaks it.
- **Views inside a callback.** During a call to the vault, Obligation or the Lien (in a token's callback, or a call from a named vault) their views show the state before the call, and a product gates no decision on them. `createSet` runs no receiver callback; a module's `onCreate` runs inside `createContingentSet` after the set is stored, and can only read the core there.
- **A set's Lien and its Obligation.** `createSet` accepts any lien contract. A product that reaches a Lien through `lienOf(setId)` checks `lien.obligation()`, and that `getLien(lienId)` names the same set and vault, before relying on it.
- **A position's vault.** A set's vault can be any contract. A product that values or buys a position from the Lien's views accepts the vault `getLien` returns.
- **Obligation and the Lien at the named vault.** A product names only a vault where both are admitted; at any other vault, payments from vault balances, collection into a vault account and enforcement can revert.
- **Recipients at creation.** Holders and the obligor receive claim units and the position with no receiver hook and cannot refuse them. A product learns of a creation from balances, `ownerOfPosition` or its own records, and a product that receives units or a position itself can move or collect them.
- **Token kind.** The vault does not ask a token what it is. A product that takes an asset list from a caller checks each key's kind, or refuses ERC-721 legs.
- **Asset choice.** Which tokens a product offers is its own choice; a misbehaving token harms only the positions that hold it.

## What a product does around an engine

An engine holds nothing and checks little: it trusts its controller to escrow, to push pauses and to check results. Each product carries the following list itself; the deployed products are built to it.

**Routing**

- Each engine's address is fixed at deployment, never chosen per auction.
- Every bid is escrowed in the vault **before** the engine records it, under the product's own context id; nothing is recorded without escrow, and no escrow is taken without a record. Either failing reverts the transaction.
- A displaced, withdrawn or voided bid's escrow is released **in the same transaction**, from the product's own records.
- The bidder passed to the engine is the account whose escrow backs the bid: the principal of a delegated call, never its operator.
- Every input the product narrows is checked before the auction opens; a value outside the product's bounds never reaches the engine.
- A paged close is run to its end through an entry anyone may call, and a group's product moves no value on any page before the auction is closed.

**Checking every result**

- The engine id comes from the product's own record of the auction it opened, and the result's `controller` is the product itself.
- Every `ref` the engine returns is the product's own record, of that auction, held by that bidder.
- Each fill is within its escrow, and the fills sum to at most the quantity.
- On any failed check the product settles nothing (it refuses, it does not repair) and the auction ends through the product's exits. Where the engine writes records before returning a result (paged closes), the product checks inside a call to itself that reverts on failure, undoing the engine's records, and takes its exit only on its own refusal; any other revert, including running out of gas, is re-raised so no caller can force the exit.

**Exits**

- The product copies the auction's latest end when it opens, and measures its deadlines from it, pause-adjusted on its own clock.
- Every escrow can be released by a permissionless expiry path that reads only the product's records. The engine's `expire` is tried afterwards, and its failure does not stop the release.
- Exits are never paused and **never depend on an engine call** that could revert.
- An auction someone else has already closed is treated as closed: the product reads the result rather than calling `close`.
- A paged close that stalls past the product's stall period is terminal: no further page runs and nothing settles, and every bidder releases from the product's records.

**The pause shift.** Engines have no pause. A paused product pushes the paused time into each auction **lazily**, at its next call: before every engine call that bids, takes or ends an open auction (and before `cancel` and `reprice`), it pushes the whole length of each pause that began before the auction's end, and never more; it never pushes before an exit. Until that push, the engine's own end is stale, so the product's views add the unpushed time. An engine caps the total shift per auction at 30 days.

## Writing a module

A contingent obligation names its modules at creation. Obligation keeps the bounds; a module only answers questions, by `staticcall` (apart from `onCreate` at creation), and can never write Obligation's records or move value through Obligation. A module implements `IResolver`, and an exercise module also `IExercise`:

| Function | Asked | Answer |
|---|---|---|
| `amountOf(obligationId, resolverData)` | At `resolve`, once | The amount owed. Clamped to the minimum and maximum fixed at creation |
| `conditionMet(obligationId, triggerData)` | At `fire`, for a `RESOLVER` trigger | Whether the obligation has come into force |
| `exerciseAllowed(obligationId, exerciser, data)` | At `exercise`, if an exercise module is named | Whether this exercise is valid now |
| `onCreate(obligationId, data)` | At creation, once per role the contract is named in, after the set is active | Sets up any starting state; a module with none does nothing. A revert reverts the creation |
| `targetsOf(data)` | At creation | The outside contract the module reads, at most one |

**The guard.** Each call runs with a fixed allowance of 100,000 gas. Before calling, Obligation checks that it can pass the whole allowance, and otherwise reverts the transaction (`InsufficientResolverGas`), so no caller can starve a module into failing. A reply is accepted only if it has exactly the expected length. On failure:

- `amountOf` failing, or answering out of shape, records the **fallback** bound (the minimum or the maximum, as fixed at creation), **for good**;
- `conditionMet` failing, or answering anything but true, does not fire;
- `exerciseAllowed` failing, or answering anything but true, refuses the exercise (`ExerciseRefused`);
- `onCreate` or `targetsOf` failing reverts the creation; `targetsOf` returning more than one address reverts `TooManyTargets`.

**A module never fails because its data is not ready.** Because a failed `amountOf` fixes the fallback permanently, a module answers, from the moment it can be asked (`resolveAt` for an obligation in force from creation, or the moment it fires), from data fixed at that moment. The deployed `SpotReferenceResolver` is the pattern: anyone may fix a reference price once, after its reference time, from the spot market's recorded clearings, paying the gas themselves; `conditionMet` and `amountOf` then read one stored record, well within the allowance, and answer the same whoever calls and whenever.

**Disclosure.** `targetsOf` names the contract outside Gavel's own that the module reads. Obligation calls it at creation and lists the result, one entry per role a module is named in, in the `ContingentCreated` event; an empty answer records the module itself. A reader sees every dependency of an obligation before taking it on. A module harms only the obligations that named it.

A `STATE` trigger needs no module: it reads `breachOf` from the contract it names.

## The rules the published contracts follow

The protocol's author builds its own products to these rules, layer by layer:

- **The four promises** (conservation, isolation, exits, resolution) hold for every user whatever anyone else chooses, and no contract relies on a layer above it to keep them.
- **The narrowest guard.** A risk is closed by a bound or check that refuses no one acting within the promises, never by an allowlist; a risk left open is disclosed.
- **Named at creation, never rewired; not upgradeable.**
- **No admin and no fee** beyond the emergency surface (pausing, and the vault's whitelist).
- **Credits within the vault, refunds pulled**; **exits survive** every pause, de-authorisation and other party's choice.
- **Bounded.** Every loop has a constant bound or runs over the caller's own arrays; every call that can stop a resolution has a gas cap and a fallback; every resolution has a backstop.
- **No price feed**, and every read of another protocol's state disclosed per obligation, by target.
- **No reversing the past, no discretion no one agreed to, no indefinite stays** where a resolution was promised.
- **Delegation keeps the user as principal**, approved per contract.
- **Expected values** on calls whose outcome depends on state the caller saw (the bullet loan's `expectedTerms`, for example), and **pause-adjusted deadlines** so that a pause neither causes a default nor lets an offer lapse.

A product chooses its own assets, counterparties and bounds, keeps them in itself rather than in the core, gives every escrow an expiry path, and holds user value only in the vault except where its page discloses otherwise. A module needs no privileged access or custody, resolves within bounds fixed at creation, and harms only what named it.

## Third-party products

The **base layer** is the core, the venues and the products published by the protocol's author that meet its rule; see [the base layer and what you trust](../concepts/base-layer-and-trust.md). A product anyone else builds is a third-party product, whatever rules it follows: it is built on the same blocks, and the core's promises still protect every user from it, but the base layer's guarantee does not extend to it. The same holds for any product that reads a price feed, charges a fee or has an operator of its own.

## Next

- [Writing](writing.md): the calls, approvals and asset keys in practice.
- [The engines](../concepts/engines.md), [the vault](../concepts/vault.md), [Obligation](../concepts/obligation.md), [the Lien](../concepts/lien.md).
- [The bullet loan](../products/bullet-loan.md): the pattern, deployed.
