# Liquidation cover

Liquidation cover pays a borrower on Morpho Blue a fixed sum if their position is liquidated. The sum is at most the liquidation penalty on their debt. The position is held in a **cover account** that no one can manage at the lending market while the cover can still pay, so only the market's price (or its interest rate) can bring it to liquidation. The cover itself is an [insurance](insurance-and-bills.md) policy written through `CoverOrchestrator` against an underwriter's offer, fully margined under the Lien.

The contracts are `MorphoCoverAdapter`, three instances on one market, and `CoverAccount`, the implementation each account is cloned from. On the testnet they run against `MockMorpho`, `MockOracle` and `MockIrm`, a test copy of Morpho Blue. Addresses are on the [deployments](../deployments.md) page.

## The parts

| Contract | What it does |
|---|---|
| `MorphoCoverAdapter` | One per lending market, `CoverOrchestrator` and ceiling. It opens cover accounts, keeps their state, and is the module `CoverOrchestrator` and `Obligation` consult for a cover: it answers whether a position may be covered and for how much (`coverTerms`), whether it has been liquidated (`breachOf`), and binds each cover to its position (`onCreate`) |
| `CoverAccount` | One borrower's position at the lending market, as a minimal clone (EIP-1167) at its own address. It does only what its adapter commands: one call to the market on its own behalf, or a transfer of a stray token. It never grants an authorisation at the market and cannot sign, so no one can withdraw its collateral except through the adapter |

The three adapters differ only in their **ceiling**, `MAX_HEALTH_USE`: 0.6, 0.7 and 0.8 of the position's borrowing limit. An underwriter chooses which to cover by naming one adapter in its offer.

## How it works: the borrower

1. **Open and fund.** `openAccount()` clones a `CoverAccount`; the caller is its owner for good. `supplyCollateral(accountId, assets)` and `borrow(accountId, assets, receiver)` act on the position (approve the adapter for the collateral token first). The market's own health and liquidity checks apply.
2. **Arm.** `arm(accountId)` returns a `ref` and **freezes** the account. It is refused unless the position is within the adapter's ceiling. The adapter records the position's collateral as `tracked`.
3. **Fill.** On `CoverOrchestrator`, `insureVia(offerId, ref, cover, windowEnd, beneficiary, expected)` against an offer naming this adapter. The borrower pays the premium from their vault balance (approve `CoverOrchestrator` at the vault). `cover` may be at most `coverTerms`' `maxCover`; `windowEnd` must be in the future and no later than the offer's `dueTo`. The cover's units go to the beneficiary, which must be the account's owner: the adapter refuses to bind a cover unless the owner holds every unit.
4. **Wait.** While the cover can still pay, the account stays frozen.

An armed account with no cover can be ended with `disarm(ref)`.

**The freeze.** An account is frozen while its current `ref` is armed, or bound to a cover that is still pending and not released. While frozen, `borrow`, `arm` and `withdrawCollateral` are refused, except that collateral can be withdrawn once the debt is zero. `supplyCollateral` (owner), `repay` and `repayAll` (anyone) always work.

## What is covered

The cover pays on a **recorded liquidation**: a fall in the position's collateral below `tracked`. Because the account can lower its collateral only through the adapter, and the adapter refuses withdrawals while frozen, such a fall can only be a liquidation. `record(ref)` (anyone) reads the position and records the first fall (`liquidatedAt`). Every adapter call on an account records first. Collateral supplied on the account's behalf by someone else raises `tracked`, never lowers it.

**The amount.** `maxCover` is the position's debt, accrued to now, times `LIF − 1`, where `LIF` is the market's liquidation incentive factor, `min(1.15, 1 / (1 − 0.3 × (1 − LLTV)))`. On the testnet's market (LLTV 86%) that is about 4.38% of the debt. The amount is fixed at fill; it does not depend on how much is liquidated.

## The oracle read

`MorphoCoverAdapter` reads the lending market's own oracle, the price that market liquidates by, and its interest rate model. It uses them for one thing: the **ceiling**. At `arm`, and again in `coverTerms` during the fill, it computes the position's use of its borrowing limit, `debt / (collateral × price × LLTV)`, with the debt accrued to now, and refuses unless it is at most `MAX_HEALTH_USE`. This stops a borrower arming while safe, waiting while the price falls, and then buying cover on a position already close to liquidation.

The read never sets an amount. A failing oracle or rate model makes `coverTerms` refuse the fill (and `arm` refuse), never return a wrong cap. The cover's `ContingentCreated` event names the lending market as the contract it reads; the adapter's `ORACLE` and `IRM` views name the oracle and the rate model.

## On a trigger, and on expiry

| Step | Call | Who |
|---|---|---|
| A liquidation is recorded | `record(ref)` on the adapter | Anyone |
| The cover comes into force | `fire(coverId)` on `Obligation`, once a liquidation recorded within the window exists | Anyone |
| The amount is fixed | `resolve(coverId)`: the cover resolves at its full amount | Anyone |
| Payment | The underwriter pays by the due date: the earlier of firing plus `payDelay` and the window's end plus `payDelay` | The underwriter |
| If unpaid | After the due date plus `coverGrace`: `markBreached`, then `enforce` on the Lien, which takes the margin | Anyone |
| Collecting | `redeem` (or `collect`) of the cover's units on `Obligation` | The units' holder |

Once the cover fires the account unfreezes; the amount is already fixed. `CoverOrchestrator.settle` does not drive these policies: each step is a direct call.

**On expiry.** If no liquidation is recorded by the window's end, anyone can `lapse(coverId)`. The account unfreezes, and the underwriter's margin is returned through the Lien (`release`, then `claim`). **Ending early:** the holder of every unit can `release` them on `Obligation`, which ends the cover and unfreezes the account.

## Moving an existing position in

A borrower with a position held directly at the market can move it into a new cover account in one atomic batch of their own calls, with no helper contract and no flash loan. The batch opens the account and then repeats rounds of: withdraw spare collateral from the old position at the market, `supplyCollateral` it to the account, `borrow` against it from the account, and repay that much of the old debt by shares at the market. A last step moves the remaining collateral. Repaying by shares closes the old debt exactly. Moving between adapters (to change ceiling) works the same way with the old account's `withdrawCollateral`, `repay` and, last, `repayAll`, and only from an account that is not frozen.

The batch must run atomically (a wallet that supports atomic batches, such as a smart account); a plain address cannot run it in one transaction. If anything moves between signing and inclusion (the price, available liquidity, another account opened first), the whole batch reverts and only gas is lost. The new account is not armed by the move.

## The testnet's venue

Morpho Blue is not deployed on Arbitrum Sepolia, so the testnet uses `MockMorpho`, `MockOracle` and `MockIrm`, which copy its arithmetic (shares, health check, liquidation incentive, bad debt). Its one market lends mUSDC against mWBTC at an LLTV of 86%. **The mock oracle's price and the mock rate are set by a testnet key**, the deploying account, which also created the market. At deployment the price was $60,000 per mWBTC and the rate about 5% a year; at 14 November 2026 both can be changed by that key at any time, which can make any position liquidatable, and the key can make the oracle revert, which refuses new cover. Liquidation on the mock is open to anyone, as on Morpho.

## Owners, pause and the vault

`MorphoCoverAdapter` and `CoverAccount` have **no owner, no pause and no setter**. Every parameter is fixed at deployment. They **never call the vault** and are not on its whitelist: the premium, margin and payout move through `CoverOrchestrator` and `Obligation`. The collateral stays at the lending market, outside the vault.

A pause of `CoverOrchestrator` stops new offers and fills only. A pause of `Obligation` stretches every cover's window, so an account stays frozen for as long as the pause lasts.

**Morpho's own owner powers are outside Gavel.** On Morpho Blue the owner can enable rate models and LLTVs and set a fee taken from suppliers' interest; none of these touches a borrower's position, and a market's parameters are fixed at its creation. The mock has no owner; its testnet key's powers are above.

## Things to know

- **Recording is the claimant's burden.** A liquidation recorded after the window's end is not covered. `record` is open to anyone and earns nothing.
- **A touch pays in full.** At any touch of the liquidation level, the borrower can liquidate a dust amount of their own position from a second address and take the whole sum. The cover is priced as a payout on a touch.
- **A masking top-up.** After a liquidation, anyone can supply collateral on the account's behalf before it is recorded, so `record` sees no fall. It costs at least the collateral seized.
- **The oracle and the rate move positions.** A frozen position can reach liquidation by the market's oracle alone, or by interest if someone pushes the market's utilisation up.
- **`breachOf` is public.** Anyone can create a contingent obligation reading it, but only covers created by `CoverOrchestrator` bind and freeze an account.
- **One cover per `ref`, one live `ref` per account**, and an account cannot be transferred. A `ref` with a recorded liquidation cannot be filled; its owner `disarm`s it and arms again.

## Reading it

| View | Returns |
|---|---|
| `accountOf(accountId)` | The clone's address, its owner, its current `ref` |
| `refOf(ref)` | Its account, status (`ARMED`, `BOUND`, `ENDED`), cover id, `tracked`, `liquidatedAt` |
| `frozen(accountId)` | Whether the account is frozen |
| `healthOf(accountId)` | The debt accrued to now, and its use of the borrowing limit |
| `coverTerms(ref, insured)` | Whether it can be covered now, the asset, `maxCover` |
| `marketParams()`, `MAX_HEALTH_USE`, `LIF`, `ORACLE`, `IRM` | The market and the instance's ceiling |

Events: `AccountOpened`, `Armed`, `Disarmed`, `Bound`, `LiquidationRecorded`, `CollateralSynced`, `Swept`, `Supplied`, `Borrowed`, `Repaid`, `Withdrawn`.

## Next

- [Insurance and bills](insurance-and-bills.md): the orchestrator, offers and margin.
- [The base layer and what you trust](../concepts/base-layer-and-trust.md): why this product's oracle read is disclosed.
- [Obligation](../concepts/obligation.md): `STATE` triggers, firing and lapsing.
