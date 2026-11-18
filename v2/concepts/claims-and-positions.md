# Claims and obligor positions

Every set of obligations creates two kinds of token: one for the creditors' side, one for the debtor's. Both are ordinary, transferable tokens, so either side of a debt can be sold.

| Token | Standard | One per | Whoever holds it |
|---|---|---|---|
| **Claim units** | ERC-1155, a fixed supply per id | Obligation (the id is the obligation id) | Is paid: a pro-rata share of everything credited to that obligation |
| **Obligor position** | ERC-721 | Set (the token id is the set id) | Owes the set, may pay it, and receives any collateral returned from its lien |

A bullet loan has one claim id, whose 1,000,000 units the lender holds, and one obligor position, which the borrower holds.

## Claim units

The claim token is `Obligation` itself: claim units are ERC-1155 tokens of the Obligation contract.

- **Supply** is minted once, when the obligation is created, to the holders named then, and never increased. Units are burned when a holder redeems a final obligation or releases units to forgive the debt.
- **Units of one id are interchangeable.** Different ids never are, even on the same date and asset, since obligors and collateral differ.
- **Transfers** are standard `safeTransferFrom` and `safeBatchTransferFrom`, with the receiver hook the standard requires. Approval is by `setApprovalForAll`, which covers every claim id the holder has, present and future.
- **Credit follows the units.** Before units move, what the sender and receiver have each earned so far is recorded, so credit earned before a transfer stays with the seller.
- **Whole-only claims.** If a set's collateral includes an ERC-721, the set holds one obligation whose units one account holds, and those units move only as a whole holding (`NotWholeTransfer`). Products on the testnet take ERC-20 collateral only.
- **Units cannot be sent to the set's vault** (`ClaimsNotDepositable`): claims are not vault collateral.
- **Transfers work while Obligation is paused.**
- **Contingent obligations share the token.** Their ids sit beside plain ones; `contingentStateOf(id)` tells them apart, and until resolution such an id's amounts read at their maximum.

`uri(id)` returns a fixed URI template, set when Obligation was deployed; the metadata behind it is rendered off-chain from chain state.

## The obligor position

`ObligorPosition` is a separate ERC-721 contract. Only Obligation mints and burns it, and that link was set once at deployment and cannot be changed.

- **Minted** to the obligor when the set is created; **burned** when the set is finished: by the Lien at the end of a secured set (at the claim, if anything is to be claimed), or by `close` for an unsecured one.
- **Transfers** are standard ERC-721: `transferFrom`, `safeTransferFrom`, `approve`, `setApprovalForAll`. Nothing moves a position without its holder's approval, and no one can pause its transfers.
- **What a transfer carries:** the debt, and for a secured set the equity in its collateral, including any credit the Lien holds for the position. The new holder is the one who can pay, pay off and `claim`.
- **A position cannot be sent to its set's vault** (`PositionNotDepositable`).
- **Its state** is read with `obligation.positionState(setId)`:

| State | The holder has |
|---|---|
| `ACTIVE` | The debt, the right to pay, and the equity in any collateral |
| `IN_ENFORCEMENT` | A secured set in breach: the right to pay off through the Lien until it is enforced, then only a claim to whatever the Lien credits to the position |
| `BREACHED_UNSECURED` | The debt, and the right to pay late |
| `CREDIT_PENDING` | The set has ended; whatever the Lien holds for the position, waiting to be claimed |
| `FINISHED` | Nothing: the position is burned |

## Creation cannot be refused

Claim units and the obligor position are minted with **no receiver hook**: no code of a holder or obligor runs while a set is created, so a contract named as either cannot block a product's settlement by refusing what it is given. Transfers keep their hooks.

This departs from ERC-1155, which asks that a mint to a contract call `onERC1155Received`. (ERC-721 allows a mint without the check.) Wallets and indexers see the usual `TransferSingle` and `Transfer` events. A contract that bids or borrows should be able to move ERC-1155 and ERC-721 tokens, and to call `collect`, `redeem` and `claim`; one that cannot keeps what it is given and may strand its credit.

Because no recipient can refuse, anyone can create a set naming any address as a holder or as the obligor. Each creation makes new ids, so no one can add units to an id you already hold. A contract that counts the ids it holds, rather than the ones it asked for, should allow for ones it did not ask for.

## Selling either side

Both tokens trade on the [secondary market](../products/secondary-market.md): claim units on `ClaimLotMarket`, positions on `ObligorPositionMarket`. Either can also be transferred directly to anyone, by the holder's own transaction.

## Next

- [The engines](engines.md): the auctions that set the terms these tokens record.
