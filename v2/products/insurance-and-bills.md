# Insurance and bills

Insurance on Gavel is an underwriter's promise to pay if someone else's loan is not repaid. The underwriter posts an offer at a flat premium; each policy taken from it creates a **cover**: a contingent obligation on `Obligation`, owed by the underwriter, that comes into force if the insured loan breaches, and is backed in full by the underwriter's cash under the Lien. The **bill** packages insured loan claims due in the same week into one ERC-20 per week, which trades on its own spot market.

The contracts are `CoverOrchestrator` (with its linked libraries `CoverIssuance` and `CoverSettlement`), `ContingentBondAdapter`, and `BillPackager`. Each bill series has a `BillToken`, created at the series' first deposit, and a `SpotMarket`. Offers run on the `PostedOffer` engine; covers and loans live on the core. Addresses are on the [deployments](../deployments.md) page.

## Underwriting: the offer

An underwriter calls `createOffer(terms)`. The offer is opened on `PostedOffer`, open to anyone, takeable in parts, never repriced. Its `capacity` is reserved from the underwriter's vault balance, so the underwriter must first have approved `CoverOrchestrator` as a consumer.

| Term | Meaning | Bounds (code constants) |
|---|---|---|
| `asset` | The settlement asset: cover, margin and premium | An ERC-20 key |
| `capacity` | Total cover the offer may write | Above zero, at most `2^128 − 1` |
| `premiumBps` | Premium per policy, in basis points of the cover | 1 to 10,000 |
| `coverGrace` | Grace on the cover's own payment | 1 second to 365 days |
| `payDelay` | Time from the cover coming into force to its due date | 1 second to 30 days |
| `collateralKey`, `minCollateralWad` | The one collateral asset an insured loan must have, and its least amount per unit owed (×10^18) | For loan cover: an ERC-20 key, above zero |
| `maxLoanGrace` | The most grace an insured loan may have | 1 second to 365 days |
| `dueFrom`, `dueTo` | The range an insured loan's due date must fall in | `dueTo` within 30 years |
| `expiry` | No fill after it | 10 minutes to 365 days ahead, and not after `dueTo` |
| `adapter` | Zero for loan cover; otherwise the one adapter whose positions the offer covers (see [liquidation cover](liquidation-cover.md)) | — |

The offer's end is `expiry` plus any time `CoverOrchestrator` has been paused since it opened, at most 30 days more (`offerEnd`). The underwriter can `cancelOffer` at any time; after the end, anyone can `closeExpired`. Either returns the unused reservation to the underwriter's free balance.

## Insuring a loan

The holder of a loan's claim units calls `insure(offerId, loanId, units, beneficiary, expected)`. `expected` repeats the offer's terms and sets `maxPremium`; any mismatch reverts (`TermsMismatch`, `PremiumAboveMax`). `quote(offerId, loanId, units)` returns the cover and premium in advance.

The loan must have the shape a [bullet loan](bullet-loan.md) creates: a one-obligation set on the same `Obligation`, secured by the Lien, in the offer's asset, with prepayment `FULL` or `NONE`, not breached and with no module of its own. Its grace must be at most `maxLoanGrace`, its due date within `[dueFrom, dueTo]` and still ahead, and its collateral a single leg of `collateralKey` of at least `minCollateralWad` per unit owed. A loan has at most one active policy at a time.

**The cover is fixed at fill:** the insured units' share of everything the loan owes at maturity, principal and interest. The cover's claim units equal its amount in base units. In one transaction:

1. the units move from the insured to `CoverOrchestrator`, which holds them for the policy's life;
2. the premium, `cover × premiumBps / 10,000` rounded up, moves from the insured's vault account to the underwriter's;
3. the cover is taken from the offer on the engine;
4. the cover is locked from the underwriter's balance as margin, and created on `Obligation`, owed by the underwriter, with a Lien on that margin. Its units are minted to the orchestrator.

The insured must have approved `CoverOrchestrator` at the vault (`approveConsumer`) and on `Obligation` (`setApprovalForAll`).

**The cover's trigger** is a `STATE` trigger on the loan's set: it comes into force if the loan breaches within the cover's window, which runs from the fill to the loan's due date plus its grace plus `COVER_TAIL` (14 days). Once in force it is due at the earlier of firing plus `payDelay` and the window's end plus `payDelay`.

## Settlement

`settle(policyId)` is open to anyone and never paused. Each call does whatever the states now allow (marking the loan breached, firing or lapsing the cover, marking the cover breached and enforcing its Lien) and records the result. `settleMany` takes a list.

| Outcome | Phase | Who receives what |
|---|---|---|
| The loan is repaid | `REPAID` | The repayment goes to the beneficiary's vault account. The cover is released, so the underwriter's margin can be returned (`Lien.release`, then `claim`) |
| The loan breaches within the window | `SUBROGATED`, then `PAID` | The loan's units, and any collateral or payoff already credited to them, go to the cover's original underwriter. The cover is then paid to the beneficiary: by the underwriter by its due date plus `coverGrace`, or else from the margin by TAKE |
| The loan breaches but the cover lapsed | `RETURNED` | The loan's units and their credit go to the beneficiary; the cover owed nothing |

Because the margin is the cover in full, in the cover's own asset, the beneficiary is paid whatever the underwriter does. Everything moves inside the vault; the orchestrator holds claim units, never a balance.

**The beneficiary** is fixed at fill and can be moved by `transferBeneficiary(policyId, to)` while the phase is `ACTIVE` or `SUBROGATED`.

**A refused push.** Units are pushed with a gas cap (200,000). If the receiver refuses, they are held (`UnitsHeld`), and that account calls `claimUnits(policyId, destination)` later.

## The bill

`BillPackager` keeps a weekly calendar: series `k` matures at `D_k = SERIES_EPOCH + k × 7 days`. On the testnet `SERIES_EPOCH` is Monday 9 November 2026 00:00 UTC, so series 1 matures on Monday 16 November 2026 and each later series a week on. A loan due at `d` belongs to the first series with `D_k ≥ d`, that is, the series whose week it is due in.

**Depositing.** A lender calls `deposit(offerId, loanId, units, recipient, expected)`. The packager pulls the units and the premium from the lender, insures the claim through `CoverOrchestrator` with itself as insured and beneficiary, and mints `cover` bills of series `k` to `recipient`. The first deposit into a series creates its `BillToken` (`openSeries`, which anyone may also call); its address is fixed in advance (`predictToken`). The lender must have approved `BillPackager` as a vault consumer and on `Obligation`.

The packager accepts only offers that fit the bill:

| Rule | Value on the testnet |
|---|---|
| Settlement asset and collateral | mUSDC and mWBTC |
| `payDelay + coverGrace` | At most 14 days (`BILL_SETTLE_BOUND`) |
| `maxLoanGrace` | At most 7 days (`BILL_GRACE_BOUND`); since a bullet loan's grace is 7 days, exactly 7 days |
| Cover per deposit | At least 100 mUSDC (`MIN_BILL_COVER`) |
| Timing | Before the series' date and before the offer's end |

**A bill** is a plain ERC-20, `GBILLk` ("Gavel Insured Bill k"), with the settlement token's decimals: one bill is a claim to one unit of mUSDC at the series' end. It has no owner, pause or transfer restriction; only its packager mints and burns.

**Finalising.** `finalise(policyId)` (anyone, never paused, no reward) settles the policy through the orchestrator and books what arrived into the series, once per policy. A policy is final when its loan is repaid, its cover paid, or, after a lapse, its loan's Lien has ended.

**Redeeming.** From `D_k`, once every policy in the series is final, a holder calls `redeem(k, amount, destination)`. The bills are burned and the holder receives the same share of the series' mUSDC and mWBTC.

**Markets.** Each listed series has its own [`SpotMarket`](spot-market.md), the bill against mUSDC. At 14 November 2026 series 1 to 10 (16 November 2026 to 18 January 2027) are listed; each market was deployed before its token exists. Each has slots of 60 to 600 seconds, a minimum order of 1 bill and 1 mUSDC, a low-volume line of 5 bills, a keeper reward of 1,000 bps (uncapped) and a discovery step reward of 0.01 mUSDC. No contract reads a bill's price.

## Pause and exits

`CoverOrchestrator`'s owner (on the testnet, the deploying account) can pause it. **A pause stops only `createOffer` and the fills** (`insure`, `insureVia` and their `…For` forms). Cancelling and closing offers, `settle`, `claimUnits` and `transferBeneficiary` keep working. A pause of `Obligation` also stops fills, since each creates a set. `BillPackager`, `BillToken` and `ContingentBondAdapter` have no owner and no pause; `finalise` and `redeem` always work, including if the vault's owner de-authorises the packager.

## Things to know

- **The premium is the underwriter's at fill**, and is not refunded if the loan is repaid early.
- **The breach must be recorded in time.** A loan breach marked after the cover's window has ended is not covered: the cover lapses and the beneficiary gets the loan's collateral in kind. `settle` marks it; anyone can. If `Obligation` is paused, a breach recorded between the window's original end and its pause-adjusted end can be fired until that later end, and lapsed after it.
- **Subrogation goes to the original underwriter**, `obligorOf(coverSetId)`, even if the cover's obligor position has been sold.
- **Self-dealing is bounded by economics, not addresses.** One address cannot insure against its own offer (the vault refuses the premium transfer); no other address check is made.
- **Too little gas reverts `settle`** (`InsufficientGas`) rather than skipping a step.
- **A bill pays its face** if each loan is repaid or its cover pays. If a cover lapsed, the series redeems partly in mWBTC.
- **Redemption waits for every policy in the series.** One default can delay it by the loan's grace (7 days), the tail (14 days) and the cover's `payDelay + coverGrace` (up to 14 days), plus the time callers take. A later pause of `Obligation` can also move a loan's due date past its series' date.
- **Rounding favours later redeemers**, by at most one base unit per asset per earlier redemption. Credit reaching a lapsed policy's units after it is final stays in the packager's vault account, outside every series.
- **The packager approves the orchestrator** for its whole vault account, which holds every series' cash.
- **An offer naming an adapter trusts that adapter.** `MockCoverAdapter`, also deployed, is a test adapter whose positions and breaches anyone can register.
- **`ContingentBondAdapter`** lets an underwriter stake BOND on paying a cover on time; a cover whose margin had to be taken reads as a shortfall, and the stake is burned. See [BOND](bond.md).

## Reading it

| View | Returns |
|---|---|
| `getOffer(offerId)`, `offerEnd(offerId)` | An offer's terms, status, remaining capacity and end |
| `getPolicy(policyId)`, `policyOfLoan(loanId)`, `deliveredOf(policyId, key)` | A policy's kind, phase, cover, ids, beneficiary, and what it has delivered |
| `dateOf(k)`, `seriesOf(due)`, `tokenOf(k)`, `predictToken(k)` | The calendar and each series' token |
| `seriesView(k)`, `recordOf(policyId)` | A series' open policies, face, supply and holdings |

Events: `OfferCreated`, `OfferClosed`, `PolicyCreated`, `PolicySettled`, `Delivered`, `UnitsHeld`, `UnitsDelivered`, `BeneficiaryTransferred`; `SeriesOpened`, `Deposited`, `Booked`, `PolicyFinalised`, `Redeemed`. The cover itself emits `Obligation`'s contingent events (`ContingentCreated`, `Fired`, `Resolved`, `Lapsed`), and each fill a `PostedOffer` take.

## Next

- [Liquidation cover](liquidation-cover.md): the same orchestrator, covering a lending position elsewhere.
- [Obligation](../concepts/obligation.md) and [the Lien](../concepts/lien.md): contingent obligations, and how the margin is taken.
- [Bullet loan](bullet-loan.md): the loans that can be insured.
