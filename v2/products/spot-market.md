# Spot market

The spot market exchanges one token for another in batches. Buyers and sellers place limit orders; the market collects them in time **slots**, and each slot clears at **one uniform price** for every order it fills. There is no continuous matching, no maker and no taker, and no fee: the only payment besides the trade itself is a reward to whoever runs the clearing, taken from the gap between a buyer's limit and the clearing price (below).

Each pair is one `SpotMarket` contract, which holds every order's escrow as a reservation in the vault and settles fills there. All pairs share one `BatchAuction` engine, which keeps the order books and walks them (see [the engines](../concepts/engines.md)). Every clearing price is recorded on chain, readable by date; the [cash-settled future](cash-settled-future.md) and [BOND](bond.md) read it. Addresses are on the [deployments](../deployments.md) page.

## Prices and amounts

A price is raw quote units per raw base unit, scaled by 10^18: `quote = amountBase × price / 10^18`, rounded down. On mWBTC/mUSDC (8 and 6 decimals), 100,000 mUSDC per mWBTC is a price of `10^21`.

## Placing an order

1. **Deposit and approve.** Deposit the asset you are giving into the vault and approve the pair's `SpotMarket` as a consumer of your account.
2. **Place.** `placeOrder(side, amountBase, limitPrice, expiresAtSlot, hintPrev)`:
   - `side`: `BUY` or `SELL`, of the base asset;
   - `amountBase`: how much base to buy or sell, at least the pair's minimum;
   - `limitPrice`: the most a buyer will pay or the least a seller will take, per unit;
   - `expiresAtSlot`: the last slot in which the order may fill, or 0 for no expiry;
   - `hintPrev`: the order to insert after in the sorted book (`findHint` returns one). If the book has moved, the call reverts and can be retried.
3. **Escrow.** A sell reserves `amountBase` of the base asset in your vault account. A buy reserves `amountBase × limitPrice / 10^18` of the quote asset: the most you could pay. A buy whose reservation is below the pair's minimum quote is refused.

An order rests in the book from the slot it was placed in until it is filled, cancelled or past its expiry slot, and may fill across several slots. When a part-filled order's remainder falls below the minimum size (or, for a buy, its quote falls below the minimum quote), it is closed and its reservation released. Each owner may hold at most 80 open orders on a pair.

An address you approve with `setOperatorApproval` may place and cancel for you (`placeOrderFor`, `cancelOrderFor`); the order and its proceeds stay yours.

## Slots and the clearing price

A slot opens as the last one closes. **Anyone may call `clear(slotId, maxSteps)`** once the current slot has been open for the pair's minimum duration. That call:

1. **closes the slot** and opens the next one at once, so new orders go to the next slot;
2. **finds the price.** It walks the buy book from the highest limit down and the sell book from the lowest up, matching while the best buy limit is at or above the best sell limit. Orders that have expired, or that were placed in a later slot, are passed over. The clearing price is the midpoint of the last matched buy limit `Lb` and sell limit `Ls`: `Ls + ⌊(Lb − Ls)/2⌋`;
3. **settles the fills** at that one price, in the vault: base to the buyer, `amount × price / 10^18` of the quote to the seller, and the buyer's unused reservation released or reduced.

Clearing runs **in pages**: each call does at most `maxSteps` steps and keeps its place, and anyone may call `clear` on the same slot to continue, first finding the price, then settling. The close must find some match within its own call, or it reverts (`NothingToClear`). Only one slot clears at a time.

A slot with nothing to clear can be ended by anyone with `expireEmptySlot(slotId)` after the pair's maximum duration; resting orders carry over to the next.

**Self-pairs are skipped.** When the best buy and the best sell have the same owner, the sell is passed over for that clearing. The check is by owner address only.

## What the clearer is paid

The call that clears earns a reward, paid in the quote asset out of each fill's **gap**: the difference between what the buyer reserved at their limit and what the seller is paid at the clearing price.

- Per fill, `KEEPER_REWARD_BPS` of the gap is taken: 1,000 basis points (10%) on every deployed market. The rest of the gap stays with the buyer.
- The pages that find the price earn `DISCOVERY_STEP_REWARD` per step as a credit. Credits are paid first, oldest first, out of the fills' shares; whatever is left of each share goes to whoever ran the settling page. Credits not paid by the end of settlement lapse.
- A buyer is never paid from their own fill: a share whose payee is the fill's buyer is not taken.
- `KEEPER_REWARD_CAP` limits the total per slot. On every deployed market it is the largest `uint256`, so in effect there is no cap.

So a buyer pays the clearing price plus a tenth of the gap to their limit, and never more than their limit. A buy at exactly the clearing price pays no reward. Sellers pay none.

## Cancelling and leaving

`cancelOrder(orderId)` is never stopped by a pause. A cancelled order leaves the book at once unless the clearing now in progress could still fill it, in which case it stays in that clearing. **Its reservation is released when the drain reaches it**, after the slot it was queued in has cleared or expired.

**The drain.** `drain(maxEntries)`, open to anyone and never paused, releases the cancelled and expired orders of finished slots, up to 90 a call. Each close drains up to 50 first, and reverts while more are waiting.

**Your release does not depend on the drain.** `cancelOrder` releases your reservation at once, from the market's own record, when:

- the clearing in progress has **stalled** (below);
- you cancelled the order at least the stall period ago and it has not been released (call `cancelOrder` again); or
- the engine cannot process the cancel and something is still reserved.

## A stalled clearing

A clearing has **stalled** when its slot has been clearing for the pair's stall period (1 day) since it closed, not counting time paused. From then, `clear` refuses that slot, and anyone may call `abandonClearing(slotId)`. Anyone may also abandon the clearing in progress while the market is not authorised at the vault.

- If no price had been found, the slot is ended with nothing filled.
- If a price had been found, the slot ends as cleared with the volume settled so far. **The recorded price stays.** Orders not yet settled keep their reservations, stay in the book, and can be cancelled.

A page that fails the market's own checks on what the engine returns is refused, and the slot abandoned in the same call (`ClearingRefused`).

## Parameters on the testnet

Every pair: slots of 60 seconds to 10 minutes, at most 80 open orders per owner, a stall period of 1 day, a reward of 1,000 basis points of the gap, and no reward cap. Minimums are in raw units:

| Pair | Market id | Minimum order (base) | Minimum buy (quote) | Low-volume line (base) | Step reward (quote) |
|---|---|---|---|---|---|
| mWBTC/mUSDC | 1 | 100000 (0.001 mWBTC) | 1000000 (1 mUSDC) | 500000 | 10000 (0.01 mUSDC) |
| mWETH/mUSDC | 4 | 3×10^16 (0.03 mWETH) | 1000000 | 1.5×10^17 | 10000 |
| mWBTC/mUSDT | 5 | 100000 | 1000000 (1 mUSDT) | 500000 | 10000 |
| mUSDT/mUSDC | 6 | 100000000 (100 mUSDT) | 1000000 | 500000000 | 10000 |
| mWETH/mWBTC | 7 | 3×10^16 | 1000 (0.00001 mWBTC) | 1.5×10^17 | 10 |
| mcbBTC/mUSDC | 8 | 100000 (0.001 mcbBTC) | 1000000 | 500000 | 10000 |

mWETH/mWBTC is quoted in mWBTC. A slot whose volume is below the low-volume line is flagged (`lowVolumeFlag`); the flag changes nothing in the market.

Each bill series also has a `SpotMarket` (bill token against mUSDC); see [insurance and bills](insurance-and-bills.md). Every parameter is fixed at deployment.

## Pause

The owner of each market (on the testnet, the deploying account) can pause it. **A pause stops `placeOrder`, `clear`, `expireEmptySlot` and `abandonClearing`.** `cancelOrder`, `drain` and the release within a cancel keep working. Time paused is added to the current slot's timing at the next place, clear or expiry (at most 30 days a slot), and is not counted toward a stall. The owner has no other power: no setter, and no release of anyone's escrow.

## Things to know

- **One owner, two addresses.** The self-pair check is by address. One person with two addresses can trade with themselves at any price and have it recorded.
- **A recorded price is not a settled one.** A price is recorded when it is found, before its fills settle, at that call's time. A clearing abandoned during settlement keeps its price in the record. The views cannot tell it from a slot settled in full; only the events can (`SettlementAbandoned`, `ClearingRefused`).
- **Two clearings can share a time.** If one block finds two slots' prices, both carry the same `clearedAt`, and `getClearingAtOrBefore` returns the later one.
- **Liveness.** A slot clears only if someone calls `clear`; otherwise nothing is recorded.
- **De-authorisation.** If the vault's owner removes a market from the whitelist, it cannot record a new price or settle fills, and the clearing in progress can be abandoned. Cancels, the drain and releases still work.

## Reading it

| View | Returns |
|---|---|
| `getLatestClearedPrice()` | `(price, slotId, clearedAt)` of the last recorded clearing; zeros if none |
| `getClearingAtOrBefore(timestamp)` | The last clearing recorded at or before a time; zeros if none. For a past time the answer never changes |
| `clearedSlotCount()`, `clearedSlotAt(index)` | The record by index |
| `getSlot(slotId)` | Timing, status, price, matched and settled volume, reward paid, low-volume flag |
| `getCurrentSlot()`, `clearingSlot()`, `getCurrentSlotClearableAt()`, `getCurrentSlotMaxExpiresAt()` | The open slot, the one clearing, and when each can be cleared or expired, pause included |
| `getOrder(orderId)`, `getOrderRecord(orderId)`, `openOrdersByUser(user)` | An order, the market's own record of it, an owner's open count |
| `getBook`, `bestOrder`, `findHint`, `getWalk`, `getDiscoveryCredits`, `drainPosition` | The book and the clearing's progress |

Events from the market: `ClearingFound`, `ClearingPageRun`, `SlotCleared`, `ClearingRefused`, `OrderReleased`. The engine emits the order and slot events (`OrderPlaced`, `OrderFilled`, `OrderCancelled`, `SettlementAbandoned` and others) and the standard auction events; filter them by `AuctionController`.

## Next

- [The engines](../concepts/engines.md): `BatchAuction` and the event stream.
- [Cash-settled future](cash-settled-future.md): a product that settles from these prices.
- [Reading data](../build/reading-data.md) and [writing](../build/writing.md).
