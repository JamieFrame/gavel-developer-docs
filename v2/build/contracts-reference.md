# Contracts reference

This page lists the callable surface of every deployed V2 contract: what each one is for, the state-changing functions a user or a product calls (with who may call them), the main views, the events and the custom errors. It is a reference, not a guide: each section links to the concept or product page that explains the behaviour.

**The full ABI and verified source of each contract are on Arbiscan.** Every entry in the [address list](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.md) (machine-readable: [`v2-testnet-addresses.json`](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.json)) links to its Arbiscan page. Struct layouts, return tuples and which event parameters are `indexed` are given there; this page gives types and names. Network and status are on the [deployments](../deployments.md) page. The contracts are **not audited**.

## Conventions

| Term | Meaning |
|---|---|
| **Anyone** | Any address; nothing is paid for calling unless stated |
| **Operator** | An address the named user approved on **that** contract with `setOperatorApproval(operator, true)`. A `…For(user, …)` function acts for `user`, who stays the owner of the order, bid, escrow or proceeds |
| **Controller** | On an engine: whoever opened the auction (in practice a product contract) |
| **Owner only** | Held by the account that deployed the testnet contracts. Ownership is two-step (`transferOwnership`, then `acceptOwnership` by the new owner). `renounceOwnership` reverts (`RenounceOwnershipDisabled`) everywhere except on `BondListing` |
| **Key** | A vault asset key (`bytes32`): `0x01`, eleven zero bytes and the token address for an ERC-20. See [the vault](../concepts/vault.md) |

Enum values used in calls:

| Enum | Values, in order (0, 1, …) |
|---|---|
| `PaySource` | `VAULT`, `WALLET` |
| `Prepayment` | `NONE`, `FULL`, `ACCRUED` |
| `ObligationStatus` | `OPEN`, `DISCHARGED`, `BREACHED`, `SETTLED`, `PAID_LATE`, `RELEASED` |
| `TriggerKind` | `UNTRIGGERED`, `STATE`, `RESOLVER`, `EXERCISE_HOLDER`, `EXERCISE_OBLIGOR` |
| `Side` (spot) | `BUY`, `SELL` |
| `Side` (future) | `LONG`, `SHORT` |
| `Opening` (bullet loan) | `REQUEST_ONE`, `REQUEST_MANY`, `OFFER_ONE`, `OFFER_MANY` |
| `Kind` (`BondListing`) | `LOAN_TOKEN`, `PRICED`, `SOURCE`, `MINT_CAP`, `MARKET` |

Shared errors: most contracts inherit `InvalidAddress`, `RenounceOwnershipDisabled` and `UnexpectedReceipt`. Owned contracts also emit the standard `OwnershipTransferStarted` and `OwnershipTransferred`. A call that reaches the vault, Obligation, the Lien or an engine can revert with that contract's error, which bubbles up unchanged.

Paused contracts that use the shared pause clock (`SpotMarket`, the secondary markets, `FutureOrchestrator`, `CoverOrchestrator`) emit `PauseStarted(uint256 timestamp)` and `PauseEnded(uint256 timestamp, uint256 durationAdded)` and expose `paused()`, `totalPausedDuration()` and `pauseElapsed(uint256 snapshot)`.

---

## The core

### Vault

Custody and nothing else: per-account balances (free, reserved, locked), reservations and collateral locks for approved consumers. No pause. See [the vault](../concepts/vault.md).

| Function | Who may call | Effect |
|---|---|---|
| `deposit(address user, bytes32 asset, uint256 amount, bytes params)` | `user` | Pulls the token from `user`'s wallet into free balance. `params` empty for an ERC-20 |
| `depositFor(address user, bytes32 asset, uint256 amount, bytes params)` | `user` or operator | The same; draws only on `user`'s wallet |
| `withdraw(address user, bytes32 asset, uint256 amount, bytes externalDestination) returns (bytes32 withdrawalId)` | `user` | Sends free balance to the ABI-encoded destination |
| `withdrawFor(address user, bytes32 asset, uint256 amount) returns (bytes32 withdrawalId)` | `user` or operator | The same, paid only to `user` |
| `approveConsumer(address consumer, bool approved)` | Any account, for itself | Lets a whitelisted consumer act on your account |
| `setOperatorApproval(address operator, bool approved)` | Any account, for itself | Lets `operator` call `depositFor` and `withdrawFor` for you |
| `setAcceptsEnlargement(bool accepted)` | A lock owner | Opts every lock it owns in or out of enlargement |
| `enlargeLock(uint256 vaultLoanId, uint256[] increases)` | Anyone, from own free balance, if the lock's owner opted in | Adds to existing legs; the addition becomes the depositor's |

Called by consumers (products, Obligation, the Lien), not by users:

| Function | Who may call |
|---|---|
| `reserve(address user, bytes32 asset, uint256 amount, uint256 contextId) returns (bool)` | A whitelisted consumer `user` approved |
| `releaseReservation(address user, bytes32 asset, uint256 contextId) returns (uint256 released)` | The consumer holding the reservation |
| `reduceReservation(address user, bytes32 asset, uint256 contextId, uint256 newAmount) returns (bool)` | The consumer holding it |
| `settleReservation(address from, address to, bytes32 asset, uint256 contextId, uint256 amount) returns (bool)` | The consumer holding it, while whitelisted |
| `transfer(address from, address to, bytes32 asset, uint256 amount) returns (bool)` | A consumer moving its own balance (if ever whitelisted), or a whitelisted consumer `from` approved |
| `lockCollateral(bytes params) returns (uint256 vaultLoanId)` | A whitelisted consumer the depositor approved |
| `releaseCollateral(uint256 vaultLoanId, bytes recipient) returns (bool)` | The lock's owner |
| `reduceLock(uint256 vaultLoanId, uint256[] newAmounts)` | The lock's owner |
| `transferLoanOwnership(uint256 vaultLoanId, address newOwner)` | The lock's owner |
| `enlargeLockFor(address contributor, uint256 vaultLoanId, uint256[] increases)` | A whitelisted consumer `contributor` approved |

**Owner only:** `setAuthorisedConsumer(address consumer, bool authorised)`, `emergencyDeauthoriseAll()`.

**Views:** `balance(user, asset)` (free), `totalBalance(user, asset)`, `reservedBalance(consumer, user, asset, contextId)`, `ledgerTotal(asset)`, `isAuthorisedConsumer(consumer)`, `isUserApproved(user, consumer)`, `isOperator(user, operator)`, `authorisedConsumerCount()`, `authorisedConsumerAt(index)`, `wasEverAuthorised(account)`, `getLoanRecord(vaultLoanId)`, `getLoanLegs(vaultLoanId)`, `getDepositor(vaultLoanId)`, `getLoanOwner(vaultLoanId)`, `getPreviousOwner(vaultLoanId)`, `isCollateralReady(vaultLoanId)`, `acceptsEnlargement(lockOwner)`, `getNFTDetail(key)`, `owner()`.

**Events:** `Deposited(address user, bytes32 asset, uint256 amount)`; `WithdrawalInitiated(address user, bytes32 asset, bytes32 withdrawalId, uint256 amount, bytes externalDestination)`; `Reserved(address consumer, address user, bytes32 asset, uint256 contextId, uint256 amount)`; `ReservationReleased(address consumer, address user, bytes32 asset, uint256 contextId, uint256 amount)`; `ReservationReduced(address consumer, address user, bytes32 asset, uint256 contextId, uint256 previousAmount, uint256 newAmount)`; `ReservationSettled(address from, address to, bytes32 asset, address consumer, uint256 contextId, uint256 amount)`; `Transferred(address from, address to, bytes32 asset, address consumer, uint256 amount)`; `CollateralLocked(uint256 vaultLoanId, address depositor, bytes32 asset, uint256 amount)`; `BasketLocked(uint256 vaultLoanId, address depositor, uint256 legCount)`; `CollateralReleased(uint256 vaultLoanId, address recipient, bytes32 asset, uint256 amount)`; `CollateralReduced(uint256 vaultLoanId, bytes32 asset, uint256 previousAmount, uint256 newAmount)`; `CollateralEnlarged(uint256 vaultLoanId, address contributor, bytes32 asset, uint256 previousAmount, uint256 newAmount)`; `LoanOwnershipTransferred(uint256 vaultLoanId, address previousOwner, address newOwner)`; `EnlargementAcceptanceSet(address owner, bool accepted)`; `ConsumerAuthorisationSet(address consumer, bool authorised)`; `ConsumerApprovalSet(address user, address consumer, bool approved)`; `OperatorApprovalSet(address user, address operator, bool approved)`.

**Errors:** `InvalidAmount`, `InvalidAsset`, `InvalidContextId`, `ConsumerNotAuthorised`, `UserHasNotApproved`, `InsufficientBalance`, `ReservationNotFound`, `ReservationAlreadyExists`, `InsufficientReservation`, `InvalidReservationReduction`, `SelfTransferNotPermitted`, `LoanNotFound`, `LoanAlreadyReleased`, `NotLoanOwner`, `EmptyBasket`, `TooManyLegs`, `LegLengthMismatch`, `InvalidReduction`, `AssetsNotStrictlyAscending`, `EnlargementNotAccepted`, `LedgerTotalExceeded`, `UnexpectedDepositAmount`, `InvalidNFTAmount`, `NFTNotHeld`, `AssetMismatch`, `NotAccountHolder`, `NoAuthorisedConsumers`, `UnsolicitedNFT`, `NotApprovedOperator`.

### Obligation (with ObligationLogic and ModuleLogic)

The register of debts, and the ERC-1155 claim token. `ObligationLogic` and `ModuleLogic` are linked libraries holding part of its code: they have no state of their own and are not called directly. See [Obligation](../concepts/obligation.md) and [claims and obligor positions](../concepts/claims-and-positions.md).

| Function | Who may call | Effect |
|---|---|---|
| `createSet(address obligor, uint64 grace, address vault, address lienContract, address releaseGate, ObligationInput[] obligations) returns (uint256 setId, uint256[] ids)` | Anyone (becomes the creator) | Creates a set; mints claim units and the obligor position. `ObligationInput` is `(asset, principal, interest, accrualStart, dueAt, prepayment, prepaymentChargeCap, holders[], units[])` |
| `createContingentSet(address obligor, uint64 grace, address vault, address lienContract, address releaseGate, ObligationInput[] obligations, ContingentTerms terms) returns (uint256 setId, uint256 id)` | Anyone | The same for one obligation with modules. `ContingentTerms` is `(kind, triggerTarget, triggerData, windowStart, windowEnd, dueInterval, resolveAt, resolver, resolverData, minAmount, maxAmount, fallbackToMax)` |
| `pay(uint256 obligationId, uint256 amount, PaySource source)` | Anyone | Pays up to `amount`, charging only the payoff. Whole payoff only on a secured set |
| `markBreached(uint256 obligationId)` | Anyone | After `dueAt + grace` (pause-adjusted); breaches the whole set |
| `collect(uint256 obligationId, bytes32 key, address destination, bool toVault) returns (uint256 amount)` | A holder, or former holder with credit | Pays out the caller's accrued credit in one asset |
| `collectFor(address holder, uint256 obligationId, bytes32 key) returns (uint256 amount)` | Operator of `holder` | The same, into `holder`'s vault account |
| `redeem(uint256 obligationId, address destination, bool toVault)` | A holder, once final | Collects every asset and burns the caller's units |
| `close(uint256 setId)` | Anyone | Burns the obligor position of an ended unsecured set |
| `release(uint256 obligationId, uint256 units)` | A holder | Burns units, forgiving that share of the debt |
| `releaseFor(address holder, uint256 obligationId, uint256 units)` | Address approved by `setReleaseApproval` | The same, for `holder` |
| `releaseShare(uint256 obligationId, uint256 shareWad)` | The set's `releaseGate` | Forgives a share of what every unit is owed |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves `collectFor` |
| `setReleaseApproval(address operator, bool approved)` | Any account | Approves `releaseFor` |
| `fire(uint256 id)` | Anyone | Brings a contingent obligation into force if its trigger is met in the window |
| `exercise(uint256 id)` | The exerciser (holder of every unit, or of the position) | Fires an `EXERCISE_…` obligation |
| `resolve(uint256 id)` | Anyone | Fixes a fired obligation's amount within its bounds |
| `lapse(uint256 id)` | Anyone | Discharges an obligation whose window ended unfired |
| `bindLien(uint256 setId, bool takeMode, bool wholeOnly)` | The set's lien contract | Marks the set secured |
| `markSettled(uint256 setId, uint256 enforcedAt, uint256 enforcedPause, bytes32[] keys, uint256[] amounts, bool hasValued, uint256 valuedShortfall, uint256[] prices, bool take) returns (uint256[] excess)` | The set's lien contract | Records enforcement and credits what was taken |
| `burnPosition(uint256 setId)` | The set's lien contract | Burns the position of a finished secured set |

Claim units are standard ERC-1155: `safeTransferFrom`, `safeBatchTransferFrom`, `setApprovalForAll`, `balanceOf`, `balanceOfBatch`, `isApprovedForAll`, `uri`.

**Owner only:** `pause()`, `unpause()` (stop only `createSet` and `createContingentSet`).

**Views:** `getSet(setId)`, `getObligation(id)`, `vaultOf(setId)`, `lienOf(setId)`, `creatorOf(setId)`, `obligorOf(setId)`, `isBound(setId)`, `ownerOfPosition(setId)`, `positionState(setId)`, `breachOf(setId)`, `settledAsTake(setId)`, `valueAt(id, t)`, `outstanding(id)`, `payoff(id)`, `debtAt(setId, t)`, `accrued(id, holder, key)`, `creditKeys(id)`, `unitsOutstanding(id)`, `heldInVault(vault, key)`, `releaseScaleOf(id)`, `forgivenAfterBreachOf(id)`, `reportedShortfall(id)`, `isOperator(user, operator)`, `isReleaseApproved(holder, operator)`, `pausedTotal()`; contingent: `contingentStateOf(id)`, `isContingentSet(setId)`, `triggerOf(id)`, `windowOf(id)`, `amountBoundsOf(id)`, `resolvedAmountOf(id)`, `resolverOf(id)`.

**Events:** `SetCreated(uint256 setId, address creator, address obligor, uint64 grace, address vault, address lienContract, address releaseGate)`; `ObligationCreated(uint256 obligationId, uint256 setId, bytes32 asset, uint256 principal, uint256 interest, uint64 accrualStart, uint64 dueAt, Prepayment prepayment, uint256 prepaymentChargeCap)`; `Paid(uint256 obligationId, address payer, uint256 amount, PaySource source, bool late)`; `Discharged(uint256 obligationId)`; `PaidLate(uint256 obligationId)`; `SetDischarged(uint256 setId)`; `Breached(uint256 obligationId, uint256 setId)`; `LienBound(uint256 setId, address lienContract, bool takeMode, bool wholeOnly)`; `SetSettled(uint256 setId, uint256 enforcedAt, uint256 total, bool take)`; `ShortfallRecorded(uint256 obligationId, uint256 shortfall)`; `SettlementValued(uint256 setId, uint256 valuedShortfall, bytes32[] keys, uint256[] prices)`; `Collected(uint256 obligationId, address holder, bytes32 key, address destination, bool toVault, uint256 amount)`; `Redeemed(uint256 obligationId, address holder, uint256 units)`; `PositionBurned(uint256 setId)`; `Released(uint256 obligationId, address holder, uint256 units, uint256 releaseScale)`; `ShareReleased(uint256 obligationId, address gate, uint256 shareWad, uint256 releaseScale)`; `ObligationReleased(uint256 obligationId)`; `ShortfallReduced(uint256 obligationId, uint256 amount)`; `OperatorApprovalSet(address user, address operator, bool approved)`; `ReleaseApprovalSet(address holder, address operator, bool approved)`; `PauseStarted(uint256 at)`; `PauseEnded(uint256 at, uint256 durationAdded)`; `ContingentCreated(uint256 obligationId, uint256 setId, TriggerKind kind, address triggerTarget, bytes triggerData, uint64 windowStart, uint64 windowEnd, uint64 dueInterval, uint64 resolveAt, address resolver, bytes resolverData, uint256 minAmount, uint256 maxAmount, bool fallbackToMax, address[] targets)`; `Fired(uint256 obligationId, uint256 dueAt)`; `Resolved(uint256 obligationId, uint256 amount, bool usedFallback)`; `Lapsed(uint256 obligationId)`; `Exercised(uint256 obligationId, address exerciser)`; and the ERC-1155 `TransferSingle`, `TransferBatch`, `ApprovalForAll`.

**Errors:** `InvalidAsset`, `InvalidAmount`, `InvalidGrace`, `InvalidTerms`, `InvalidStatus`, `InvalidUnits`, `InvalidKeys`, `TooManyObligations`, `TooManyMints`, `TooManyAssets`, `ClaimsNotDepositable`, `PositionNotDepositable`, `NotDue`, `NotFinal`, `NotSetLien`, `NotReleaseGate`, `NotApprovedOperator`, `NotHeldWhole`, `NotWholeTransfer`, `NFTCreditUncollected`, `MixedSettlementAssets`, `PaymentAboveOwed`, `PrepaymentNotAllowed`, `MustPayWhole`, `CreditNotArrived`, `VaultTransferFailed`, `Paused`; contingent: `InvalidContingentTerms`, `NotResolved`, `NotResolvable`, `WindowNotOpen`, `WindowNotEnded`, `TriggerConditionNotMet`, `TriggerConditionMet`, `InsufficientResolverGas`, `ResolverHookFailed`, `TooManyTargets`, `NotExerciser`, `ExerciseRefused`.

### ObligorPosition

The ERC-721 obligor position; the token id is the set id. Only Obligation mints and burns it. See [claims and obligor positions](../concepts/claims-and-positions.md).

| Function | Who may call | Effect |
|---|---|---|
| `transferFrom`, `safeTransferFrom`, `approve`, `setApprovalForAll` | Holder or approved | Standard ERC-721; moves the debt and the equity in its collateral |
| `mint(address to, uint256 setId)`, `burn(uint256 setId)` | Obligation only | At set creation and at the set's end |

**Owner only:** `setObligation(address obligation_)`, used once at deployment; any later call reverts `ObligationAlreadySet`.

**Views:** `ownerOf(setId)`, `exists(setId)`, `obligation()`, `balanceOf`, `getApproved`, `isApprovedForAll`. **Events:** `ObligationSet(address obligation)`; ERC-721 `Transfer`, `Approval`, `ApprovalForAll`. **Errors:** `NotObligation`, `ObligationAlreadySet`, `PositionNotDepositable`, `InvalidStatus`.

### Lien

Binds one vault lock to one set; releases it on payment, or passes it in kind to the creditors on breach (TAKE only). See [the Lien](../concepts/lien.md).

| Function | Who may call | Effect |
|---|---|---|
| `createLien(LienTerms terms, bytes extra) returns (uint256 lienId)` | The set's creator, having handed the lock to the Lien | Binds lock `(vault, lockId)` to `setId`. `extra` is empty |
| `release(uint256 lienId)` | Anyone | Set `DISCHARGED`: the lock becomes credit for the position's holder |
| `payoff(uint256 lienId, uint256 maxAmount, PaySource source)` | Anyone | Set `BREACHED`, not enforced: pays the debt now; the lock becomes credit |
| `enforce(uint256 lienId)` | Anyone | Set `BREACHED`: TAKE; the remainder becomes credit |
| `claim(uint256 lienId)` | The obligor position's holder | Moves the credit to the caller's vault account and burns the position |

**Owner only:** `pause()`, `unpause()` (stop only `createLien`).

**Views:** `getLien(lienId)` (vault, lockId, setId, settlementAsset, status), `lienOfSet(setId)`, `lienOfLock(vault, lockId)`, `creditOf(lienId, key)`, `creditKeys(lienId)`, `obligation()`.

**Events:** `LienCreated(uint256 lienId, uint256 setId, address vault, uint256 lockId, bool wholeOnly)`; `LienReleased(uint256 lienId)`; `LienPaidOff(uint256 lienId, address payer, uint256 amount)`; `LienEnforced(uint256 lienId, uint256 debt)`; `CollateralTaken(uint256 lienId, bytes32 key, uint256 amount)`; `CreditRecorded(uint256 lienId, bytes32 key, uint256 amount)`; `Claimed(uint256 lienId, address owner, bytes32 key, uint256 amount)`; `PauseStarted(uint256 at)`; `PauseEnded(uint256 at)`.

**Errors:** `NotSetCreator`, `NotSetLien`, `VaultMismatch`, `LockNotOwned`, `LockInUse`, `DepositorNotObligor`, `LockNotHandedOver`, `InvalidTerms`, `InvalidStatus`, `SetNotDischarged`, `SetNotBreached`, `PayoffAboveMax`, `NotPositionOwner`, `NothingToClaim`, `UnexpectedDepositAmount`, `Paused`, `VaultTransferFailed`.

---

## The engines

Engines find a price and hold nothing; users do not call them directly. In every engine, **anyone may open** an auction or market and becomes its controller; **every other write is controller-only**, except `close` (`SingleAuction`, `GroupAuction`) and `end` (`PostedOffer`, `ClockAuction`), which anyone may also call once no further pause could reopen the auction. No engine has an owner or a pause. See [the engines](../concepts/engines.md).

Shared events (all five engines): `AuctionOpened(uint256 auctionId, Kernel kernel, Better better, bytes32 quoteAsset, uint256 size, address itemContract, uint256 itemId, uint256 term, uint256 start, uint256 end, uint256 limitNum, uint256 limitDen, uint256 boundNum, uint256 boundDen)`; `BidPlaced(uint256 auctionId, address bidder, uint256 priceNum, uint256 priceDen, uint256 size)`; `AuctionCleared(uint256 auctionId, uint256 clearingNum, uint256 clearingDen, uint256 volume, uint256 fills)`; `AuctionSettled(uint256 auctionId, address settledIn, uint256 settledId)`; `AuctionExpired(uint256 auctionId, address claimer)`; `AuctionCancelled(uint256 auctionId)`; `AuctionController(uint256 auctionId, address controller, address principal)`. Filter by `AuctionController` to attribute an auction to a product.

### SingleAuction

One winner; bids improve up or down by a step.

| Function | Effect |
|---|---|
| `open(SingleInputs inputs, address principal) returns (uint256 auctionId)` | Opens an auction |
| `bid(uint256 auctionId, address bidder, uint256 price, uint256 quantity, bytes32 ref, bytes32[] proof) returns (uint256 bidId, Displaced displaced, bool closedAtBound)` | Records a bid; returns the bid it displaced |
| `cancel(uint256 auctionId, bytes32 reason)` | Cancels an open auction, as its cancel rule allows (for example, only before the first bid) |
| `shift(uint256 auctionId, uint256 seconds_) returns (uint256 applied)` | Pushes paused time into the end (30-day cap) |
| `close(uint256 auctionId) returns (Result)` | Fixes the result after the end |
| `finalise(uint256 auctionId, address settledIn, uint256 settledId)` | Records where the result was settled |
| `expire(uint256 auctionId)` | Marks an ended or closed auction expired |

**Views:** `getAuction`, `standing`, `getBid`, `result`, `effectiveEnd`, `minimumNextBid`, `phase`; constants `MAX_START_DELAY`, `MIN_DURATION`, `MAX_DURATION`, `MAX_EXTENSION`, `MAX_STEP_BPS`, `MAX_SHIFT`, `PRICE_SCALE`. **Events:** `BidDisplaced(uint256 auctionId, uint256 bidId)`, `AuctionExtended(uint256 auctionId, uint256 newEnd)`, `AuctionClosedAtBound(uint256 auctionId)`, `AuctionShifted(uint256 auctionId, uint256 seconds_, uint256 newEnd)`, `AuctionCancelReason(uint256 auctionId, bytes32 reason)`. **Errors:** `InvalidInputs`, `NotController`, `AuctionNotFound`, `NotOpen`, `NotLive`, `AuctionNotEnded`, `PrincipalCannotBid`, `ZeroBidder`, `NotAllowed`, `QuantityOutOfRange`, `OwnBidStanding`, `CancelNotAllowed`, `NotClosed`, `NotClearedResult`, and the bid-rule errors `AtBound`, `BeyondBound`, `BeyondLimit`, `StepNotMet`.

### GroupAuction

A ranked book with up to 50 winners at one clearing price; closes in pages.

| Function | Effect |
|---|---|
| `open(GroupInputs inputs, address principal) returns (uint256 auctionId)` | Opens an auction |
| `bid(uint256 auctionId, address bidder, uint256 price, uint256 quantity, uint256 minFill, uint256 hintPrev, bytes32 ref, bytes32[] proof) returns (uint256 bidId, bool closedAtBound)` | One bid per bidder, placed after `hintPrev` |
| `amend(uint256 auctionId, uint256 bidId, uint256 price, uint256 quantity, uint256 minFill, uint256 hintPrev, bytes32 ref) returns (bool closedAtBound)` | Improves a bid |
| `withdraw(uint256 auctionId, uint256 bidId)`, `void(uint256 auctionId, uint256 bidId, bytes32 reason)` | Removes a bid, where the auction's rules allow |
| `cancel(uint256 auctionId, bytes32 reason)`, `shift(uint256 auctionId, uint256 seconds_) returns (uint256)` | As `SingleAuction` |
| `close(uint256 auctionId, uint256 maxSteps) returns (Page)` | One page of the close |
| `finalise(uint256 auctionId, address settledIn, uint256 settledId)`, `expire(uint256 auctionId)` | As `SingleAuction` |

**Views:** `getAuction`, `getBid`, `fillOf`, `bidOf(auctionId, bidder)`, `bidsOf`, `book`, `findHint`, `wouldFill`, `winnersFrom`, `result`, `closeProgress`, `effectiveEnd`, `phase`; constants `MAX_WINNERS`, `MAX_BOUND_STEPS`, `MAX_HINT_STEPS`, `MAX_START_DELAY`, `MIN_DURATION`, `MAX_DURATION`, `MAX_EXTENSION`, `MAX_STEP_BPS`, `MAX_SHIFT`, `PRICE_SCALE`. **Events:** `BidAmended(uint256 auctionId, uint256 bidId)`, `BidWithdrawn(uint256 auctionId, uint256 bidId)`, `BidVoided(uint256 auctionId, uint256 bidId, bytes32 reason)`, `AuctionExtended`, `AuctionClosedAtBound`, `AuctionShifted`, `AuctionCancelReason`. **Errors:** `InvalidInputs`, `NotController`, `AuctionNotFound`, `NotOpen`, `NotLive`, `AuctionNotEnded`, `PrincipalCannotBid`, `ZeroBidder`, `NotAllowed`, `QuantityOutOfRange`, `CancelNotAllowed`, `NotClosed`, `NotClearedResult`, `MinFillOutOfRange`, `OneBidPerBidder`, `AmendNotAllowed`, `NotImprovement`, `WithdrawNotAllowed`, `BidNotInBook`, `VoidNotAllowed`, `ZeroReason`, `BeyondBound`, `BeyondLimit`; book errors `StaleHint`, `InvalidId`, `AlreadyLinked`, `NotLinked`.

### PostedOffer

A maker posts a price; takers take some or all until it ends.

| Function | Effect |
|---|---|
| `open(PostedInputs inputs, address principal) returns (uint256 offerId)` | Posts an offer |
| `take(uint256 offerId, address taker, uint256 quantity, uint256 priceBound, bytes32 ref, bytes32[] proof) returns (uint256 price, uint256 filled, uint256 remaining_)` | Takes part or all |
| `reprice(uint256 offerId, uint256 newPrice)` | Changes the price, where allowed |
| `withdraw(uint256 offerId) returns (uint256 remaining_)` | Withdraws an open offer |
| `end(uint256 offerId) returns (uint256 remaining_)` | Ends the offer after its end |
| `shift(uint256 offerId, uint256 seconds_) returns (uint256 applied)` | Pushes paused time |

**Views:** `getOffer`, `phase`, `remaining`, `takeOf(offerId, index)`; constants `MAX_START_DELAY`, `MIN_DURATION`, `MAX_OFFER_DURATION`, `MAX_SHIFT`, `PRICE_SCALE`. **Events:** `OfferRepriced(uint256 offerId, uint256 oldPrice, uint256 newPrice)`, `AuctionShifted(uint256 offerId, uint256 seconds_, uint256 newEnd)`. **Errors:** `InvalidInputs`, `NotController`, `OfferNotFound`, `NotOpen`, `NotLive`, `NotEnded`, `PrincipalCannotTake`, `ZeroTaker`, `NotAllowed`, `PriceBoundNotMet`, `TakeOutOfRange`, `InsufficientRemaining`, `RepriceNotAllowed`.

### BatchAuction

Two-sided books cleared in time slots at one uniform price. No principal and no gate; the controller places orders for their owners.

| Function | Effect |
|---|---|
| `openMarket(MarketInputs m) returns (uint256 marketId)` | Opens a market |
| `placeOrder(uint256 marketId, address owner, Side side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 hintPrev) returns (uint256 orderId, uint256 reserveAmount)` | Adds an order |
| `cancelOrder(uint256 marketId, address owner, uint256 orderId) returns (bool unlinked, uint256 queuedInSlot)` | Removes it, or queues the removal |
| `clear(uint256 marketId, uint256 slot, uint256 maxSteps, bool canRecord) returns (Page)` | One page of a slot's clearing |
| `expireSlot(uint256 marketId, uint256 slot)`, `abandonClearing(uint256 marketId, uint256 slot) returns (uint256 settledVolume)` | Ends an empty slot; abandons a clearing |
| `drain(uint256 marketId, uint256 maxEntries) returns (Released[] released, uint256 processed)` | Releases cancelled and expired orders of finished slots |
| `shift(uint256 marketId, uint256 seconds_) returns (uint256 applied)` | Pushes paused time into the slot |

**Views:** `getMarket`, `getOrder`, `getSlot`, `getWalk`, `bestOrder`, `getBook`, `findHint`, `getCurrentSlot`, `clearingSlot`, `activeOrderCount`, `getSlotCancelQueue`, `getSlotExpiryList`, `drainPosition`, `quoteForBase`, `getCurrentSlotClearableAt`, `getCurrentSlotMaxExpiresAt`, `openOrdersOf`; constants `PRICE_PRECISION`, `MAX_DRAIN_AT_CLOSE`, `MAX_DRAIN_PER_CALL`, `MAX_HINT_STEPS`, `MIN_SLOT`, `MAX_SLOT`, `MAX_SHIFT`, `MAX_ORDERS_PER_OWNER`. **Events:** `MarketOpened(uint256 marketId, address controller, MarketInputs inputs)`, `MarketShifted(uint256 marketId, uint256 applied, uint256 shiftTotal)`, `OrderPlaced(uint256 orderId, address owner, uint256 slotId, Side side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 prevOrderId)`, `OrderCancelQueued(uint256 orderId, uint256 queuedInSlot)`, `OrderCancelled(uint256 orderId)`, `OrderExpired(uint256 orderId)`, `OrderFilled(uint256 orderId, uint256 fillAmount, uint256 remainingAfterFill, bool fullyFilled)`, `OrderRemainderClosed(uint256 orderId, uint256 remainder)`, `SlotOpened(uint256 slotId, uint256 startTime)`, `SlotClosed(uint256 slotId)`, `ClearingFound(uint256 slotId, uint256 clearingPrice, uint256 matchedBaseVolume)`, `SlotCleared(uint256 slotId, uint256 clearingPrice, uint256 matchedBaseVolume)`, `SettlementAbandoned(uint256 slotId, uint256 settledVolume)`, `SlotExpiredEmpty(uint256 slotId, bool forcedByMaxDuration)`. **Errors:** `InvalidAmount`, `InvalidPrice`, `OrderBelowMinSize`, `OrderBelowMinQuote`, `ExpiresAtNotInFuture`, `OrderNotFound`, `OrderNotOpen`, `NotCurrentSlot`, `SlotNotClearing`, `SlotNotDrained`, `PreviousSlotNotFinished`, `NothingToClear`, `NotOrderOwner`, `SlotNotClearableYet`, `UserOrderCapReached`, `NotController`, `MarketNotFound`, `InvalidInputs`, `NotInMarket`, `RecordRefused`, `ZeroOwner`; book errors `StaleHint`, `InvalidId`, `AlreadyLinked`, `NotLinked`.

### ClockAuction

A price that moves with time until accepted, in parts. Deployed; no product on the testnet uses it.

| Function | Effect |
|---|---|
| `open(ClockInputs inputs, address principal) returns (uint256 auctionId)` | Opens a clock |
| `accept(uint256 auctionId, address taker, uint256 quantity, uint256 priceLimit, bytes32 ref, bytes32[] proof) returns (uint256 price, uint256 filled, uint256 remaining_)` | Accepts at the current price |
| `cancel(uint256 auctionId, bytes32 reason)`, `end(uint256 auctionId) returns (uint256 remaining_)`, `shift(uint256 auctionId, uint256 seconds_) returns (uint256 applied)` | As above |

**Views:** `priceAt(auctionId, t)`, `getAuction`, `phase`, `remaining`, `acceptanceOf(auctionId, index)`; constants `MAX_START_DELAY`, `MIN_DURATION`, `MAX_DURATION`, `MAX_STEPS`, `MAX_SHIFT`, `PRICE_SCALE`. **Events:** `AuctionShifted`, `AuctionCancelReason`. **Errors:** `InvalidInputs`, `NotController`, `AuctionNotFound`, `NotOpen`, `NotLive`, `NotEnded`, `PrincipalCannotAccept`, `ZeroTaker`, `NotAllowed`, `PriceLimitNotMet`, `AcceptOutOfRange`, `InsufficientRemaining`, `CancelNotAllowed`.

### SealedFront

A commit-reveal helper in front of `GroupAuction` or `SingleAuction`; finds no price. Deployed; no product on the testnet uses it.

| Function | Effect |
|---|---|
| `open(address engine, uint256 engineAuctionId, uint256 commitEnd, uint256 revealEnd, uint256 maxCommitsPerBidder) returns (uint256 sealedId)` | Opens commit and reveal windows over an engine auction |
| `commit(uint256 sealedId, address bidder, bytes32 commitment) returns (uint256 index)` | Records a hash |
| `reveal(uint256 sealedId, uint256 index, Bid bid, bytes32 salt, uint256 engineBidId, bytes4 refusal)` | Checks the reveal and records the engine's outcome |
| `shift(uint256 sealedId, uint256 seconds_) returns (uint256 applied)` | Pushes paused time |

**Views:** `getSealed`, `getCommitment`, `commitCountOf`, `phase`, `hashOf(sealedId, bidder, bid, salt)`; constants `SEALED_DOMAIN`, `MIN_COMMIT_DURATION`, `MAX_COMMIT_DURATION`, `MIN_REVEAL_DURATION`, `MAX_REVEAL_DURATION`, `MAX_COMMITS_PER_BIDDER`, `MAX_SHIFT`. **Events:** `SealedOpened(uint256 sealedId, address controller, address engine, uint256 engineAuctionId, uint256 commitEnd, uint256 revealEnd)`, `Committed(uint256 sealedId, uint256 index, address bidder, bytes32 commitment)`, `Revealed(uint256 sealedId, uint256 index, uint256 engineBidId)`, `RevealRefused(uint256 sealedId, uint256 index, bytes4 reason)`, `SealedShifted(uint256 sealedId, uint256 applied)`. **Errors:** `InvalidInputs`, `NotController`, `SealedNotFound`, `NotCommitting`, `NotRevealing`, `ZeroBidder`, `ZeroCommitment`, `TooManyCommits`, `CommitmentNotFound`, `AlreadyRevealed`, `BadReveal`.

---

## The products

### BulletLoanRequest and BulletLoanOffer

Fixed-rate, fixed-term secured loans: the borrower's openings (`BulletLoanRequest`) and the lender's (`BulletLoanOffer`), on `SingleAuction` and `GroupAuction`. See [bullet loan](../products/bullet-loan.md).

| Function | Contract | Who may call | Effect |
|---|---|---|---|
| `openRequest(address borrower, RequestTerms t, bool many) returns (uint256 loanId)` | Request | `borrower` or operator | Locks the collateral and opens the auction. `RequestTerms` is `(token, lockParams, amount, minAmount, maxRepayment, bidStep, term, duration)` |
| `bidRequest(address lender, uint256 loanId, uint256 quantity, uint256 repayment, uint256 minFill, uint256 hintPrev, bytes32 expectedTerms) returns (uint256 entryId)` | Request | `lender` or operator | Reserves the amount lent and bids a repayment |
| `openOffer(address lender, OfferTerms t, bool many) returns (uint256 loanId)` | Offer | `lender` or operator | Reserves the principal and opens the auction. `OfferTerms` is `(token, collateral, principal, minLoan, maxLoan, loanToCollateral, floorPrice, ceilingPrice, maxFills, term, duration)` |
| `bidOffer(address borrower, uint256 loanId, uint256 loanSize, uint256 repayment, uint256 minFill, uint256 hintPrev, bytes32 expectedTerms) returns (uint256 entryId)` | Offer | `borrower` or operator | Locks collateral for the loan size and bids |
| `executeFill(uint256 loanId, uint256 entryId)` | Offer | Anyone | Settles one winner of an offer to many |
| `finalise(uint256 loanId, uint256 maxSteps)` | Both | Anyone | Closes the auction (in pages for many) and settles |
| `cancel(uint256 loanId)` | Both | The opener or its operator | Ends a loan that never had a bid |
| `release(uint256 entryId)` | Both | Anyone | Releases a losing, stalled or ended bid's escrow |
| `expire(uint256 loanId)` | Both | Anyone | Releases the opener's escrow after the deadline, or after an auction with no escrowed bid |
| `setOperatorApproval(address operator, bool approved)` | Both | Any account | Approves the `address`-first calls above |

`closePage(uint256 loanId, uint256 maxSteps)` is callable only by the contract itself (`OnlySelf`). `expectedTerms` is `keccak256(abi.encode(terms, many))`.

**Owner only:** `pause()`, `unpause()`.

**Views:** `loanOf(loanId)`, `deadlineOf(loanId)`, `entryOf(entryId)`, `liveEntryOf(loanId, account)`, `winnersOf(loanId)`, `originationOf(ref)`, `bondOrigination(ref)`, `originator()`, `pauseCount()`, `pauseAt(i)`, `isOperator(user, operator)`, `paused()`; constants as listed on the product page.

**Events:** `LoanOpened(uint256 loanId, Opening opening, address opener, uint256 auctionId)`; `EntryRecorded(uint256 loanId, uint256 entryId, address owner, uint256 size, uint256 repayment)`; `EntryReleased(uint256 loanId, uint256 entryId)`; `LoanOriginated(uint256 loanId, uint256 ref, uint256 setId, uint256 lienId)`; `LoanEnded(uint256 loanId)`; `PauseStarted(uint256 at)`; `PauseEnded(uint256 at, uint256 length)`; `OperatorApprovalSet(address user, address operator, bool approved)`.

**Errors:** `InvalidTerms`, `WrongState`, `TermsChanged`, `Stalled`, `NotStalled`, `OnlySelf`, `PageRefused`, `ResultRefused`, `NotReleasable`, `NotApprovedOperator`, `WiringMismatch`, `DisbursementMismatch`, `VaultTransferFailed`.

### SpotMarket

One market per pair, over the shared `BatchAuction`; escrow is reserved in the vault. Each bill series has one too. See [spot market](../products/spot-market.md).

| Function | Who may call | Effect |
|---|---|---|
| `placeOrder(Side side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 hintPrev) returns (uint256 orderId)` | Anyone (the caller owns the order) | Reserves the escrow and places the order |
| `placeOrderFor(address owner, Side side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 hintPrev) returns (uint256 orderId)` | Operator of `owner` | The same for `owner` |
| `cancelOrder(uint256 orderId)` | The order's owner | Cancels; releases at once where the market's own record allows |
| `cancelOrderFor(address owner, uint256 orderId)` | Operator of `owner` | The same |
| `clear(uint256 slotId, uint256 maxSteps)` | Anyone | Runs one page of the slot's clearing; earns the keeper reward |
| `expireEmptySlot(uint256 slotId)` | Anyone | Ends a slot with nothing to clear after the maximum duration |
| `abandonClearing(uint256 slotId)` | Anyone | Abandons a stalled clearing, or one in progress while the market is not authorised at the vault |
| `drain(uint256 maxEntries) returns (uint256 processed)` | Anyone | Releases cancelled and expired orders of finished slots |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves the `…For` calls |

`runPage(uint256 slotId, uint256 maxSteps, address keeper)` is callable only by the market itself (`NotSelf`).

**Owner only:** `pause()`, `unpause()`.

**Views:** `getLatestClearedPrice()`, `getClearingAtOrBefore(timestamp)`, `clearedSlotCount()`, `clearedSlotAt(index)`, `getSlot(slotId)`, `getCurrentSlot()`, `currentSlotId()`, `clearingSlot()`, `getCurrentSlotClearableAt()`, `getCurrentSlotMaxExpiresAt()`, `getOrder(orderId)`, `getOrderRecord(orderId)`, `openOrdersByUser(user)`, `activeOrderCount()`, `getBook(side, fromOrderId, limit)`, `bestOrder(side)`, `findHint(side, limitPrice, fromOrderId, maxSteps)`, `getWalk()`, `getDiscoveryCredits(slotId)`, `drainPosition()`, `getSlotCancelQueue`, `getSlotExpiryList`, `quoteForBase(amountBase, limitPrice)`, `orderIdNonce()`, `engine()`, `marketId()`, `baseVault()`, `quoteVault()`, `baseAsset()`, `quoteAsset()`; parameters `STALL_PERIOD`, `MIN_SLOT_DURATION`, `MAX_SLOT_DURATION`, `MIN_ORDER_SIZE_BASE`, `MIN_ORDER_SIZE_QUOTE`, `MIN_CLEAR_VOLUME_BASE`, `KEEPER_REWARD_BPS`, `KEEPER_REWARD_CAP`, `DISCOVERY_STEP_REWARD`, `MAX_OPEN_ORDERS_PER_USER`, `PRICE_PRECISION`, `BPS_DENOMINATOR`, `MAX_DRAIN_AT_CLOSE`, `MAX_HINT_STEPS`.

**Events (from the market):** `ClearingFound(uint256 slotId, uint256 clearingPrice, uint256 matchedBaseVolume, bool lowVolume)`; `ClearingPageRun(uint256 slotId, address keeper, uint256 steps, uint256 credited, uint256 keeperPaid)`; `SlotCleared(uint256 slotId, uint256 clearingPrice, uint256 matchedBaseVolume, uint256 keeperReward, bool lowVolume)`; `ClearingRefused(uint256 slotId, uint8 code)`; `OrderReleased(uint256 orderId, address owner, uint256 amount)`; `OperatorApprovalSet`; pause events. Order and slot events come from `BatchAuction`.

**Errors:** `PageCheckFailed`, `NotSelf`, `ClearingStalled`, `InsufficientGas`, `ConsumerNotAuthorised`, `StillAuthorised`, `InvalidAsset`, `InvalidAmount`, `NotOrderOwner`, `OrderNotFound`, `SlotNotClearing`, `NotApprovedOperator`; engine errors bubble up.

### ClaimLotMarket and ObligorPositionMarket

Ascending auctions of claim units and obligor positions on `SingleAuction`. `ClaimLotMarket` links the library `LotCreditIndex`. See [secondary market](../products/secondary-market.md).

| Function | Contract | Who may call | Effect |
|---|---|---|---|
| `createAuction(uint256 id, uint256 units, address paymentToken, uint256 reservePrice, uint256 duration, bytes32 expectedAsset, uint64 expectedDueAt) returns (uint256 auctionId)` | Claim lot | The seller | Takes the units and opens a sale |
| `createAuctionFor(address user, uint256 id, uint256 units, address paymentToken, uint256 reservePrice, uint256 duration, bytes32 expectedAsset, uint64 expectedDueAt) returns (uint256 auctionId)` | Claim lot | Operator of `user` | The same for `user` |
| `createAuction(uint256 setId, address paymentToken, uint256 reservePrice, uint256 duration, uint256 expectedMaxDebt) returns (uint256 auctionId)` | Position | The seller | Takes the position and opens a sale |
| `createAuctionFor(address user, uint256 setId, address paymentToken, uint256 reservePrice, uint256 duration, uint256 expectedMaxDebt) returns (uint256 auctionId)` | Position | Operator of `user` | The same for `user` |
| `bid(uint256 auctionId, uint256 amount, uint256 expectedItem, address expectedPaymentToken, uint256 maxDebt)` | Both | Anyone but the seller | Escrows the bid (vault or wallet) and bids |
| `bidFor(address user, uint256 auctionId, uint256 amount, uint256 expectedItem, address expectedPaymentToken, uint256 maxDebt)` | Both | Operator of `user` | The same for `user` |
| `settle(uint256 auctionId)` | Both | Anyone | Delivers the item and pays the seller, or returns the item |
| `cancelAuction(uint256 auctionId)`, `cancelAuctionFor(address user, uint256 auctionId)` | Both | The seller, or its operator | Cancels a sale with no bid |
| `claimRefund(address token, address destination)` | Both | The account owed | Pays out wallet-route refunds and proceeds |
| `claimUnits(uint256 id, address destination)` | Claim lot | The recipient | Takes units held for it, with their credit |
| `setOperatorApproval(address operator, bool approved)` | Both | Any account | Approves the `…For` calls |

**Owner only:** `pause()`, `unpause()`.

**Views:** `getAuction(auctionId)`, `effectiveEnd`, `latestEnd`, `currentPrice`, `minimumBid`, `latestClearing(itemId)`, `pendingRefunds(user, token)`, `paymentVault()`, `engine()`, `obligation()`; `ClaimLotMarket`: `lotAccrued(auctionId, key)`, `pendingUnits(user, id)`, `CLAIM_BUFFER`, `INDEX_SCALE`; `ObligorPositionMarket`: `auctionIdByItem(setId)`, `obligorPosition()`; constants `EXTENSION_WINDOW`, `MIN_BID_INCREMENT_BPS`, `MIN_DURATION`, `MAX_DURATION`, `STALL_PERIOD`.

**Events:** `AuctionCreated(uint256 auctionId, address seller, uint256 size, uint256 latestEnd)`; `BidEscrowed(uint256 auctionId, address bidder, uint256 amount, bool vaultRouted)`; `AuctionSold(uint256 auctionId, address winner, uint256 price)`; `AuctionUnsold(uint256 auctionId)`; `AuctionSettledUnderlying(uint256 auctionId)`; `AuctionSettledWithoutDelivery(uint256 auctionId)`; `SaleRefused(uint256 auctionId)`; `RefundAvailable(address user, address token, uint256 auctionId, uint256 amount)`; `RefundClaimed(address user, address token, address destination, uint256 amount)`; `ClaimLotMarket`: `LotCreditPaid(uint256 auctionId, address recipient, bytes32 key, uint256 amount)`, `UnitsHeld(address recipient, uint256 id, uint256 units)`, `UnitsClaimed(address recipient, uint256 id, address destination, uint256 units)`; `OperatorApprovalSet`; pause events.

**Errors:** `InvalidAmount`, `InvalidDuration`, `EndBeyondLatest`, `VaultNotAccepted`, `InvalidItemState`, `ExpectedMismatch`, `DebtAboveMax`, `UnderlyingFinished`, `NotSeller`, `CannotCancelWithBids`, `WiringMismatch`, `AuctionNotFound`, `AuctionClosed`, `AuctionEnded`, `AuctionNotEnded`, `EngineMismatch`, `NoRefundAvailable`, `NotApprovedOperator`, `VaultTransferFailed`; `ClaimLotMarket` also `EntryExists`, `SupplyTooFine`, `UnexpectedDeposit`, `NoUnitsHeld`.

### FutureOrchestrator

A cash-settled future: a maker's posted offer on `PostedOffer`, and two margined contingent legs on the core when taken. See [cash-settled future](../products/cash-settled-future.md).

| Function | Who may call | Effect |
|---|---|---|
| `offer(OfferTerms terms) returns (uint256 offerId)` | Anyone (the maker) | Reserves the margin and posts the offer. `OfferTerms` is `(makerSide, quantity, forwardPrice, capPerUnit, refTime, samples, lookback, expiresAt, taker)` |
| `offerFor(address maker, OfferTerms terms) returns (uint256 offerId)` | Operator of `maker` | The same |
| `cancelOffer(uint256 offerId)`, `cancelOfferFor(address maker, uint256 offerId)` | The maker, or its operator | Releases the maker's reservation |
| `expireOffer(uint256 offerId)` | Anyone, after the offer's end | The same |
| `take(uint256 offerId) returns (uint256 futureId)`, `takeFor(address taker, uint256 offerId) returns (uint256 futureId)` | The taker, or its operator | Locks both margins and creates both legs |
| `settle(uint256 futureId)` | Anyone | Fixes the reference if needed, then fires and resolves each leg |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves the `…For` calls |

**Owner only:** `pause()`, `unpause()`.

**Views:** `getOffer(offerId)`, `offerEnd(offerId)`, `getFuture(futureId)`, `futureOfSet(setId)`, `offerIdNonce()`, `futureIdNonce()`, `bounds()`, `settlementAsset()`, `vault()`, `obligation()`, `lien()`, `resolver()`, `engine()`.

**Events:** `FutureOffered(uint256 offerId, address maker, uint256 engineId, OfferTerms terms, uint256 cap)`; `OfferCancelled(uint256 offerId, uint256 released)`; `OfferExpired(uint256 offerId, uint256 released)`; `FutureOpened(uint256 futureId, uint256 offerId, address long, address short, uint256 longSetId, uint256 shortSetId, uint256 longObligationId, uint256 shortObligationId, bytes32 referenceKey)`; `Settled(uint256 futureId)`; `OperatorApprovalSet`; pause events.

**Errors:** `InvalidTerms`, `InvalidExpiry`, `InvalidBounds`, `OfferNotOpen`, `OfferNotExpired`, `NotMaker`, `NotAuthorised`, `ReservationMismatch`, `FutureNotFound`, `WiringMismatch`, `EngineMismatch`, `NotApprovedOperator`.

### SpotReferenceResolver

Fixes a future's reference price, the median of recorded `SpotMarket` clearings, and answers Obligation as the legs' resolver module. No owner, no pause. See [cash-settled future](../products/cash-settled-future.md).

| Function | Who may call | Effect |
|---|---|---|
| `fix(uint256 refTime, uint256 samples, uint256 lookback) returns (uint256 price, bool isVoid)` | Anyone, after `refTime` | Stores the reference once |
| `conditionMet(uint256, bytes triggerData)`, `amountOf(uint256, bytes resolverData)`, `targetsOf(bytes)` (views); `onCreate(uint256, bytes)` (does nothing) | Read and called by Obligation as the legs' module | Trigger, amount and disclosure |

**Views:** `referenceOf(refTime, samples, lookback)`, `referenceKey(refTime, samples, lookback)`, `spotAuction()`; constants `MAX_SAMPLES`, `MIN_LOOKBACK`, `MAX_LOOKBACK`. **Events:** `ReferenceFixed(bytes32 key, uint256 refTime, uint256 samples, uint256 lookback, uint256 price, bool isVoid, uint256[] slotIds)`. **Errors:** `ReferenceNotYet`, `InvalidReference`, `AlreadyFixed`.

### CoverOrchestrator

Insurance: underwriters' offers on `PostedOffer`, and policies whose cover is a margined contingent obligation on the core. Links the libraries `CoverIssuance` and `CoverSettlement`. See [insurance and bills](../products/insurance-and-bills.md) and [liquidation cover](../products/liquidation-cover.md).

| Function | Who may call | Effect |
|---|---|---|
| `createOffer(OfferTerms terms) returns (uint256 offerId)` | Anyone (the underwriter) | Reserves `capacity` and opens the offer. `OfferTerms` is `(asset, capacity, premiumBps, coverGrace, payDelay, adapter, collateralKey, minCollateralWad, maxLoanGrace, dueFrom, dueTo, expiry)` |
| `createOfferFor(address underwriter, OfferTerms terms) returns (uint256 offerId)` | Operator of `underwriter` | The same |
| `cancelOffer(uint256 offerId)`, `cancelOfferFor(address underwriter, uint256 offerId)` | The underwriter, or its operator | Returns the unused reservation |
| `closeExpired(uint256 offerId)` | Anyone, after the offer's end | The same |
| `insure(uint256 offerId, uint256 loanId, uint256 units, address beneficiary, Expected e) returns (uint256 policyId)` | The claim holder (the insured) | Loan cover: takes the units and premium, writes the cover. `Expected` is `(asset, premiumBps, adapter, collateralKey, minCollateralWad, coverGrace, payDelay, maxPremium)` |
| `insureFor(address insured, uint256 offerId, uint256 loanId, uint256 units, Expected e) returns (uint256 policyId)` | Operator of `insured` | The same, `insured` as beneficiary |
| `insureVia(uint256 offerId, uint256 ref, uint256 cover, uint64 windowEnd, address beneficiary, Expected e) returns (uint256 policyId)` | The insured | Cover on an adapter position (`ref`) |
| `insureViaFor(address insured, uint256 offerId, uint256 ref, uint256 cover, uint64 windowEnd, Expected e) returns (uint256 policyId)` | Operator of `insured` | The same |
| `settle(uint256 policyId)`, `settleMany(uint256[] policyIds)` | Anyone | Drives a loan-cover policy as far as states allow |
| `claimUnits(uint256 policyId, address destination)`, `claimUnitsFor(address account, uint256 policyId)` | The account units are held for, or its operator | Takes units whose push was refused |
| `transferBeneficiary(uint256 policyId, address to)`, `transferBeneficiaryFor(address beneficiary, uint256 policyId, address to)` | The beneficiary, or its operator | Moves the beneficiary while `ACTIVE` or `SUBROGATED` |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves the `…For` calls |

**Owner only:** `pause()`, `unpause()`.

**Views:** `getOffer(offerId)`, `offerEnd(offerId)`, `quote(offerId, loanId, units)`, `getPolicy(policyId)`, `policyOfLoan(loanId)`, `deliveredOf(policyId, key)`, `vault()`, `core()`, `coreLien()`, `engine()`; constants `COVER_TAIL`, `ADAPTER_GAS`, `UNIT_PUSH_GAS`, `PUSH_RESERVE_GAS`, `MAX_PAY_DELAY`.

**Events:** `OfferCreated(uint256 offerId, address underwriter, OfferTerms terms)`; `OfferClosed(uint256 offerId, uint256 remaining)`; `PolicyCreated(uint256 policyId, uint256 offerId, Kind kind, address insured, address beneficiary, uint256 coverSetId, uint256 coverId, uint256 loanIdOrRef, uint256 units, uint256 cover, uint256 premium)`; `PolicySettled(uint256 policyId, Phase phase)`; `Delivered(uint256 policyId, address to, bytes32 key, uint256 amount)`; `UnitsHeld(uint256 policyId, address account, uint256 units)`; `UnitsDelivered(uint256 policyId, address account, uint256 units)`; `BeneficiaryTransferred(uint256 policyId, address from, address to)`; `OperatorApprovalSet`; pause events.

**Errors:** `TermsMismatch`, `OfferNotOpen`, `OfferExpired`, `OfferNotExpired`, `InvalidOffer`, `InsufficientCapacity`, `PremiumAboveMax`, `LoanNotInsurable`, `CollateralBelowFloor`, `DueOutOfRange`, `ZeroCover`, `AdapterRefused`, `NoInterest`, `CoverAboveCap`, `InvalidWindow`, `NotGavel`, `NothingToSettle`, `NotHeldFor`, `NotBeneficiary`, `NotUnderwriter`, `UnexpectedUnits`, `InsufficientGas`, `EngineMismatch`, `NotApprovedOperator`.

### ContingentBondAdapter

Reads Obligation to answer `BondProtocol` about a cover, so an underwriter can stake BOND on paying it. View-only; no owner, no pause. See [insurance and bills](../products/insurance-and-bills.md).

**Views:** `obligationTerms(uint256 id) returns (address asset, uint256 amountOwed, uint64 settlesAt, address obligor)`, `obligationState(uint256 id) returns (State state, uint256 shortfall)`, `core()`, `breachIsShortfall()`. **Errors:** `InvalidAddress`.

### BillPackager

Packages insured loan claims due in the same week into one ERC-20 per series. No owner, no pause. See [insurance and bills](../products/insurance-and-bills.md).

| Function | Who may call | Effect |
|---|---|---|
| `deposit(uint256 offerId, uint256 loanId, uint256 units, address recipient, Expected e) returns (uint256 policyId)` | The lender (claim holder) | Insures the claim and mints `cover` bills to `recipient` |
| `depositFor(address lender, uint256 offerId, uint256 loanId, uint256 units, Expected e) returns (uint256 policyId)` | Operator of `lender` | The same; bills to `lender` |
| `openSeries(uint256 k) returns (address token)` | Anyone | Creates series `k`'s `BillToken` (also done at its first deposit) |
| `finalise(uint256 policyId)`, `finaliseMany(uint256[] policyIds)` | Anyone | Settles a policy and books what arrived |
| `redeem(uint256 k, uint256 amount, address destination)` | A bill holder | From the series date, once every policy is final: burns bills for a share of the series |
| `redeemFor(address holder, uint256 k, uint256 amount)` | Operator of `holder` | The same, paid to `holder` |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves the `…For` calls |

**Views:** `dateOf(k)`, `seriesOf(due)`, `tokenOf(k)`, `predictToken(k)`, `seriesView(k)`, `recordOf(policyId)`, `bookedOf(policyId, key)`, `orchestrator()`, `core()`, `vault()`, `settlementKey()`, `collateralKey()`; constants `BILL_SETTLE_BOUND`, `BILL_GRACE_BOUND`, `MIN_BILL_COVER`, `SERIES_EPOCH`, `SERIES_SPACING`.

**Events:** `SeriesOpened(uint256 k, address token)`; `Deposited(uint256 policyId, uint256 k, address lender, address recipient, uint256 cover, uint256 premium)`; `Booked(uint256 policyId, bytes32 key, uint256 amount)`; `PolicyFinalised(uint256 policyId, uint256 k)`; `Redeemed(uint256 k, address holder, address destination, uint256 amount)`; `OperatorApprovalSet`.

**Errors:** `InvalidSeries`, `SeriesNotFinal`, `NotRecorded`, `OfferNotBillable`, `OfferEnded`, `CoverTooSmall`, `PremiumAboveMax`, `UnexpectedUnits`, `NotApprovedOperator`.

### BillToken

One per bill series: a plain ERC-20 (`GBILLk`) with the settlement token's decimals. No owner, no pause. See [insurance and bills](../products/insurance-and-bills.md).

Standard ERC-20 (`transfer`, `transferFrom`, `approve`, `balanceOf`, `allowance`, `totalSupply`, `decimals`). `mint(address to, uint256 amount)` and `burn(address from, uint256 amount)`: `BillPackager` only. **Views:** `packager()`, `series()`. **Events:** ERC-20 `Transfer`, `Approval`. **Errors:** `NotPackager`.

### MorphoCoverAdapter

One per lending market and ceiling (three on the testnet): opens cover accounts, keeps their state, and is the module `CoverOrchestrator` and Obligation consult for liquidation cover. No owner, no pause, no setter; never calls the vault. See [liquidation cover](../products/liquidation-cover.md).

| Function | Who may call | Effect |
|---|---|---|
| `openAccount() returns (uint256 accountId)` | Anyone (becomes owner) | Clones a `CoverAccount` |
| `supplyCollateral(uint256 accountId, uint256 assets)` | The account's owner | Supplies collateral at the market |
| `borrow(uint256 accountId, uint256 assets, address receiver)` | The owner, not frozen | Borrows at the market |
| `repay(uint256 accountId, uint256 assets)`, `repayAll(uint256 accountId)` | Anyone | Repays the account's debt from the caller's tokens |
| `withdrawCollateral(uint256 accountId, uint256 assets, address receiver)` | The owner; while frozen only with no debt | Withdraws collateral |
| `arm(uint256 accountId) returns (uint256 ref)` | The owner, within the ceiling | Freezes the account and records `tracked` |
| `disarm(uint256 ref)` | The owner | Ends an armed `ref` with no cover |
| `record(uint256 ref)` | Anyone | Records the first fall in collateral (a liquidation) |
| `sweep(uint256 accountId, address token, address to)` | The owner | Moves a stray token out of the account |
| `onCreate(uint256 coverId, bytes data)` | Obligation, for a `CoverOrchestrator` cover | Binds the cover to its `ref` |

**Views:** `coverTerms(ref, insured)`, `breachOf(ref)`, `creatorOf(ref)`, `frozen(accountId)`, `accountOf(accountId)`, `refOf(ref)`, `healthOf(accountId)`, `marketParams()`, `amountOf`, `conditionMet`, `targetsOf`; immutables `MORPHO`, `MARKET_ID`, `LOAN_TOKEN`, `COLLATERAL_TOKEN`, `ORACLE`, `IRM`, `LLTV`, `LIF`, `CONTINGENT`, `ORCHESTRATOR`, `VAULT`, `ACCOUNT_IMPLEMENTATION`, `MAX_HEALTH_USE`.

**Events:** `AccountOpened(uint256 accountId, address account, address owner)`; `Armed(uint256 ref, uint256 accountId, uint256 tracked)`; `Disarmed(uint256 ref)`; `Bound(uint256 ref, uint256 coverId, uint256 cover)`; `LiquidationRecorded(uint256 ref, uint256 tracked, uint256 collateral)`; `CollateralSynced(uint256 ref, uint256 tracked)`; `Swept(uint256 accountId, address token, address to, uint256 amount)`; `Supplied(uint256 accountId, uint256 assets)`; `Borrowed(uint256 accountId, uint256 assets, address receiver)`; `Repaid(uint256 accountId, address payer, uint256 assets)`; `Withdrawn(uint256 accountId, uint256 assets, address receiver)`.

**Errors:** `NotOwner`, `Frozen`, `NotArmed`, `NotLive`, `CeilingExceeded`, `LiquidatedBefore`, `NotContingent`, `NotOrchestrator`, `BadTerms`, `NotOwnerUnits`, `InvalidAddress`, `ZeroAmount`, `UnknownRef`, `UnknownMarket`.

### CoverAccount

The implementation each cover account is cloned from (EIP-1167): one borrower's position at the lending market. It does only what its adapter commands. No owner. See [liquidation cover](../products/liquidation-cover.md).

`initialise(address adapter_)`: called once by the adapter when it clones the account. `act(Op op, address target, MarketParams p, uint256 a, uint256 b, address to) returns (uint256 amount)`: the adapter only. **Errors:** `NotAdapter`, `AlreadyInitialised`.

---

## BOND

### BondToken

The BOND ERC-20 (18 decimals). Minted only by `BondProtocol`, apart from the fixed genesis amount at deployment. No owner, no pause. See [BOND](../products/bond.md).

Standard ERC-20. `burn(uint256 amount)`: any holder, its own balance. `mint(address to, uint256 amount)`: `BondProtocol` only. **Views:** `minter()`. **Events:** ERC-20 `Transfer`, `Approval`. **Errors:** `NotMinter`, `InvalidMinter`.

### BondProtocol

Registrations, stakes, commitments and minting. Not a vault consumer. No owner, no setter, no pause. See [BOND](../products/bond.md).

| Function | Who may call | Effect |
|---|---|---|
| `registerSet(uint256 setId, uint256 stake, uint256 ref)` | The holder of the set's obligor position | Registers the set, pulling `stake` BOND |
| `commit(IBondSource source, uint256 ref, uint256 amount)` | The borrower the source names, or its operator | Adds BOND to a commitment on a pending origination |
| `bind(IBondSource source, uint256 ref)` | The borrower or operator for 1 day after origination, then anyone | Binds the commitment as the set's stake |
| `reclaimCommitment(IBondSource source, uint256 ref)` | Anyone | Returns a commitment to the borrower once the origination ended or a bind was skipped |
| `observe(IBondSource source, uint256 ref)` | Anyone, within 30 days of origination | Records the loan's rate on the curve |
| `resolveObligation(uint256 obligationId)` | Anyone | Mints, or marks why not, for a finished obligation |
| `resolveSet(uint256 setId)` | Anyone | Resolves the set's obligations, then returns or burns the stake |
| `registerExternal(IObligationAdapter adapter, uint256 externalId, uint256 stake, uint64 longStop)` | The obligor the adapter names | Stakes on a debt read through an adapter; never mints |
| `resolveExternal(IObligationAdapter adapter, uint256 externalId)` | Anyone | Burns or returns an external stake |
| `setOperatorApproval(address operator, bool approved)` | Any account | Approves `commit` and early `bind` |

**Views:** `getRegistration(setId)`, `commitment(source, ref)`, `getExternal(adapter, externalId)`, `estimateMint(obligationId)`, `curveRate(loanToken, term)`, `minted(loanToken)`, `isSource(source)`, `isOperator(user, operator)`, `listing()`, `curve()`, `bondToken()`, `obligation()`, `lien()`, `vault()`, `MAX_CURVE_RATE`; constants `OBSERVE_WINDOW`, `BIND_GRACE`, `MAX_EXTERNAL_TERM`, `ADAPTER_GAS`, `YEAR`, `RATE_PRECISION`.

**Events:** `SetRegistered(uint256 setId, address registrant, uint256 stake, bool secured, bool minting, uint64 termStart)`; `SetValued(uint256 setId, uint256 principalUsd, uint256 lockUsd)`; `StakeCommitted(address source, uint256 ref, address borrower, uint256 amount, uint256 total)`; `CommitmentBound(address source, uint256 ref, uint256 setId, uint256 amount)`; `BindSkipped(address source, uint256 ref, uint256 setId, BindSkipReason reason)`; `CommitmentReclaimed(address source, uint256 ref, address borrower, uint256 amount)`; `CurveObserved(address source, uint256 setId, address loanToken, uint256 bucket, uint256 rate, uint256 lenderCount)`; `MintCapped(uint256 setId, uint256 obligationId, uint256 uncapped, uint256 minted)`; `BondMinted(uint256 setId, uint256 obligationId, address registrant, uint256 amount)`; `MintSkipped(uint256 setId, uint256 obligationId, MintSkipReason reason)`; `MintingLapsed(uint256 setId, address holder)`; `BondStakeReturned(uint256 setId, address registrant, uint256 amount)`; `BondStakeBurned(uint256 setId, uint256 amount)`; `ExternalRegistered(address adapter, uint256 externalId, address registrant, uint256 stake, uint64 longStop)`; `ExternalStakeBurned(address adapter, uint256 externalId, uint256 amount)`; `ExternalStakeReturned(address adapter, uint256 externalId, address registrant, uint256 amount, ExternalReturnReason reason)`; `OperatorApprovalSet`.

**Errors:** `NotHolder`, `SetNotActive`, `SetStillActive`, `AlreadyRegistered`, `NotRegistered`, `RegistrationNotActive`, `IneligibleAsset`, `ReleaseGateSet`, `ContingentSet`, `WrongRef`, `UseBind`, `UnknownSource`, `NotPending`, `NotOriginated`, `NotBorrower`, `NothingCommitted`, `BindReserved`, `NotReclaimable`, `AlreadyObserved`, `ObservationClosed`, `NotQualified`, `AlreadyResolved`, `NotFinished`, `InvalidAmount`, `InvalidAdapter`, `NotObligor`, `NotOpen`, `InvalidLongStop`, `NotFinal`, `InsufficientGas`, `InvalidConfig`, `LoanTokenUnpriced`, `NotApprovedOperator`.

### BondListing

Which loan tokens, priced assets and sources count for new registrations, and what collateral is worth. No pause. See [BOND](../products/bond.md).

No function for users: every write is the owner's.

**Owner only** (until `LISTING_END` or renouncement; then `ListingEnded`): `queueLoanToken(address token, bool priced, uint256 mintCap, uint256 minObservedPrincipal)`, `queuePriced(address token, address spot, uint256 maxPrice)`, `queueSource(address source)`, `queueMarket(address token, address spot)`, `queueMintCap(address token, uint256 newCap)` (each live 7 days after queueing); `cancel(Kind kind, address key)`; `remove(Kind kind, address key)` (immediate, final); `lowerMaxPrice(address token, uint256 newMax)`, `lowerMintCap(address token, uint256 newCap)`; `renounceOwnership()` (ends the listing power at once).

**Views:** `loanToken(token)`, `mintCap(token)`, `minObservedPrincipal(token)`, `pricedOf(token)`, `assetPrice(token)`, `value(lockId, loanToken_, principal)`, `sourceOfOriginator(originator)`, `originatorOf(source)`, `loanTokens(i)`, `pricedTokens(i)`, `sources(i)`, `loanTokenCount()`, `pricedTokenCount()`, `sourceCount()`, `pendingMarket(token)`, `pendingMintCap(token)`, `LISTING_END()`, `endedAt()`, `vault()`, `owner()`; constants `LISTING_DELAY`, `MAX_LISTING_PERIOD`, `MAX_ENTRIES`, `PRICE_SAMPLES`, `PRICE_WINDOW`, `PRICE_PRECISION`, `PRICE_GAS`.

**Events:** `LoanTokenQueued(address token, bool priced, uint256 mintCap, uint256 minObservedPrincipal, uint64 effectiveAt)`; `PricedQueued(address token, address spot, address quote, uint256 maxPrice, uint64 effectiveAt)`; `SourceQueued(address source, address originator, uint64 effectiveAt)`; `MarketQueued(address token, address spot, address quote, uint64 effectiveAt)`; `MintCapQueued(address token, uint256 mintCap, uint64 effectiveAt)`; `ListingCancelled(Kind kind, address key)`; `ListingRemoved(Kind kind, address key)`; `MaxPriceLowered(address token, uint256 maxPrice)`; `MintCapLowered(address token, uint256 mintCap)`; `ListingPowerEnds(uint64 endsAt)`; `ListingPowerRenounced(uint64 at)`.

**Errors:** `ListingEnded`, `PastEnd`, `InvalidListing`, `AlreadyListed`, `NotQueued`, `NotLive`, `NotLower`, `TooMany`, `InUse`, `InsufficientGas`.

### BondCurve

One rate curve per loan token, in 34 term buckets. Its one writer, `BondProtocol`, is fixed at deployment. No owner, no pause. See [BOND](../products/bond.md).

`record(address loanToken, uint256 bucket, uint64 time, uint64 term, uint256 rate, address[] lenders) returns (uint256 lenderCount)`: `BondProtocol` only. **Views:** `curveRate(loanToken, term)`, `curveBucket(loanToken, bucket)`, `bucketOf(term)`, `writer()`; constants `CURVE_RING`, `CURVE_WINDOW`, `CURVE_MIN_OBS`, `CURVE_MIN_LENDERS`, `NUM_BUCKETS`. **Errors:** `NotWriter`, `InvalidAddress`.

---

## Linked libraries

Each is deployed and verified separately, is linked into the contract named, has no state of its own and is not called directly.

| Library | Linked into |
|---|---|
| `ObligationLogic`, `ModuleLogic` | `Obligation` (above) |
| `BulletOpening` | `BulletLoanRequest`, `BulletLoanOffer`: opens the engine auctions |
| `BulletSettlement` | `BulletLoanRequest`, `BulletLoanOffer`: creates the set and lien at settlement |
| `CoverIssuance` | `CoverOrchestrator`: opens offers and writes covers |
| `CoverSettlement` | `CoverOrchestrator`: settles policies |
| `LotCreditIndex` | `ClaimLotMarket`: tracks the credit a listed lot earns |

## Test and superseded contracts

**Test contracts.** The testnet's mock tokens (`mWBTC`, `mUSDC`, `mUSDT`, `mWETH`, `mcbBTC` and a few unused ones), the mock lending market (`MockMorpho`, `MockOracle`, `MockIrm`) and `MockCoverAdapter` are test contracts, not part of the protocol. See [deployments](../deployments.md).

**Superseded.** A first `Obligation`, `Lien`, `ObligorPosition` and `AuctionOrchestrator`, and a first BOND package, are superseded; they are marked in the address list.

## Next

- [Writing](writing.md): your first transactions.
- [Reading data](reading-data.md): views, events and the API.
- [Composing a product](composing-a-product.md): building on the core and the engines.
- [The base layer and what you trust](../concepts/base-layer-and-trust.md): what the owners can and cannot do.
