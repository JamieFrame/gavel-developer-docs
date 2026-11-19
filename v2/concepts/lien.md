# The Lien

The Lien binds one lock of collateral in the vault to one set of obligations. If the set is paid, the collateral goes back. If the set is breached, anyone may pay it off, and anyone may enforce it, passing the collateral in kind to the creditors. It uses no price, no venue and no keeper, and adds no delay.

One `Lien` contract serves every set on the testnet's `Obligation`. Its address is on the [deployments](../deployments.md) page.

## Creating a lien

A product creates a lien in the same transaction as the set it secures:

1. `obligation.createSet(...)`, naming the Lien as the set's `lienContract`;
2. `vault.transferLoanOwnership(lockId, lien)`, handing the collateral lock to the Lien;
3. `lien.createLien({vault, lockId, setId}, "")`.

`createLien` checks that the caller created the set and handed the lock over; that the lock is in the set's vault and is not already used; and that the lock's depositor is the set's obligor and still holds the obligor position. It then binds the set (`obligation.bindLien`), after which the set is **secured**.

A lock serves at most one lien, and a set has at most one.

## Four ways a lien ends

| Function | Who may call | When | What happens |
|---|---|---|---|
| `release(lienId)` | Anyone | The set is `DISCHARGED` (paid) | The whole lock is credited to the obligor position's holder |
| `payoff(lienId, maxAmount, source)` | Anyone | The set is `BREACHED` and the lien not yet enforced | The payer pays exactly the debt now; the whole lock is credited to the position's holder. The breach stands on the record |
| `enforce(lienId)` | Anyone | The set is `BREACHED` | **TAKE**: the collateral passes in kind to the creditors (below) |
| `claim(lienId)` | The obligor position's holder | After any of the three, if anything was credited | Moves the credit to the caller's vault account and burns the position |

Nothing is pushed. Collateral returned to the debtor's side is held in the Lien as **credit attached to the obligor position**, and only whoever holds the position when they call `claim` receives it. If the position is sold, the credit goes with it. A borrower who has repaid therefore sends two transactions: `release`, then `claim`.

Once a set is breached, `payoff` and `enforce` are both open to anyone, and the first to land wins. Only a payment before the grace period ends is sure to keep the collateral.

`payoff` by `VAULT` needs the payer to have approved the Lien as a consumer; by `WALLET`, to have approved the Lien on the token. `maxAmount` caps what the payer will pay (`PayoffAboveMax` if the debt has grown past it).

## Enforcement: TAKE

The deployed Lien has one enforcement mode, TAKE. In one transaction, with `D` the set's debt at that moment:

1. **If there is no debt**, nothing is taken; the whole lock becomes credit for the position's holder.
2. **If the lock holds at least `D` of the set's own payment asset** (the *cash leg*), only `D` of it is taken. Everything else, the other legs included, becomes credit for the position's holder.
3. **Otherwise every leg is taken**, the payment asset included, whatever the collateral is worth.

What is taken goes to Obligation, which credits it to the set's obligations pro rata by what each was owed at enforcement, and then to each obligation's claim units. Interest stops at enforcement. Holders `collect` each asset as they would a payment.

**What TAKE means for value.** Under step 3 the creditors receive the collateral in kind, and the borrower receives no surplus, even if the collateral is worth far more than the debt; the Lien has no price with which to tell. No shortfall is recorded either: appropriation ends the debt. The deployed Lien has no sale mode.

**Cross-default and accrued value.** For a set of several obligations, the debt at breach counts obligations not yet due at their accrued value: a default never charges interest not yet earned. A single-obligation loan, such as the bullet loan, is unaffected.

## Bounded resolution

A lien always resolves within bounds fixed when it is made. From the set's latest due date plus its grace, pause-adjusted, anyone can call `markBreached` on Obligation and then `enforce`, with no dependency on a price, a venue, a keeper or the owner. A lien on a contingent obligation holds its lock while the obligation is pending, releases it if the obligation lapses or resolves at zero, and is enforced at the resolved amount.

## Pause and emergencies

The Lien's owner can pause it. **A pause stops only `createLien`.** Release, payoff, enforcement and claim keep working; the Lien reads Obligation's pause clock, so its deadlines move with Obligation's.

If the vault's owner removes consumers from the whitelist, release, enforcement and claim still work. Only `payoff` by `VAULT` stops; `payoff` by `WALLET` continues.

## Reading a lien

| View | Returns |
|---|---|
| `getLien(lienId)` | Its vault, lock, set, payment asset and status (`ACTIVE`, `RELEASED`, `PAID_OFF`, `ENFORCED`) |
| `lienOfSet(setId)`, `lienOfLock(vault, lockId)` | The lien securing a set or holding a lock |
| `creditOf(lienId, key)`, `creditKeys(lienId)` | What the position's holder can claim |
| `obligation()` | The Obligation this Lien serves |

A set can name any address as its lien contract, so a set on another Obligation could name this Lien. When you reach a Lien through `obligation.lienOf(setId)`, check that `lien.obligation()` is the Obligation you started from.

Events: `LienCreated`, `LienReleased`, `LienPaidOff` (the net amount paid), `LienEnforced`, `CollateralTaken` (per asset taken), `CreditRecorded` (per asset credited), `Claimed`, `PauseStarted`, `PauseEnded`.

## Things to know

- **Hand over and create in one transaction.** A lock handed to the Lien with no lien created stays there until the contract that handed it over creates one.
- **A position holder that cannot call `claim`** (a contract without that ability, or an address no one controls) strands its credit.
- **A position's value rests on its vault.** A set may name any vault. Before valuing a position from the Lien's views, satisfy yourself that the vault is the one you expect.

## Next

- [Claims and obligor positions](claims-and-positions.md): who is paid and who gets the collateral back.
- [The base layer and what you trust](base-layer-and-trust.md): the limits on what anyone can do to a lien.
