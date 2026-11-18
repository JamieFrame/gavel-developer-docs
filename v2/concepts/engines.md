# The engines

The engines find a price and hold nothing. Each is a deployed contract that runs one kind of auction: it records bids, applies its rule and hands back a result. The product that opened the auction holds every bidder's escrow in the vault and settles the result on the core.

## What every engine shares

- **No value.** An engine holds no token, has no vault balance, is not on the vault's whitelist and has no `payable` function.
- **No outside calls.** Every input arrives as an argument.
- **Controller-only writes.** Anyone may open an auction and becomes its **controller**, usually a product contract. Only the controller records bids, closes, finalises or expires it. Except on `BatchAuction`, the controller names a **principal** (the borrower, lender, seller or maker the auction is run for), and the engine refuses the principal as a bidder, by address.
- **A deterministic result**, which anyone can read once the auction is closed.
- **No owner, no pause, no upgrade.**
- **Bounded work.** Every loop is bounded by a constant or by the caller's argument. Engines whose book has no size limit close **in pages**: each call does at most the number of steps its caller asks for and keeps its place, so a page's gas is bounded however large the book.

Because an engine holds nothing, **your exits never depend on it.** If a product stops calling an engine, or an engine call fails, each bidder's escrow is still released from the product's own records.

## The five engines

| Engine | What it runs | Used on the testnet by |
|---|---|---|
| `SingleAuction` | An open auction with one winner. Bids improve upward (ascending) or downward (descending) by at least a step; the best bid at the end wins | The bullet loan (a request to one lender, an offer to one borrower); the secondary market |
| `GroupAuction` | A ranked book with many winners, all at one clearing price. Bids may be partial or all-or-nothing. At most 50 winners; the close runs in pages | The bullet loan (a request to many lenders, an offer to many borrowers) |
| `PostedOffer` | A maker posts terms at a fixed price; takers take some or all of it until it ends | The cash-settled future; insurance |
| `BatchAuction` | A two-sided market cleared in time slots: every buy and sell order in a slot clears at one uniform price. Each slot closes in pages | The spot market (one market per pair) |
| `ClockAuction` | A price that moves with time (falling for a seller, rising for a buyer) until someone accepts, in parts | Deployed; no product on the testnet uses it yet |

**`SealedFront`** is a helper, not an engine: it finds no price. It takes **sealed bids** in front of `GroupAuction` or `SingleAuction`. In a first window bidders commit a hash of their bid; in a second they reveal it, and the product records each revealed bid into the engine, whose own rules then clear the auction. It proves only that each bid was fixed before any was seen. It is deployed; no product on the testnet uses it yet.

**`BatchAuction` differs from the other four.** A market on it has no principal and no gate: both sides place orders through the product. Its slots last from 60 seconds to 30 days, as the market sets, and only its controller (the product) can clear a slot or expire one; anyone may ask the product to do so. The rest of this page, up to *Prices and quantities*, is about the other four engines.

## Who may bid

Each auction on `SingleAuction`, `GroupAuction`, `PostedOffer` or `ClockAuction` has a **gate**, fixed when it opens:

| Gate | Who may bid |
|---|---|
| `NONE` | Anyone |
| `CONTROLLER` | Whoever the product admits; the product vets bidders itself |
| `MERKLE` | Holders of a proof against a Merkle root of permitted addresses |

The product chooses the gate. A named counterparty is a one-address Merkle tree: the cash-settled future uses one when its maker names a taker. A product that must admit only verified participants can use `CONTROLLER` or `MERKLE` for every auction it opens.

## Time and pauses

Each of these engines bounds every auction's life when it opens:

| Bound | Value |
|---|---|
| Start | Now, or up to 30 days ahead |
| Shortest auction | 10 minutes |
| Longest auction | 90 days (`PostedOffer`: 365 days) |
| Anti-sniping extension (`SingleAuction`, `GroupAuction`) | At most 1 hour, never past the auction's latest end |
| Total pause pushed into one auction | At most 30 days (on `BatchAuction`, per slot) |

Products set narrower bounds of their own; each product's page gives them.

**Engines have no pause and read only the block's time.** When a product is paused, it pushes the paused time into each of its auctions at its next call (a bid, a close), so bidding never ends during a pause. The product's own views add any pause not yet pushed, and are the ones to read. Beyond 30 days in total, an engine stops moving the end, and bidding may end during a long pause; the product's exits are unaffected.

Closing an open auction is the controller's call, so that no one can fix a result at a time the pause should have moved. Products close permissionlessly through their own finalisation, which anyone may call. As a backstop on these four engines, anyone may close an auction directly once no further pause could reopen it.

## Prices and quantities

Prices are `uint256` at 10^18 = one unit of the quote per unit of quantity; quantities are `uint128`. The product converts amounts to and from this scale, rounding against the bidder at entry and in the principal's favour at settlement, at most one 10^-18 of a unit.

## One event stream

All five engines emit the same events, so one indexer reads every price formed on Gavel:

| Event | When |
|---|---|
| `AuctionOpened` | An auction opens. It names how the price is formed (`kernel`: `OPEN_BID`, `CLOCK`, `RANKED`, `POSTED` or `BATCH`), which way the price improves for the opener (`better`: `LOWER`, `HIGHER`, or `NONE` for a batch), what is priced and in what, and the term for credit |
| `AuctionController` | With every opening: the auction's controller and principal |
| `BidPlaced` | A bid is recorded |
| `AuctionCleared` | The price is found |
| `AuctionSettled` | The controller settled the result, naming where (`settledIn`, `settledId`): for a loan, the set it created. Once per take for a posted offer or a clock |
| `AuctionExpired`, `AuctionCancelled` | It ended with no settlement |

Because anyone can open an auction on an engine, **filter events by controller**: an auction counts as a product's only if its `AuctionController` names that product. Products' own events say which of their auctions an engine id belongs to.

## Things a bidder should know

- **Self-bidding.** An engine refuses the principal as a bidder by address only. A principal can bid from a second address.
- **Posted offers are free options.** A maker's posted price can be taken at any time until the offer ends, whatever has happened since; the maker can cancel.
- **First to accept a clock wins**, in the order the sequencer includes transactions.
- **In a group, a skipped bid** (all-or-nothing, or with a large minimum fill, that does not fit) can let a lower-ranked bid win, so the clearing price can be worse for the principal than the skipped bid.
- **A stale hint.** Bids into a sorted book carry a hint to their place in it; if the book moved, the call reverts `StaleHint` and can be retried.

## Next

- [The base layer and what you trust](base-layer-and-trust.md): what the engines, the core and the products promise, and what their owners can do.
- The products that use the engines: [bullet loan](../products/bullet-loan.md), [spot market](../products/spot-market.md), [secondary market](../products/secondary-market.md).
