# Obligation

Obligation is the register of debts: who owes whom what, and when, what has been paid, what has been forgiven, and how each debt ended. It holds what creditors are owed in a vault account and pays it out when they collect. It knows no prices and no collateral rules, and it never calls a lien.

One `Obligation` contract serves every product on the testnet. Its logic is split across two linked libraries, `ObligationLogic` and `ModuleLogic`, to keep it within the contract size limit. Its address is on the [deployments](../deployments.md) page.

## Sets and obligations

A **set** is one or more obligations created by one call, owed by one obligor, ending together. A bullet loan is a set of one obligation; an instalment loan would be a set of several.

When a set is created it fixes, for good:

| Field | Meaning |
|---|---|
| `obligor` | Who owes it. The obligor receives the set's **obligor position**, an ERC-721 whose holder owes the set from then on |
| `vault` | The vault holding the set's money and collateral |
| `lienContract` | The only contract that may secure and settle the set; zero if it is unsecured |
| `releaseGate` | An address that may forgive a share of what is owed (below); zero for none |
| `grace` | Seconds after the due date before an unpaid obligation can be marked breached: from 1 second to 365 days |

Each obligation in it fixes:

| Field | Meaning |
|---|---|
| `asset` | The ERC-20 it is payable in (a vault key) |
| `principal`, `interest` | The amounts |
| `accrualStart`, `dueAt` | When interest starts and when the whole is due, at most 30 years ahead |
| `prepayment` | Whether it can be paid before `dueAt`: `NONE`, `FULL` (the whole amount, interest included) or `ACCRUED` (what has accrued, plus a charge up to a cap) |
| `holders`, `units` | Who receives the obligation's **claim units**, and how many each |

A set holds at most 25 obligations and 200 holder-obligation pairs. Its terms never change after creation; no owner function can alter them.

Anyone may create a set by calling `createSet(obligor, grace, vault, lienContract, releaseGate, obligations[])` and becomes its **creator**. In practice a product creates sets with the terms its auction found. See [composing a product](../build/composing-a-product.md).

## Value at a moment

Interest accrues in a straight line between `accrualStart` and `dueAt`:

| At time `t` | Value |
|---|---|
| Before `accrualStart` | `principal` |
| Between the two | `principal + interest × (t − accrualStart) / (dueAt − accrualStart)` |
| At or after `dueAt` | `principal + interest` |

What is **owed** is that value less what has been paid, scaled down by anything forgiven. Rounding favours the obligor. Every time is read **pause-adjusted**: if Obligation has been paused since the set was created, each deadline moves later by the time paused.

`valueAt(id, t)`, `outstanding(id)` (owed now) and `payoff(id)` (what would pay it in full now) return these figures. `debtAt(setId, t)` sums them over a set.

## The life of an obligation

| Status | How it gets there |
|---|---|
| `OPEN` | Created |
| `DISCHARGED` | Paid in full |
| `BREACHED` | Still unpaid when `dueAt + grace` has passed, and someone called `markBreached` |
| `PAID_LATE` | Paid in full after breach (unsecured sets only) |
| `SETTLED` | Its set was settled by its lien after breach |
| `RELEASED` | Its creditors forgave all of it |

### Paying

`pay(obligationId, amount, source)` may be called by anyone, at any time, and is never stopped by a pause. A payment gives the payer no rights; it simply reduces the debt. `source` is `VAULT` (from the payer's free balance, having approved Obligation as a consumer) or `WALLET` (pulled from the payer's wallet, having approved Obligation on the token).

- `amount` is a ceiling: only the payoff is charged, so a payment is not refused because a little interest accrued between signing and inclusion.
- Before `dueAt`, the obligation's `prepayment` rule applies: `NONE` refuses payment, `FULL` charges the full amount, `ACCRUED` charges what has accrued plus the charge.
- **On a secured set, a payment must pay the whole payoff** (`MustPayWhole`). Part-payments are possible only on unsecured sets.
- **If the vault's owner de-authorises Obligation**, payment by `VAULT` stops; payment by `WALLET` continues.

### Breach

`markBreached(obligationId)` may be called by anyone once `dueAt + grace` has passed (pause-adjusted) and the obligation is still `OPEN`. **A breach of any obligation breaches its whole set** (cross-default).

- **A secured set** in breach takes no more payment through `pay` and cannot be cured. The obligor's way out is the Lien's `payoff`, until someone enforces the lien. See [the Lien](lien.md).
- **An unsecured set** in breach still takes late payment, which ends as `PAID_LATE`. A late payment never reverses the breach.

### Collecting and redeeming

Payments, and collateral passed in kind on enforcement, are credited to an obligation's claim units pro rata. Each holder collects their share:

| Function | Who may call | What it does |
|---|---|---|
| `collect(id, key, destination, toVault)` | A holder, or a former holder with credit left | Pays out what has accrued to the caller in one asset, to a wallet or to a vault account |
| `collectFor(holder, id, key)` | An operator the holder approved with `setOperatorApproval` | The same, always into the holder's vault account |
| `redeem(id, destination, toVault)` | A holder, once the obligation is final | Collects every asset and burns the caller's units |
| `close(setId)` | Anyone | Burns the obligor position of an unsecured set that has ended |

Nothing is pushed to a holder. A token that blocks an address therefore cannot stop a repayment: the borrower pays Obligation, and each holder collects to a destination of their choice. Credit follows the units: when units change hands, what they had already earned stays with the seller.

### Forgiving

A creditor can give up what it is owed. Release moves no asset, only lowers a debt, and is never stopped by a pause.

| Function | Who may call | What it does |
|---|---|---|
| `release(id, units)` | A holder | Burns the caller's units; the debt falls in proportion. Each remaining unit's entitlement is unchanged |
| `releaseFor(holder, id, units)` | An operator approved with `setReleaseApproval` (approval to collect is not enough) | The same, for the holder |
| `releaseShare(id, shareWad)` | The set's `releaseGate` only | Every unit gives up the same share of what is still owed, with no units burned |

A forgiven default never reads as repaid: `reportedShortfall(id)` includes anything forgiven after breach.

## Modules: amounts, triggers and exercise

Some debts are not fully known when they are made: insurance pays only if a loan defaults; a future's amount is fixed later from a recorded price; an option comes into force only if its holder elects. Obligation handles these through **modules**, named per obligation when it is created with `createContingentSet`. Obligation keeps the bounds; each module answers only within them, and can never write Obligation's records or move value.

| Module | What it answers | Called by |
|---|---|---|
| **M1, resolved amount** | How much is owed, clamped between a minimum and a maximum fixed at creation | `resolve` |
| **M2, trigger** | Whether the obligation has come into force, within a window | `fire` |
| **M2s, another set's breach** (`STATE` trigger) | Whether a named set has breached within the window, read from the `breachOf` view of a contract named at creation: Obligation itself, or an adapter that answers in the same shape | `fire`, `lapse` |
| **M3, exercise** | Whether an election to exercise is valid | `exercise` |
| **M4, set-up and disclosure** | Starting state, and which outside contract the module reads | At creation |

A contingent set has exactly one obligation. Its trigger kind is one of:

| Kind | In force when |
|---|---|
| `UNTRIGGERED` | From creation; only its amount waits to be resolved |
| `STATE` | A named set breaches within the window |
| `RESOLVER` | A named module says so within the window |
| `EXERCISE_HOLDER` | Whoever holds every outstanding unit exercises within the window |
| `EXERCISE_OBLIGOR` | Whoever holds the obligor position exercises within the window |

Its module state runs `PENDING` → `FIRED` → `RESOLVED`, or `PENDING` → `LAPSED` if the window ends with no trigger. An `UNTRIGGERED` obligation starts at `FIRED`. `fire`, `exercise`, `resolve` and `lapse` are callable by anyone (`exercise` by the exerciser) and are never stopped by a pause. A lapse, or a resolution at zero, discharges the obligation with no value moved.

**Until it resolves, an obligation cannot be paid or breached** (`NotResolved`), and every plain reading of it (`getObligation`, `outstanding`, `debtAt`) shows its **maximum**. To tell a contingent obligation from a plain one, read `contingentStateOf(id)`: `NONE` means plain. `isContingentSet(setId)` does the same per set.

**Each module call is guarded.** It runs with a fixed gas allowance (100,000), and the whole call reverts if the caller did not supply enough, so no one can starve a module into its fallback. A resolver that fails or answers out of shape gives the fallback bound (minimum or maximum, as fixed at creation). A trigger that fails does not fire.

**Every outside read is disclosed when the obligation is created.** The `ContingentCreated` event names each contract the obligation's modules read, so a reader can see an obligation's dependencies before taking it on. A module harms only the obligations that named it.

## Pause

Obligation's owner can pause it. **A pause stops only new sets** (`createSet`, `createContingentSet`). Paying, breach, collection, redemption, release, the Lien's calls and every module call keep working, and every deadline moves later by the time paused, so a pause never causes a default. `pausedTotal()` returns the pause clock.

## Reading a set

| View | Returns |
|---|---|
| `getSet(setId)`, `getObligation(id)` | The stored fields |
| `vaultOf`, `lienOf`, `creatorOf`, `obligorOf`, `isBound` | A set's counterparts |
| `ownerOfPosition(setId)` | Who holds the obligor position now, or zero once it is burned |
| `positionState(setId)` | `ACTIVE`, `IN_ENFORCEMENT`, `BREACHED_UNSECURED`, `CREDIT_PENDING` or `FINISHED`: what the position's holder has |
| `breachOf(setId)` | When the set breached, and the pause clock then |
| `accrued(id, holder, key)`, `creditKeys(id)`, `unitsOutstanding(id)` | Credit and supply |
| `contingentStateOf`, `triggerOf`, `windowOf`, `amountBoundsOf`, `resolvedAmountOf`, `resolverOf` | A contingent obligation's terms and state |

Events, in the order a set lives them: `SetCreated`, `ObligationCreated`, `Paid`, `Discharged`, `SetDischarged`, `Breached`, `LienBound`, `SetSettled`, `ShortfallRecorded`, `Collected`, `Redeemed`, `PaidLate`, `Released`, `ShareReleased`, `ObligationReleased`, `ShortfallReduced`, `PositionBurned`; for contingent obligations `ContingentCreated`, `Fired`, `Exercised`, `Resolved`, `Lapsed`; and the standard ERC-1155 events for claim units. The full list, with errors, is in the [contracts reference](../build/contracts-reference.md).

## Things to know

- **Anyone can name you.** Anyone may create a set naming any address as a holder or as the obligor, and no code of the recipient runs at creation, so a contract cannot refuse it. A reader that attributes debts to an obligor should check `creatorOf(setId)` and count only sets made by a product it trusts.
- **One vault and one asset per secured set.** A set names one vault, and a secured set's obligations must all be payable in the same asset.
- **Bind in the same transaction.** Anyone can stop a later `bindLien` by paying one base unit first. Products create the set and its lien together.
- **Exercise needs the whole claim.** Under `EXERCISE_HOLDER` only an account holding every outstanding unit can exercise.

## Next

- [Claims and obligor positions](claims-and-positions.md): the two tokens a set creates.
- [The Lien](lien.md): how a set is secured.
