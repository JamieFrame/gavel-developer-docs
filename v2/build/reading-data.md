# Reading data

Everything Gavel V2 records is on chain: every debt, lien, lock, bid and clearing price is readable from the contracts' views and events on Arbitrum Sepolia (chain 421614). There are two ways to read it:

1. **From the chain directly**, with views and logs. This needs nothing but an RPC endpoint and is the authoritative source.
2. **From the testnet API** run by Aletheia Analytics SASU, which indexes the same events and serves them as JSON. It is a convenience, not part of the protocol: everything it serves can be read from the chain.

Addresses are in the [address list](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.md) (a machine-readable copy is [`v2-testnet-addresses.json`](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.json)) and on the [deployments](../deployments.md) page. Full signatures of every view and event are in the [contracts reference](contracts-reference.md).

## Reading from the chain

### The views that matter

Each block's page lists its views; these are the ones a reader starts from.

| Block | Views | Page |
|---|---|---|
| The vault | `getLoanRecord`, `getLoanLegs`, `getDepositor`, `getLoanOwner` for a lock; `ledgerTotal(asset)` against the token's `balanceOf(vault)`; `isAuthorisedConsumer`, `isUserApproved` | [The vault](../concepts/vault.md) |
| Obligation | `getSet`, `getObligation`; `creatorOf`, `obligorOf`, `lienOf`, `vaultOf`; `ownerOfPosition`, `positionState`; `outstanding`, `payoff`, `debtAt`; `contingentStateOf`, `isContingentSet`; `accrued`, `unitsOutstanding` | [Obligation](../concepts/obligation.md), [claims and obligor positions](../concepts/claims-and-positions.md) |
| The Lien | `getLien`, `lienOfSet`, `lienOfLock`, `creditOf`, `creditKeys`, `obligation()` | [The Lien](../concepts/lien.md) |
| The engines | The result once an auction is closed; each product's own views for live deadlines | [The engines](../concepts/engines.md) |
| Products | `loanOf`, `deadlineOf`, `originationOf` (bullet loan); `getLatestClearedPrice`, `getClearingAtOrBefore`, `getSlot` (spot market); `getAuction`, `latestClearing` (secondary market); `getOffer`, `getFuture`, `getPolicy` and the others on each product's page | [Bullet loan](../products/bullet-loan.md), [spot market](../products/spot-market.md), [secondary market](../products/secondary-market.md) |

**A product's views give the live deadline; the engine's stored end can be earlier.** A product pushes paused time into its auctions only at its next call. The product's views (`deadlineOf` on the bullet loan, `effectiveEnd` on the secondary market, `offerEnd` on posted offers) add any pause not yet pushed.

### The events to index

| Source | Events | Notes |
|---|---|---|
| **The vault** | `CollateralLocked` and `CollateralReleased`, once per leg; `BasketLocked`, once per lock after its legs; `CollateralReduced`, `CollateralEnlarged`, `LoanOwnershipTransferred`; `Deposited`, `WithdrawalInitiated`, `Reserved`, `ReservationReleased`, `ReservationReduced`, `ReservationSettled`, `Transferred` | A lock's id appears as `vaultLoanId`. `BasketLocked` carries `legCount`, so an indexer can confirm it has every leg |
| **Obligation** | `SetCreated` (with `creator`, `obligor`, `vault`, `lienContract`), `ObligationCreated`, `LienBound`, `Paid`, `Discharged`, `SetDischarged`, `Breached`, `SetSettled`, `Collected`, `Redeemed`, `Released`, `ShareReleased`, `PositionBurned`; `ContingentCreated`, `Fired`, `Exercised`, `Resolved`, `Lapsed`; the standard ERC-1155 events for claim units | The obligor position's ERC-721 `Transfer` comes from `ObligorPosition` |
| **The Lien** | `LienCreated`, `LienReleased`, `LienPaidOff`, `LienEnforced`, `CollateralTaken` and `CreditRecorded` (per asset), `Claimed` | |
| **The engines** | `AuctionOpened`, `AuctionController`, `BidPlaced`, `AuctionCleared`, `AuctionSettled`, `AuctionExpired`, `AuctionCancelled`: one stream, the same on all five engines | An auction id is unique per engine, so key auctions by (engine, id). `BatchAuction` also emits order and slot events |
| **Each `SpotMarket`** | `ClearingFound` (price, matched volume, low-volume flag), `SlotCleared`, `ClearingPageRun`, `ClearingRefused`, `OrderReleased` | `BatchAuction` emits `ClearingFound` and `SlotCleared` too, with different fields, and emits `SettlementAbandoned` when a clearing is abandoned; the market's own events come from the market's address |
| **Products** | Each product's page lists its events, such as the bullet loan's `LoanOpened`, `EntryRecorded` and `LoanOriginated` (which names the set and the lien) | |

### Rules a reader must apply

Anyone can call most of these contracts, so not every event is a product's.

1. **Filter auction events by controller.** Anyone can open an auction on an engine. An auction belongs to a product only if its `AuctionController` event names that product as `controller`.
2. **Check `lien.obligation()`.** A set can name any address as its lien contract. When you reach a Lien through `obligation.lienOf(setId)`, check that the Lien's `obligation()` is the Obligation you started from before trusting its records.
3. **Filter sets by `creatorOf`.** Anyone can create a set naming any address as obligor or holder, and no one can refuse it. A reader that attributes debts to an account, or counts a product's loans, counts only sets whose `creatorOf(setId)` (or `SetCreated.creator`) is a contract it trusts.
4. **Read `contingentStateOf` before trusting plain amounts.** Until a contingent obligation resolves, `getObligation`, `outstanding` and `debtAt` report its maximum. `contingentStateOf(id)` returns `NONE` for a plain obligation; anything else (`PENDING`, `FIRED`, `RESOLVED`, `LAPSED`) means the plain amounts are a ceiling until it is `RESOLVED`.
5. **Superseded contracts are on chain too.** A first `Obligation`, `Lien`, `ObligorPosition` and `AuctionOrchestrator`, and a first BOND package, are still deployed, and some remain admitted to the vault. The address list marks them superseded. Their events are separate from the current contracts' and are not part of the current record.

Two further points. **A recorded spot price is not a settled one**: a clearing abandoned during settlement keeps its price in the market's record, and only the events (`SettlementAbandoned` from `BatchAuction`, `ClearingRefused` from the market) tell it apart; see the [spot market](../products/spot-market.md). And **the test tokens can be burned from any holder**, the vault included, so on the testnet `ledgerTotal(asset)` can exceed what the vault holds.

### An example: a product's auctions

This reads every auction `BulletLoanRequest` has opened on `SingleAuction`, then checks a set it created. Many RPC providers cap the block range of one `eth_getLogs` call, so read long ranges in windows.

```js
import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL); // Arbitrum Sepolia, 421614

// From the address list:
const SINGLE_AUCTION = "0x...";
const BULLET_LOAN_REQUEST = "0x...";
const OBLIGATION = "0x...";

// Placeholders: the block range to read, a set id and an obligation id.
const fromBlock = 0;
const toBlock = "latest";
const setId = 1n;
const obligationId = 1n;

const engine = new ethers.Contract(SINGLE_AUCTION, [
  "event AuctionController(uint256 indexed auctionId, address indexed controller, address indexed principal)",
], provider);

// Rule 1: only auctions whose controller is the product.
const opened = await engine.queryFilter(
  engine.filters.AuctionController(null, BULLET_LOAN_REQUEST),
  fromBlock, toBlock,
);
const auctionIds = opened.map((e) => e.args.auctionId);

const obligation = new ethers.Contract(OBLIGATION, [
  "function creatorOf(uint256 setId) view returns (address)",
  "function lienOf(uint256 setId) view returns (address)",
  "function contingentStateOf(uint256 id) view returns (uint8)",
  "function outstanding(uint256 id) view returns (uint256)",
], provider);

// Rule 3: count the set only if the product created it.
const same = (a, b) => a.toLowerCase() === b.toLowerCase();
const trusted = same(await obligation.creatorOf(setId), BULLET_LOAN_REQUEST);

// Rule 2: the set's Lien must serve this Obligation. Zero means an unsecured set.
const lienAddress = await obligation.lienOf(setId);
let lienOk = false;
if (lienAddress !== ethers.ZeroAddress) {
  const lien = new ethers.Contract(lienAddress, [
    "function obligation() view returns (address)",
  ], provider);
  lienOk = same(await lien.obligation(), OBLIGATION);
}

// Rule 4: 0 (NONE) means a plain obligation; otherwise the amount is a ceiling until resolved.
const isPlain = (await obligation.contingentStateOf(obligationId)) === 0n;
const owed = await obligation.outstanding(obligationId);
```

The bullet loan's own `LoanOriginated(loanId, ref, setId, lienId)` gives the `setId` for each settled loan, and `originationOf(ref)` returns its obligation id.

## Reading from the testnet API

Aletheia Analytics SASU runs an indexer that follows the V2 testnet contracts and serves what it has indexed. It holds no keys and has no role in any contract. Every route is **testnet-only and subject to change**.

```
https://api-testnet.thegavel.io/v1/core/v2
```

At 14 November 2026 the routes are public and need no key. All are `GET` and return JSON. (For v1, see the [Aletheia API](../../data/aletheia-api.md).)

### The shared envelope

Every answer starts with the same fields:

| Field | Meaning |
|---|---|
| `indexedTo` | The last block the indexer has processed. Nothing later is in the answer; read the chain for blocks after it |
| `source` | `"indexer"` |
| `abiCommit` | An identifier of the ABI set the indexer decodes events with |
| `ownAccount` | `{ available, registered, note }`: whether account marking is available, the number of accounts in the provider's published register, and a note |

Many records also carry their own `ownAccount` flag: the API's annotation marking records that involve accounts in that register. It is `true` or `false` when marking is available, and `null` when it is not (meaning "cannot tell", not "no"). The flag is the API's annotation; nothing on chain records it.

Other conventions:

- **Integers are decimal strings.** Times (`at`) are unix seconds.
- **Lists are newest first**, except `/logs`, which is ascending like `eth_getLogs`; `/calendar`, ascending by date; and `/calendar/futures-offers`, ascending by reference time.
- **List size:** on routes that take `limit`, it defaults to 50 and is capped at 500 (on `/logs`, 1,000 and 5,000). `/calendar`, `/calendar/futures-offers`, `/bills/:k/backing`, `/bills/:k/policies`, `/accounts/:address/held-units` and `/accounts/:address/bond` return everything they match; `/spot/:market/last` looks at the market's latest 50 slots.
- **`product`** filters (on `/tape` and `/auctions`) take `bullet-loan`, `spot`, `bills`, `secondary`, `future`, `protection`, `liquidation` or `bond`.
- **Paging:** routes that page return `next` as `"block:logIndex"`; pass it back as `cursor`. `next` is `null` on the last page.
- **Errors** are a non-2xx status with `{ "error": { "code", "message" } }`, such as `invalid_address` (400) or `not_indexed` (404).
- Answers may be cached for up to 30 seconds; each still carries its own `indexedTo`.

### `/logs`: logs in `eth_getLogs` shape

`/logs` returns the logs the indexer holds for one contract and one event, in the shape `eth_getLogs` returns, so code that decodes logs from a node can decode these unchanged.

| Parameter | Meaning |
|---|---|
| `address` | Required. One contract |
| `event` | Required. An event name (for example `BidPlaced`) or its topic0 |
| `topics` | Optional. Positional filters on the indexed fields, comma-separated, each a 32-byte hex topic or empty for any (as `eth_getLogs`' `topics[1..3]`). One value per position; no OR lists |
| `fromBlock`, `toBlock` | Optional. `toBlock` is capped at `indexedTo`; the span is unlimited |
| `limit` | Optional. 1 to 5,000; default 1,000 |
| `cursor` | Optional. The previous page's `next` |

Each entry in `rows` has `address` (lowercase), `topics`, `data`, `blockNumber`, `transactionHash`, `transactionIndex`, `logIndex` and `blockHash`, as `eth_getLogs` gives them, plus `at` (block time) and `from` (the transaction's sender), which `eth_getLogs` does not. `topics` and `data` are rebuilt from the indexed event, so `iface.parseLog(log)` decodes each with your own ABI. The answer also names the `contract`, `event`, `topic0` and `coveredFrom` (the contract's deployment block, where coverage starts). No entry is ever above `indexedTo`.

**Joining with the chain.** For one filter from `start` to the chain's head: page through `/logs` until `next` is `null`, passing the first page's `indexedTo` as `toBlock` on later pages; then read the chain for the blocks after `indexedTo`; then concatenate. The two never overlap.

If the indexer does not follow the address, or the event is not in its ABI for that contract, `/logs` answers `404 not_indexed`. The shared vault is among these: read its events from the chain, or per account through `/accounts/:address/vault-events`.

### Routes

Testnet-only and subject to change. Paths are relative to the base URL.

| Path | Returns | Main query parameters |
|---|---|---|
| `/logs` | Logs for one contract and event, in `eth_getLogs` shape (above) | `address`, `event`, `topics`, `fromBlock`, `toBlock`, `limit`, `cursor` |
| `/events` | Decoded events (`args`) for one contract and event, newest first | `contract`, `event`, `fromBlock`, `toBlock`, `t1`–`t3` (match any field equal to the value), `limit`, `cursor` |
| `/tape` | One entry per user action across the products (`kind`, `product`, `contract`, `address`, `ids`, block and transaction) | `since` (unix time, within 90 days; default 7 days), `product`, `kind`, `limit`, `cursor` |
| `/auctions` | Product auctions, open or closed. Closed ones carry the clearing rate, bid quantity, distinct bidders, status, and `isPrice` with `notPriceReason` (a posted-offer take, or a clearing later expired, is not a price) | `state` (`open` or `closed`), `product`, `limit` |
| `/auctions/:engine/:auctionId/bids` | Every bid on one engine auction | |
| `/curve` | Bullet-loan clearing rates with their terms and set ids, and the latest clearing of each bill series' market, with the number of points, their time span and distinct counterparties | |
| `/bullet-loans/opened` | Bullet loans, with each origination's `ref`, `setId` and `lienId` | `opener`, `limit` |
| `/bullet-loans/entries` | Bids and takes recorded on bullet loans | `owner`, `limit` |
| `/spot/:market/batches` | A spot market's slots: clearing price, matched volume, low-volume flag, fills, orders placed | `limit` |
| `/spot/:market/last` | The market's latest cleared slot | |
| `/protection/offers` | Cover offers and their terms | `open=1`, `adapter`, `limit` |
| `/protection/policies` | Cover policies | `account`, `role` (`insured`, `beneficiary`, `seller`), `limit` |
| `/bills/:k/backing` | The bill packager's events for series `k` | |
| `/bills/:k/policies` | The policies deposited into series `k` | |
| `/calendar` | Dates ahead: loans due, futures' reference times, cover windows ending | `from`, `to` (default: now to 90 days) |
| `/calendar/futures-offers` | Future offers grouped by reference time | |
| `/futures` | Opened futures, with both legs' set and obligation ids | `account`, `limit` |
| `/secondary/sales` | Claim-lot and obligor-position sales | `market`, `state`, `limit` |
| `/secondary/:market/:auctionId/bids` | One sale's bids and escrow events | |
| `/accounts/:address/holdings` | Claim units held (including units escrowed in an open sale) and obligor positions | |
| `/accounts/:address/vault-events` | The account's vault events | `limit` |
| `/accounts/:address/held-units` | Units held for the account by a product, and the function that claims them | |
| `/accounts/:address/bond` | The account's BOND registrations, mints and burns | |

The API applies the chain rules above for you in its product views: `/auctions` lists only auctions whose controller is a followed product, and the tape counts core actions only on sets a listed product created.

### Examples

The ten latest bullet-loan actions:

```sh
curl "https://api-testnet.thegavel.io/v1/core/v2/tape?product=bullet-loan&limit=10"
```

The answer is the envelope plus `since`, `rows` and `next`. Each entry has a `kind` (such as `request-opened`, `bid` or `loan-settled`), the acting `address`, the emitting `contract`, the ids it concerns, and its `block`, `tx` and `logIndex`.

Closed auctions, with their clearing prices:

```sh
curl "https://api-testnet.thegavel.io/v1/core/v2/auctions?state=closed&product=bullet-loan&limit=20"
```

Each entry names the `engine`, `auctionId` and `controller`, and gives `clearingRate` as `{ num, den }`, `bidToCover`, `distinctBidders` and `status`. Count an entry as a price only where `isPrice` is `true`.

Every `AuctionController` log on `SingleAuction` naming the bullet loan's request product as controller, the second indexed field, padded to 32 bytes:

```sh
curl "https://api-testnet.thegavel.io/v1/core/v2/logs?address=<SINGLE_AUCTION>&event=AuctionController&topics=,0x000000000000000000000000<BULLET_LOAN_REQUEST without 0x>&limit=5000"
```

Each entry decodes with the `AuctionController` fragment from the example above. If `next` is not `null`, repeat with `cursor=<next>` and `toBlock=<indexedTo from the first page>`.

An account's claim units and obligor positions:

```sh
curl "https://api-testnet.thegavel.io/v1/core/v2/accounts/<ADDRESS>/holdings"
```

The answer lists `claims` (obligation id, units, set id, and `escrowedIn` for units in an open sale) and `positions` (set id, and `custody`: whether the account holds the position itself or it is held in the vault for the account).

## Next

- [Contracts reference](contracts-reference.md): every view, event and error.
- [Writing](writing.md): sending transactions.
- [Composing a product](composing-a-product.md): building on the blocks.
- [The base layer and what you trust](../concepts/base-layer-and-trust.md): why no indexer or API needs to be trusted.
