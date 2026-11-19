# The secondary market

The secondary market sells either side of a debt before it is due. A creditor sells a **lot** of claim units; a debtor sells an **obligor position**, which carries the debt, the right to pay it and the equity in its collateral. Each sale is an open ascending auction, paid in an ERC-20 of the seller's choice.

Two contracts run it: `ClaimLotMarket` for claim units, with the linked library `LotCreditIndex`, and `ObligorPositionMarket` for positions. Both run on `SingleAuction` and are bound to the testnet's `Obligation`, `ObligorPosition` and vault. Only claim lots and obligor positions are sold. Addresses are on the [deployments](../deployments.md) page.

Unlike most products, the market holds what it is selling: the units or the position sit in the market's own address for the life of the sale. Bids are escrowed in the vault or in the market itself, as below.

## Selling

### A lot of claim units

`ClaimLotMarket.createAuction(id, units, paymentToken, reservePrice, duration, expectedAsset, expectedDueAt)` lists `units` of claim id `id`. The seller first approves the market on Obligation (`setApprovalForAll`). The listing is refused unless:

- the obligation is `OPEN`, and its asset and stored due date match `expectedAsset` and `expectedDueAt` (`ExpectedMismatch`), which guards the seller against listing the wrong claim;
- its set's vault is the market's own vault (`VaultNotAccepted`);
- the obligation's unit supply is at most 10^18 (`SupplyTooFine`);
- the sale can end at least one day before the obligation's due date (below).

Credit the units earned before the listing stays with the seller. Credit they earn while the market holds them (a repayment, say) goes to whoever receives the lot at the end. For a set with an ERC-721 in its collateral, claim units move only as a whole holding, so the lot must be the seller's whole balance.

### An obligor position

`ObligorPositionMarket.createAuction(setId, paymentToken, reservePrice, duration, expectedMaxDebt)` lists the position of set `setId`. The seller first approves the market on the position token. The listing is refused unless:

- the position is `ACTIVE`, or `IN_ENFORCEMENT` (a secured set in breach, not yet enforced);
- its set's vault is the market's own vault (`VaultNotAccepted`);
- the set's debt now is at most `expectedMaxDebt` (`DebtAboveMax`).

The buyer takes the debt and the collateral together: whoever holds the position can pay the set, pay it off through the Lien if it is in breach, and claim whatever the Lien credits to it. See [claims and obligor positions](../concepts/claims-and-positions.md).

### Both

The seller names the payment token, a reserve (zero means 1 base unit) and a duration of 1 hour to 7 days. The engine opens with the seller as principal, so the seller cannot bid from its own address. `createAuctionFor(user, ...)` lets an operator the seller approved with `setOperatorApproval` list for it.

## The latest end

Every sale has a **latest end**, fixed when it opens, past which no bid is taken:

| Item | Latest end |
|---|---|
| Claim units | The obligation's due date (pause-adjusted at opening) less one day |
| Position, set `ACTIVE` | The due date (pause-adjusted at opening) of the set's earliest open obligation |
| Position, set `IN_ENFORCEMENT` | The opening plus 7 days, moved later by any pause of the market |

The engine also caps every sale at 90 days from its opening. `now + duration` must fall within the latest end (`EndBeyondLatest`).

Because bidding on an `ACTIVE` position stops by its earliest due date, and breach needs that date plus the grace, a position listed `ACTIVE` cannot breach while bids run.

## Bidding

`bid(auctionId, amount, expectedItem, expectedPaymentToken, maxDebt)`, or `bidFor(user, ...)` by an approved operator. `amount` is the whole price for the item, in base units of the payment token.

- `expectedItem` and `expectedPaymentToken` must match the sale. For a position, the set's debt now must be at most `maxDebt`; a lot ignores it.
- A first bid must reach the reserve; each later bid must beat the standing bid by at least 1% (`minimumBid(auctionId)` gives the least next bid). The standing bidder may raise its own bid.
- A bid on an item whose underlying has finished (below) reverts `UnderlyingFinished`; a bid at or after the latest end reverts `AuctionEnded`.

**Escrow takes one of two routes**, chosen per bid:

| Route | When | Escrow | A displaced bid |
|---|---|---|---|
| **Vault** | The market is admitted at the vault, the bidder approved it as a consumer, and the bidder's free balance covers the amount | Moved into the market's own vault account | Moved back to the bidder's free balance in the same transaction |
| **Wallet** | Otherwise | Pulled from the wallet (the bidder approves the market on the token) | Credited to `pendingRefunds`; the bidder calls `claimRefund(token, destination)` |

Neither route calls the displaced bidder's code.

**Anti-sniping.** A bid accepted in the last 10 minutes moves the end to 10 minutes after it, never past the latest end. Each extension needs a bid at least 1% higher, so there are finitely many.

## Settlement

Once the effective end has passed, **anyone** may call `settle(auctionId)`. The outcome is fixed at the end: the standing bidder wins if there is one, and nothing between the end and `settle` changes who gets what.

| Outcome | The item, and what it carries | The payment |
|---|---|---|
| **SOLD** | To the winner | To the seller: into its free vault balance if the winning bid came through the vault, otherwise credited to `pendingRefunds` for `claimRefund` |
| **UNSOLD** (no bid) | Back to the seller | — |
| **CANCELLED** | Back to the seller | — |
| **SETTLED_UNDERLYING** | Back to the seller | The best bid refunded |

For a lot, "what it carries" is the credit the units earned while the market held them; it is collected and paid into the recipient's vault balance at `settle`, before the units are delivered. ERC-721 credit travels with the units. A position carries whatever the Lien holds for it.

**Units a recipient refuses.** If the recipient's ERC-1155 hook reverts, the units are held for it (`pendingUnits`) and keep earning; it calls `claimUnits(id, destination)` to take them with their credit.

**An engine that cannot close.** If the engine can neither close the auction nor report a result, `settle` reverts until 3 days after the end (`STALL_PERIOD`); after that it unwinds the sale from the market's own records, refunds the standing bid and returns the item (`SaleRefused`). The same unwind follows any result that does not match the market's record.

## Cancellation

The seller, or its operator, may `cancelAuction(auctionId)` before the end, and only while there is no bid (`CannotCancelWithBids`). The item comes back with what it carries.

## If the set is paid or enforced during a sale

The underlying of a sale **finishes** when:

- for a lot, its obligation is `DISCHARGED`, `SETTLED` (enforced), `PAID_LATE` or `RELEASED`;
- for a position, its state is `CREDIT_PENDING` (the set paid, paid off or enforced with something credited back) or `FINISHED` (the position burned).

Anyone may pay a set at any time, the market's holding notwithstanding. **If anyone calls `settle` strictly before the end** on a finished underlying, the sale ends SETTLED_UNDERLYING: the engine auction is cancelled, the best bid refunded, and the item returned to the seller with what it carries, the repayment credit included. **If no one does,** the finish counts as after the end and the sale ends SOLD or UNSOLD as usual; a winning bidder then receives the item with whatever it carries. A seller who wants the item back, or a bidder who wants the refund, calls `settle` before the end.

- **A seller's repayment.** The seller of an `ACTIVE` position is its obligor and may repay the whole debt during the sale, then `settle` before the end. This works as a cancellation with bids, at the cost of the debt.
- **A position in enforcement.** Anyone may pay off or enforce a set `IN_ENFORCEMENT` while its position is listed. If enforcement leaves nothing for the position, the position burns: `settle` delivers nothing (`AuctionSettledWithoutDelivery`) and, if the sale had ended SOLD, still pays the seller. A bidder on a position in enforcement carries this until `settle`.

## Sets with modules

A set may name modules: an amount resolved later, a trigger, an exercise right. Such a claim or position is listed only once its terms are final: its obligation `RESOLVED`, and, where it carries an exercise right, its exercise window (pause-adjusted) closed. Otherwise the listing reverts `InvalidItemState`. A plain set passes this check.

## Pause

Each market's owner can `pause()` it; on the testnet that is the deploying account, and ownership cannot be renounced. A pause stops `createAuction` and `bid`. `settle`, `cancelAuction`, `claimRefund` and `claimUnits` keep working.

The market pushes its paused time into each sale's engine auction at the sale's next call, so bidding does not end during a pause. For a sale with a fixed latest end (lots, `ACTIVE` positions), the push stops at that latest end; a sale on the market's own clock moves its latest end with the pause. A pause of Obligation does not move a fixed latest end.

If the vault's owner withdraws the market's admission, new bids take the wallet route; every payout from the market's own vault account still works.

## Things to know

- **The market's own vault only.** A set may name any vault; each market lists only sets whose vault is the one it was deployed with.
- **Units sent to the market directly.** Units minted to `ClaimLotMarket` when a set is created are not a sale and stay there. While such a set names a lien that is not yet bound, the market refuses any listing of an id of which it holds a unit (`EntryExists`).
- **No fee.** The market charges nothing; the seller receives the whole winning bid.

## Reading it

| View | Returns |
|---|---|
| `getAuction(auctionId)` | Seller, payment token, item, size, reserve, start, best bidder and bid, its route, outcome (`NONE` while open) |
| `effectiveEnd(auctionId)`, `latestEnd(auctionId)` | The end and the latest end, with the market's unpushed paused time |
| `currentPrice(auctionId)`, `minimumBid(auctionId)` | The standing bid; the least next bid |
| `latestClearing(itemId)` | The last SOLD price, size, payment token and time for an item |
| `auctionIdByItem(setId)` | A position's open sale (`ObligorPositionMarket`) |
| `lotAccrued(auctionId, key)`, `pendingUnits(user, id)` | A lot's credit so far; units held for a recipient (`ClaimLotMarket`) |
| `pendingRefunds(user, token)` | Wallet-route refunds and proceeds to claim |

Events: `AuctionCreated`, `BidEscrowed`, `AuctionSold`, `AuctionUnsold`, `AuctionSettledUnderlying`, `AuctionSettledWithoutDelivery`, `SaleRefused`, `RefundAvailable`, `RefundClaimed`, `PauseStarted`, `PauseEnded`; on `ClaimLotMarket` also `LotCreditPaid`, `UnitsHeld`, `UnitsClaimed`. The engine emits the shared auction events, keyed by the same auction id, with the market as controller.

## Next

- [Claims and obligor positions](../concepts/claims-and-positions.md): what each token carries.
- [The bullet loan](bullet-loan.md): one source of the claims and positions sold here.
- [The engines](../concepts/engines.md): `SingleAuction`'s rules.
