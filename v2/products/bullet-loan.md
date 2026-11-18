# The bullet loan

A bullet loan is one obligation: the borrower receives the principal now and repays one fixed amount, principal and interest together, at the due date. It is secured by a TAKE lien on the borrower's collateral, with a seven-day grace after the due date. The terms are found by auction, opened four ways: a borrower **requests** a loan from one lender or from many, or a lender **offers** a loan to one borrower or to many.

Two contracts run it: `BulletLoanRequest` (the borrower's openings) and `BulletLoanOffer` (the lender's), each with the linked libraries `BulletOpening` and `BulletSettlement`. The openings to one run on `SingleAuction`, the openings to many on `GroupAuction`; settlement is on the core. Addresses are on the [deployments](../deployments.md) page.

## The four openings

| Opening | Opened by | Engine | Who bids | What a bid is | Better is |
|---|---|---|---|---|---|
| **Request to one** | Borrower, `openRequest(borrower, terms, false)` | `SingleAuction` | Lenders, `bidRequest` | A repayment for the whole amount | Lower |
| **Request to many** | Borrower, `openRequest(borrower, terms, true)` | `GroupAuction` | Lenders, `bidRequest` | A repayment for a quantity lent | Lower |
| **Offer to one** | Lender, `openOffer(lender, terms, false)` | `SingleAuction` | Borrowers, `bidOffer` | A repayment for a loan size in the lender's range | Higher |
| **Offer to many** | Lender, `openOffer(lender, terms, true)` | `GroupAuction` | Borrowers, `bidOffer` | As offer to one | Higher |

Every opening and bid names its principal (`borrower` or `lender`) as the first argument. The principal or an operator it approved with `setOperatorApproval` on the product may call. The principal, never the operator, is the opener, the bidder and the owner of the escrow. Anyone but the opener may bid; the engine refuses the opener by address.

Each bid also carries `expectedTerms`, the hash `keccak256(abi.encode(terms, many))` of the opening's terms. A bid against terms other than those it expects reverts `TermsChanged`.

### Requests

The borrower names the loan token, the `amount`, the `term`, the auction's `duration`, the `maxRepayment` for the whole amount, and the collateral as the vault's lock parameters (any basket the vault can lock, with the borrower as depositor). A request to one also names a `bidStep`, an absolute repayment step of at most 10% of the amount; a request to many names a `minAmount`, the least total it will accept.

At opening the collateral is locked in the vault under the product's key, drawing on the borrower's free balance and pulling any shortfall from the wallet.

Each lender bids the repayment it would accept. The repayment may not be less than the quantity lent (no negative interest), nor above the borrower's maximum rate. The lender's principal is reserved in the vault, under the bid's own entry, before the engine records the bid.

- **To one,** every bid is for the whole amount and must beat the standing bid by the step. A lender who bids again keeps its reservation. The displaced lender's reservation is released in the same transaction.
- **To many,** each lender places one bid, of at least `⌈amount / 25⌉`, with an optional minimum fill (`minFill`). A later bid amends it, and may only improve: a price no worse, a quantity no smaller, a minimum fill no larger. A larger quantity is reserved again in the same call. Bids are ranked by price, then larger first; the step is 1 basis point.

A bid at zero interest (a repayment equal to the amount) ends a request early: to one, at once; to many, once bids at that price cover the amount.

### Offers

The lender names the loan token, the `principal` it puts up, the range of loan sizes (`minLoan` to `maxLoan`), the collateral as one ERC-20, a `loanToCollateral` ratio, a `floorPrice` and optionally a `ceilingPrice`, the `term` and the `duration`. An offer to many also names `maxFills`, at most 25 winners; an offer to one's principal equals its `maxLoan`.

The floor and ceiling are prices, not rates: repayment per unit lent, scaled by 10^18, so 10^18 is zero interest. At opening the lender's principal is reserved in the vault.

Each borrower bids a repayment for a loan size in the range. The repayment must exceed the loan size (an offer never clears at zero interest), and its price must reach the floor. The collateral is locked from the borrower at `⌊loanSize × 10^18 / loanToCollateral⌋` before the engine records the bid.

- **To one,** every bid is a new entry with a new lock, and the displaced entry's lock is released in the same transaction, the bidder's own earlier one included. A borrower raising its own bid therefore needs both locks' worth of collateral for that transaction.
- **To many,** each borrower places one bid, amended as for a request to many; a change of size releases the lock and locks again.

A bid at the ceiling, if there is one, ends the offer early, on the same rule as a request.

## Settlement

When the auction ends, **anyone** may call `finalise(loanId, maxSteps)`. Nothing is paid for calling it.

**The openings to one** settle in that call. The product closes the auction, checks the result against its own records, and creates the loan on the core in one transaction:

1. Obligation creates a set of one obligation: the principal, the interest (the winning repayment less the principal), due `term` after settlement, prepayment `FULL`, grace 7 days. The borrower receives the obligor position; the lender receives **1,000,000 claim units**.
2. The borrower's collateral lock is handed to the Lien, which binds it to the set.
3. The lender's reservation is paid into the borrower's free balance in the vault.

For an offer to one, the loan is the winner's own size; the rest of the lender's principal is released.

**The openings to many** close in pages. While the auction is `OPEN`, each `finalise` runs one page of at most `maxSteps` (capped at 200) and records the winners it finds, moving no value. When the last page clears the auction, the loan is `CLOSED`. Every winner pays or receives **one uniform rate: the marginal winner's exact rate**, the last winner the close admitted.

- **Request to many.** One more `finalise` settles one set, with every winning lender as a holder. Each lender's claim units equal its fill, in base units of the loan token, so the units total the amount lent. The total repayment is the amount lent at the marginal rate. If the bids filled less than the amount but at least `minAmount`, the loan is for what was filled and each leg of the collateral is reduced in proportion, rounded up; the rest returns to the borrower. Below `minAmount` there is no sale and the loan ends at once.
- **Offer to many.** Each winner's loan is a set of its own. Anyone calls `executeFill(loanId, entryId)` for each winner: the borrower's lock is reduced to `⌊fill × 10^18 / loanToCollateral⌋` if partly filled, and a set is created with the lender holding 1,000,000 claim units, repaying the fill at the marginal rate. When the last winner resolves, the lender's unused principal is released.

## Cancellation, deadlines and exits

**Cancel.** The opener, or its operator, may `cancel(loanId)` while no bid has ever been made. The opener's escrow is released.

**The settlement deadline** is the auction's end plus 3 days (plus a further 3 days for an offer to many), moved later by any pause. At or past it, `finalise` and `executeFill` revert `Stalled`, and the loan can no longer settle. This is final.

**Two exits, open to anyone, never paused, never dependent on an engine call:**

| Function | When | What it returns |
|---|---|---|
| `release(entryId)` | A bid has lost (its loan has left `OPEN` without it), the loan has ended, or the deadline has passed and the loan was not settled | That bidder's escrow: a lender's reservation, a borrower's lock |
| `expire(loanId)` | The deadline has passed, or the auction ended with no escrowed bid | The opener's escrow: a request's collateral lock, an offer's reserved principal |

An offer to many past its deadline with winners not executed releases the lender's residual through `expire`; each such winner releases its own lock, and the last ends the loan.

**After settlement** the loan is an ordinary set on the core. The borrower, or anyone, repays with `obligation.pay`; prepayment is allowed, but the full repayment is owed whenever it is paid. If it is still unpaid 7 days after the due date (pause-adjusted), anyone may call `markBreached`, then pay it off or enforce the lien. See [Obligation](../concepts/obligation.md) and [the Lien](../concepts/lien.md).

## Parameters

Code constants, the same on both contracts:

| Constant | Value |
|---|---|
| `MIN_AUCTION_DURATION`, `MAX_AUCTION_DURATION` | 1 day, 7 days. A hard close: the end never extends, except by a pause |
| `MIN_LOAN_DURATION`, `MAX_LOAN_DURATION` | 7 days, 30 × 365 days |
| `PRODUCT_GRACE` | 7 days |
| `CLAIM_UNITS` | 1,000,000 per set with one lender |
| `MAX_BID_STEP_BPS` | 1,000 (a request to one's step, at most 10% of the amount) |
| `SETTLE_PERIOD`, `FILL_WINDOW` | 3 days, 3 days |
| `PAGE_STEPS` | 200 |
| `MAX_REQUEST_WINNERS`, `MAX_OFFER_FILLS` | 25, 25 |
| `MAX_REPAYMENT` | `type(uint256).max / 10^18` |

Prepayment is always `FULL` with no charge. Escrow is in the vault only: a lender deposits to the vault before bidding. There is no fee and no keeper reward.

## Pause

Each contract's owner can `pause()` it; on the testnet that is the deploying account, and ownership cannot be renounced. A pause stops openings, bids, `cancel`, `finalise` and `executeFill`. It never stops `release` or `expire`, and it does not touch a loan already made, which lives on the core.

Each pause is recorded with its start and length (`pauseCount`, `pauseAt`). At the loan's next call, the whole length of each pause that began before the auction's end is pushed into the engine auction, and each that began before the deadline moves the deadline, so bidding never ends during a pause. Beyond the engine's 30-day cap on shifts, the auction may end during a long pause; the deadline still moves.

## Things to know

- **Rounding.** Bids are exact repayments; the engine ranks them as prices at 10^18, rounded against the bidder. A group winner can pay the marginal rate when its own differs by less than 10^-18 per unit.
- **The hard close.** A bid in the last second cannot be answered.
- **Groups.** A page whose result fails the product's checks ends the loan, and every escrow is released. A bid skipped because its minimum fill does not fit can let a lower-ranked bid win.
- **A pending engine record.** When a loan ends through an exit, the engine's own call is tried and may fail, leaving the engine's auction un-expired. Funds are unaffected: exits read the product's records.
- **Two contracts, two id spaces.** Loans and entries share one counter within each contract, so an id names one record there; the same number in the other contract is a different record.

## Reading it

| View | Returns |
|---|---|
| `loanOf(loanId)` | Opener, opening, state (`OPEN`, `CLOSED`, `ORIGINATED`, `ENDED`), engine auction id, deadline as last stored, set id |
| `deadlineOf(loanId)` | The auction's end and the settlement deadline, with every pause added; `loanOf`'s stored deadline does not include them |
| `entryOf(entryId)` | Loan, state (`LIVE`, `WON`, `SETTLED`, `RELEASED`), owner, size, fill, exact repayment |
| `liveEntryOf(loanId, account)`, `winnersOf(loanId)` | An account's escrowed entry; the winners in page order |
| `originationOf(ref)` | Set, obligation, principal, repayment, due date and lien of a settled loan: `ref` is a request's loan id or an offer's entry id |
| `bondOrigination(ref)`, `originator()` | The origination record BOND reads |

Events: `LoanOpened`, `EntryRecorded`, `EntryReleased`, `LoanOriginated` (the set and lien), `LoanEnded`, `PauseStarted`, `PauseEnded`; the engines emit the shared auction events with the product as controller.

**BOND** reads both contracts as sources, through `bondOrigination` and `originator`, to register secured loans; see [BOND](bond.md).

## Next

- [The secondary market](secondary-market.md): selling claim units or an obligor position.
- [The engines](../concepts/engines.md): how `SingleAuction` and `GroupAuction` find the price.
- [Writing](../build/writing.md): calling the openings from your own code.
