# Cash-settled future

A cash-settled future is an agreement between a **long** and a **short** to settle, in the quote asset, the difference between a **forward price** fixed today and a **reference price** read later from a spot market's recorded clearings. Nothing is delivered. Each side's loss is capped, and each side posts its cap as margin, locked in the vault under the Lien.

The instrument is two obligations on the core, one owed by each side. Two contracts make it: `FutureOrchestrator`, which takes the maker's posted offer on the `PostedOffer` engine and creates both legs when it is taken, and `SpotReferenceResolver`, which fixes the reference price from the pair's `SpotMarket`. On the testnet there is one of each, on the mWBTC/mUSDC pair, settling in mUSDC. Addresses are on the [deployments](../deployments.md) page.

## The terms

| Term | Meaning |
|---|---|
| `makerSide` | `LONG` or `SHORT`, for the maker; the taker takes the other side |
| `quantity` (`Q`) | Notional, in raw base units |
| `forwardPrice` (`F`) | Quote per base, on the spot market's scale (raw units × 10^18) |
| `capPerUnit` (`P`) | The most either side can lose per unit, same scale |
| `refTime` (`T`) | The reference time, a Unix time |
| `samples` (`k`) | How many clearings the reference uses: odd, 1 to 9 |
| `lookback` (`L`) | How far before `T` a clearing may be: 10 minutes to 7 days |
| `expiresAt` | When the offer stops being takeable |
| `taker` | A named taker, or zero for anyone |

The margin, per side, is `cap = ⌊Q × P / 10^18⌋` of the quote asset. With `S` the reference price:

| Leg | Owed by | Paid to | Amount |
|---|---|---|---|
| The long's leg | The long | The short | `min(⌊Q × max(F − S, 0) / 10^18⌋, cap)` |
| The short's leg | The short | The long | `min(⌊Q × max(S − F, 0) / 10^18⌋, cap)` |

At most one leg is above zero. Seen as options, the short's leg pays the long a capped call spread and the long's leg pays the short the matching put spread.

## Offering and taking

**The maker** deposits the quote asset into the vault, approves `FutureOrchestrator` as a consumer, and calls `offer(terms)`. The product reserves `cap` in the maker's account and opens a posted offer on `PostedOffer` at price `F` for the whole quantity, never repriced. On the testnet:

- `T` must be between 1 hour and 365 days ahead;
- `expiresAt` must be at least 10 minutes ahead and no later than `T − max(1 hour, L)`, so the offer cannot be taken once a sample could already be recorded.

**The taker** approves `FutureOrchestrator` and calls `take(offerId)`, taking the whole offer. In one transaction the product:

1. releases the maker's reservation and locks `cap` from the maker and `cap` from the taker, each as a lock in its own name (the taker's is drawn from their free vault balance first, and any shortfall from their wallet);
2. creates the two legs on Obligation, each a contingent set of one obligation;
3. hands each lock to the Lien, securing the leg whose obligor posted it;
4. records the take on the engine, naming the future.

Each side ends up holding its own leg's **obligor position** (its margin, less what its leg owes) and the **claim units** of the other leg (10^18 units per leg). The four tokens are separate and transferable, so the future can be split.

`cancelOffer(offerId)` (the maker) and `expireOffer(offerId)` (anyone, once the offer has ended) release the maker's reservation. Neither is ever paused. `offerFor`, `takeFor` and `cancelOfferFor` let an address the user approved with `setOperatorApproval` act for them.

## How the reference price is fixed

**`S` is the median of the last `k` clearings the spot market recorded at or before `T`, each recorded no earlier than `T − L`.** With fewer than `k` such clearings, the reference is **void**. Because `k` is odd, the median is one recorded price, never an average.

After `T`, anyone calls `fix(T, k, L)` on the resolver, which reads the market's `getClearingAtOrBefore` backwards from `T` and stores the result once. The answer for a past `T` never changes, so whoever fixes it, and whenever, the result is the same. Futures sharing one `(T, k, L)` share one reference.

## Firing, resolution and settlement

Each leg is `RESOLVER`-triggered: it comes into force (**fires**) once its reference is fixed or void, within a window from `T + 1` to `T + 7 days`, moved later by any pause of Obligation. Then it **resolves** to its amount.

**`settle(futureId)`**, open to anyone and never paused, does every step that applies in one call: `fix` if needed, then `fire` and `resolve` on each leg. Each step can also be called directly on the resolver and Obligation.

| Outcome | What happens |
|---|---|
| **The winner's leg** | Resolves to zero and is discharged. Anyone calls the Lien's `release`; the winner `claim`s their whole margin back |
| **The loser's leg** | Resolves to `A ≤ cap`, due 1 day after firing. The loser may pay `A`, after which the margin is released |
| **The loser does not pay** | After the due date and a 1-hour grace, anyone may `markBreached` and `enforce`. The margin is in the leg's own currency, so the Lien takes exactly `A` and credits the rest to the loser's position. The winner `collect`s on their claim units either way |
| **Void reference** | Both legs resolve to zero; both margins are released |
| **A leg not fired within the window** | Anyone may `lapse` it, and it is discharged with nothing owed. **If it is the loser's leg, the loser owes nothing** |

If neither leg is fired within 7 days of `T`, the loser's leg lapses and nothing is owed; firing is open to anyone. A leg fired directly, without `resolve` in the same transaction, starts its 1-day payment period at firing.

The parties lock `2 × cap` together; the legs are not netted, there is no margin call and no early release of margin. There is no mutual cancellation; each holder can only release their own claim units.

## Pause

The owner of `FutureOrchestrator` (on the testnet, the deploying account) can pause it. **A pause stops `offer` and `take` only.** Cancelling and expiring offers, `settle`, firing, payment and release keep working. An open offer's end moves later by the time paused, never past its latest end (`T − max(1 hour, L)`) and by at most 30 days. `SpotReferenceResolver` has no owner and no pause.

## Things to know

- **A posted offer is a free option.** Anyone permitted may take it at `F` at any moment until it ends or the maker cancels, however the market has moved since. The maker controls the exposure with `expiresAt` and `cancelOffer`.
- **Naming a taker.** If the maker sets `taker`, the offer is gated by a one-address Merkle tree, which the engine checks; only that address, or an address it approved, can take. The engine also refuses the maker as taker, by address only.
- **An offer does not bind the maker.** If the maker revokes its approval of the product, or the vault's owner de-authorises the core, the Lien or the product, every take reverts; takers lose only gas.
- **A spot market pause or de-authorisation near `T`.** `T` is real time. While the spot market is paused, or not authorised at the vault, it records no new clearings. Clearings before it still count if within `L`; otherwise the reference is void. **So pausing the spot market, or de-authorising it, shortly before `T` can change which clearings count, or force void.** It cannot change the terms or the formula.
- **Self-crossing.** The spot market skips a trade between two orders of one owner by address only. One person with two addresses can record any price. Against a future the most such a party can gain is `cap`, the counterparty's margin, per future; across many futures sharing one `(T, k, L)` the gains add. To move the median they must own `(k + 1) / 2` of the `k` samples. Where independent orders rest in those slots, they fill at the pushed price at the pusher's expense. On a pair with little independent trading, the reference is whatever is recorded.
- **Prints the resolver cannot tell apart.** A clearing abandoned during settlement keeps its recorded price and counts as a sample. Two clearings recorded at the same time count once.
- **Void as an outcome.** A pair that clears rarely may record fewer than `k` clearings in `[T − L, T]`, and the future is then void.
- **Filter engine events by controller.** Anyone can open a `PostedOffer` naming this product as its item; only offers whose `AuctionController` is `FutureOrchestrator` are futures.

## Reading it

| View | Returns |
|---|---|
| `getOffer(offerId)` | The maker, status, terms, margin, engine id |
| `offerEnd(offerId)` | `(end, live)`, including any pause not yet applied |
| `getFuture(futureId)` | The long, the short, both legs' set, obligation and lien ids |
| `futureOfSet(setId)` | The future a set belongs to |
| `bounds()` | Minimum and maximum tenor, fire window, payment interval, grace |
| `settlementAsset()`, `vault()`, `obligation()`, `lien()`, `resolver()`, `engine()` | The wiring |
| `referenceOf(T, k, L)`, `referenceKey(T, k, L)` (resolver) | The reference's status (`NONE`, `FIXED`, `VOID`), price and time fixed |
| `spotAuction()` (resolver) | The `SpotMarket` it reads |

Events: `FutureOffered`, `OfferCancelled`, `OfferExpired`, `FutureOpened`, `Settled`; the resolver's `ReferenceFixed`, with the slots sampled; Obligation's `Fired`, `Resolved`, `Lapsed`; and the Lien's events. Before resolution, a leg's plain amounts read at `cap`; `contingentStateOf` gives its state.

## Next

- [Spot market](spot-market.md): where the reference prices come from.
- [Obligation](../concepts/obligation.md): contingent obligations; [the Lien](../concepts/lien.md): margin and TAKE.
- [Composing a product](../build/composing-a-product.md).
