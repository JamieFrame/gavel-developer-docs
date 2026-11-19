# The vault

The vault holds custody and nothing else. It keeps a ledger of each account's balances, lets approved contracts reserve those balances or lock them as collateral, and gives every user a way out. It knows nothing of debts, liens, prices or auctions. It is not upgradeable and has no pause.

On the testnet there is one shared `Vault`, used by the core and every product. Its address is on the [deployments](../deployments.md) page.

## Accounts and assets

Every user has an account in the vault, holding a balance of each asset in three parts:

| Part | Meaning |
|---|---|
| **Free** | Yours to use, pledge or withdraw |
| **Reserved** | Set aside by a product for a commitment you made, such as a bid. Each reservation belongs to one product and one context, and only that product can release or settle it |
| **Locked** | Held in a *lock*: a basket of up to 16 assets backing a debt. A lock has an owner (at first the product that made it, then the Lien) and a depositor (you) |

Your **total** balance is free + reserved + locked. Reserving or locking moves value between the parts of your own account; it never leaves the vault.

**Assets are identified by a 32-byte key** whose first byte is its kind:

| Kind | Key |
|---|---|
| ERC-20 | `0x01`, eleven zero bytes, then the token address |
| ERC-721 | `0x02`, then the low 31 bytes of `keccak256(abi.encode(collection, tokenId))` |

The kind is the caller's assertion: the vault does not ask the token what it is. Products on the testnet use ERC-20 keys only.

## Getting in and out

| Function | Who may call | What it does |
|---|---|---|
| `deposit(user, asset, amount, params)` | `user` only | Pulls the token from your wallet into your free balance. You first approve the vault on the token, as for any ERC-20 |
| `withdraw(user, asset, amount, destination)` | `user` only | Sends free balance to any destination other than zero or the vault itself |
| `depositFor`, `withdrawFor` | `user` or an operator `user` approved | The same, for a relayer or agent. `depositFor` still draws only on `user`'s wallet; `withdrawFor` pays only `user` |

`params` is empty for an ERC-20. For an ERC-721 it encodes the collection and token id.

Every inbound movement is measured: the vault compares its token balance before and after, and refuses a deposit that did not arrive in full (`UnexpectedDepositAmount`). Fee-on-transfer tokens therefore cannot be deposited. Rebasing tokens are not supported and cannot be detected. A token sent to the vault directly, outside `deposit`, is credited to no one and cannot be recovered.

**Withdrawal never depends on anyone else.** No pause, no owner action and no product can stop you withdrawing your free balance; only the token itself can (for example, an issuer that freezes an address).

## Consumers and your approval

A **consumer** is a contract the vault lets reserve, lock and move balances: a product, Obligation or the Lien. Two conditions must both hold before a consumer can act on your account:

1. **The vault's owner has admitted it** to the consumer whitelist (`setAuthorisedConsumer`).
2. **You have approved it** for your own account (`approveConsumer(consumer, true)`).

An admitted consumer can touch only the accounts of users who approved it, and no consumer can touch another consumer's reservations. Approval is per consumer: approving the bullet loan does not approve the spot market.

To see where a contract stands:

| View | Answers |
|---|---|
| `isAuthorisedConsumer(consumer)` | Is it on the whitelist now? |
| `isUserApproved(user, consumer)` | Has this user approved it? |
| `authorisedConsumerCount()`, `authorisedConsumerAt(i)` | The whole whitelist, one entry at a time |
| `wasEverAuthorised(account)` | Was it ever on the whitelist? (see *Exits*, below) |

On the testnet the whitelist holds the core contracts and each product contract that reserves or locks users' balances. Contracts that never reserve, lock or move users' balances (the engines, the resolvers, the libraries, BOND, liquidation cover) are not on it. The owner today is the account that deployed the testnet contracts; `owner()` returns it.

Operators are a separate approval. `setOperatorApproval(operator, true)` lets an operator call `depositFor` and `withdrawFor` for you, nothing else, and an operator cannot approve consumers or other operators. See [writing](../build/writing.md).

## Reservations

A product reserves part of your free balance when you make a commitment it must be able to honour, such as a bid. A reservation is keyed by the product, you, the asset and a context id the product chooses, so its records match the product's own.

| Function | Who may call | What it does |
|---|---|---|
| `reserve(user, asset, amount, contextId)` | An admitted consumer `user` approved | Free → reserved |
| `releaseReservation(user, asset, contextId)` | The consumer that holds it | The whole reservation back to free. Returns the amount; zero, without error, if there was none |
| `reduceReservation(user, asset, contextId, newAmount)` | The consumer that holds it | Returns part of it to free |
| `settleReservation(from, to, asset, contextId, amount)` | The consumer that holds it, while admitted | Pays some or all of it to `to`, completing the commitment |

## Locks

A lock is a basket of collateral, held whole until it is released. A product makes one with `lockCollateral`, drawing first on your free balance and pulling any shortfall of an ERC-20 from your wallet. The product owns the lock at first and hands it to the Lien when the loan is made.

| Function | Who may call | What it does |
|---|---|---|
| `lockCollateral(params)` | An admitted consumer the depositor approved | Creates a lock of 1 to 16 legs, in strictly ascending key order, all or none |
| `releaseCollateral(lockId, recipient)` | The lock's current owner | Credits every leg to `recipient`'s free balance in one ledger move. No token leaves the vault |
| `reduceLock(lockId, newAmounts)` | The lock's current owner | Returns part of each leg to the depositor; never drops a leg |
| `transferLoanOwnership(lockId, newOwner)` | The lock's current owner | Hands the lock to another admitted contract, recording who handed it over |
| `setAcceptsEnlargement(bool)` | A lock owner, for every lock it owns | Opts in to enlargement |
| `enlargeLock(lockId, increases)` | Anyone, from their own free balance, if the owner opted in | Adds to existing legs, never a new leg. A contribution becomes the depositor's and gives the contributor no claim |
| `enlargeLockFor(contributor, lockId, increases)` | An admitted consumer the contributor approved | The same, drawing on the contributor's balance |

Some names (`transferLoanOwnership`, `getLoanOwner`, `LoanNotFound`) speak of loans; a lock may secure any set of obligations.

Read a lock with `getLoanRecord`, `getLoanLegs`, `getDepositor`, `getLoanOwner`, `getPreviousOwner` and `isCollateralReady`. A lock's record stays readable after release.

## Exits: what keeps working

The vault's one emergency lever is its whitelist, and it is built so that **exits survive and entries stop**.

| If the owner de-authorises a consumer, or empties the whitelist with `emergencyDeauthoriseAll`, or you revoke your approval | |
|---|---|
| `deposit`, `withdraw` and their `…For` forms | Work: never consumer-gated |
| Releasing or reducing a reservation | Works: it can only return value to you |
| Releasing or reducing a lock | Works: only the lock's owner is checked |
| A consumer moving its own balance | Works for any contract that was ever admitted, so a lien's proceeds and a product's refunds still pay out |
| `settleReservation`, `transferLoanOwnership` | Work after **your** revocation, completing what you committed to; refused after **the owner's** de-authorisation, so nothing new originates and the product's own exits return the escrow |
| `reserve`, `lockCollateral`, `enlargeLockFor`, moving another account's balance | Refused |

No owner function changes a balance, a reservation or a lock. De-authorisation moves nothing.

## What the vault guarantees

- **Conservation.** The ledger records an asset only as it measures it entering. For each asset, the sum of every account's total equals a ledger total (`ledgerTotal(asset)`), and the vault holds at least that much of the token, unless the token itself has moved the vault's balance. Anyone can compare `ledgerTotal` with the token's `balanceOf(vault)`.
- **No overflow after entry.** Every balance, reservation and leg is bounded by its asset's ledger total, so only a deposit can fail on arithmetic, and only for that asset. A misbehaving token can stop its own new deposits; it cannot freeze a release, a payoff or an enforcement.
- **Locks are whole.** A lock is created whole or not at all, released whole to one recipient, released once, and its set of assets never changes.
- **No credit before its tokens.** During any call, a balance never includes tokens that have not yet arrived.

**On the testnet**, the mock tokens let anyone burn any holder's balance, the vault's included, so the vault can come to hold less than it records. Real tokens do not behave this way. Tokens whose issuer can move or freeze balances (some stablecoins and tokenised securities) can do the same to the vault on a live network, and whoever withdraws last bears the difference.

## Events and errors

Events: `Deposited`, `WithdrawalInitiated`; `Reserved`, `ReservationReleased`, `ReservationReduced`, `ReservationSettled`; `Transferred`; `CollateralLocked` and `CollateralReleased` (one per leg), `BasketLocked`, `CollateralReduced`, `CollateralEnlarged`, `LoanOwnershipTransferred`, `EnlargementAcceptanceSet`; `ConsumerAuthorisationSet`, `ConsumerApprovalSet`, `OperatorApprovalSet`. For deposits and locks, the token's `Transfer` is logged before the vault's own events; a withdrawal logs `WithdrawalInitiated` first.

The full error list is in the [contracts reference](../build/contracts-reference.md). The ones a user meets most often: `ConsumerNotAuthorised` (the contract is not on the whitelist), `UserHasNotApproved` (you have not approved it), `InsufficientBalance`, `UnexpectedDepositAmount` and `LedgerTotalExceeded(asset)`.

## Next

- [Obligation](obligation.md): the debts a lock can secure.
- [The Lien](lien.md): how a lock is bound to a debt.
