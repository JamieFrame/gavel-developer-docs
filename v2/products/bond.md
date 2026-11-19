# BOND

BOND is a credit record. It is an ERC-20 minted to whoever registered a secured loan, as the loan's interest is paid on time, and it can be staked on obligations, where it is burned if the obligation is not paid in time. It reads the record of loans rather than writing to it: no BOND contract creates, pays or enforces an obligation. Four contracts make it up: `BondToken` (the token), `BondProtocol` (registrations, stakes, commitments and minting), `BondListing` (which loan tokens, priced assets and sources count, and what collateral is worth) and `BondCurve` (a rate curve for each loan token). Addresses are on the [deployments](../deployments.md) page. A first BOND package is also on chain, superseded.

BOND carries **no governance, fee or revenue right**. There is no vote, no treasury and no share of anything.

## The token

`BondToken` is a plain ERC-20 with 18 decimals, named and symbolled "BOND". Its only minter is `BondProtocol`, fixed in the constructor (`minter()`), and its only other issue was a **genesis amount of 1,000,000 BOND, minted once at deployment to the deploying account**. It cannot recur. Any holder can `burn` its own balance; there is no `burnFrom`, no owner, no pause, no permit and no hooks.

## Registering a secured loan

| Function | Who may call |
|---|---|
| `registerSet(setId, stake, ref)` | The holder of the set's obligor position (`NotHolder` otherwise) |

The caller becomes the **registrant**, who receives any BOND minted and the stake back. The stake may be zero; if not, it is pulled from the caller (approve `BondProtocol` on `BondToken` first). A set can be registered once, ever, and only while `ACTIVE`. Every obligation in it must be in a **live loan token** (`IneligibleAsset`), and it must have no release gate. A set made by an Obligation module (a contingent set) is refused (`ContingentSet`); it can carry a stake only as an external registration (below).

**Which sets can mint.** `BondProtocol` looks up the set's creator (`creatorOf(setId)`) in `BondListing`'s sources. A set may mint only if:

- its creator is the originator of a **live source** (on the testnet, `BulletLoanRequest` and `BulletLoanOffer`, each its own originator), and the loan originated at or after the source went live;
- `ref` names that source's origination for this set (`WrongRef` otherwise): for a request, the loan id; for an offer, the borrower's entry id;
- it has one obligation, and its obligor is the borrower the source names;
- it is **secured**: bound to the deployment's `Lien` over a lock in the deployment's vault, with the lien active.

Any other set registers **stake-only**: it carries a stake but never mints. A set created by any other contract has no source, so `ref` must be zero and the set is stake-only. If a commitment is waiting on the origination (below), `registerSet` refuses (`UseBind`), so a borrower cannot register first without the stake and then take the commitment back.

**What is counted, once, at registration.** For a set that may mint, `BondListing.value` puts the principal and the locked collateral in dollars at 18 decimals, and the registration stores the **counted principal**: the lower of the two. It also stores the **counted rate**: the lowest of the loan's contracted rate, the loan token's curve rate for its term, and `MAX_CURVE_RATE`. If the loan token is unpriced at that moment, `registerSet` reverts (`LoanTokenUnpriced`) rather than spend the set's one registration on a zero. **The counted term starts at registration**, not at origination.

## How collateral and loan tokens are priced

BOND reads no price feed. It reads **Gavel's own spot clearings**: each priced asset is listed with one `SpotMarket`, and its price is **the lowest of that market's last 5 clearings (`PRICE_SAMPLES`), each within 7 days (`PRICE_WINDOW`)**, capped at the asset's `MAX_PRICE`. If the market has fewer than five clearings, any of the five is older than 7 days, the read fails, or the market's quote token is no longer a live dollar loan token, the asset is **unpriced** and counts zero.

A dollar loan token counts at 1 by its listing, and only in a loan in that same token: an mUSDT leg in an mUSDC loan counts zero. A leg in any token that is neither the loan token nor a priced asset counts zero.

On the testnet:

| Kind | Token | Market or terms |
|---|---|---|
| Dollar loan token | mUSDC, mUSDT | Mint cap 10,000,000 BOND each; observation floor 1,000 mUSDC or mUSDT |
| Priced loan token | mWETH | Mint cap 1,000,000 BOND; observation floor 0.3 mWETH; priced as below |
| Priced asset | mWBTC | Its mWBTC/mUSDT `SpotMarket`; `MAX_PRICE` $300,000 |
| Priced asset | mWETH | Its mWETH/mUSDC `SpotMarket`; `MAX_PRICE` $9,000 |
| Priced asset | mcbBTC | Its mcbBTC/mUSDC `SpotMarket`; `MAX_PRICE` $300,000 |

The caps and `MAX_PRICE`s are as at 14 November 2026; read them with `mintCap(token)` and `pricedOf(token)`. An mWETH loan, or any collateral in a priced asset, depends on its market having cleared five times in the last 7 days; `assetPrice(token)` shows whether it is priced now.

## Minting as interest is paid

| Function | Who may call | When |
|---|---|---|
| `resolveObligation(obligationId)` | Anyone | The obligation is no longer `OPEN` or `BREACHED` |
| `resolveSet(setId)` | Anyone | The set is no longer `ACTIVE`; resolves every unresolved paid obligation first, then the stake |

An obligation mints when the registration may mint, the obligor position is still held by the registrant, the obligation is `DISCHARGED`, it was paid **in time** (by its due date plus the set's grace, extended by any time Obligation was paused), and both its interest and its counted rate are above zero. Otherwise it is marked with a reason (`MintSkipped`).

The amount is

```
BOND = countedPrincipal × countedRate × countedTerm / year × releaseScale
```

where the counted term runs from registration (or the obligation's accrual start, if later) to the earlier of payment and the due date, less any Obligation pause after registration, and `releaseScale` is the share of the debt that was paid rather than forgiven. **For a loan held to term, at or below the curve rate, with collateral worth at least the principal, one BOND is minted for each dollar of interest paid**, valued at registration.

Each loan token has a **mint cap**: the most BOND that may ever be minted through it (`minted(token)` counts it). A mint above the room left is cut to the room (`MintCapped`); with no room, nothing is minted (`CAP_REACHED`). A registration reserves no room.

## The curve

`BondCurve` keeps one curve per loan token, in 34 term buckets (7, 30, 90, 180 and 365 days, then yearly to 30 years), each holding its last 32 observations. Observations come only from `observe(source, ref)`, which anyone may call within 30 days (`OBSERVE_WINDOW`) of an origination by a live source, for a secured single-obligation loan with interest, at or above the loan token's observation floor. Each records the contracted rate and up to three distinct lenders.

A bucket is **well-observed** with at least 5 observations within 90 days from at least 3 distinct lenders; its rate is their median (the lower middle of an even count). An unobserved bucket takes the nearest well-observed bucket's rate (a tie takes the lower), and a term beyond every well-observed bucket takes the lowest. With nothing well-observed the rate is zero. `BondProtocol.curveRate(token, term)` returns it, capped at `MAX_CURVE_RATE`, **20% a year** on the testnet, fixed in `BondProtocol`.

## Stakes, commitments and burning

**A registered stake** is returned to the registrant by `resolveSet` if the set ends `DISCHARGED` with every obligation paid in time. In every other ending (breached, enforced, settled, paid late) it is **burned**.

**A commitment** lets a borrower stake on a loan before it exists, so lenders can see it while they bid:

| Function | Who may call | Effect |
|---|---|---|
| `commit(source, ref, amount)` | The borrower the source names, or an operator it approved with `setOperatorApproval` | Adds BOND to the commitment while the origination is `PENDING` and the source live. It can only grow |
| `bind(source, ref)` | The borrower or its operator for 1 day (`BIND_GRACE`) after origination, then anyone | Binds the whole commitment to the set as its stake, with the borrower as registrant |
| `reclaimCommitment(source, ref)` | Anyone | Returns the commitment to the borrower once the origination has `ENDED`, or after a skipped bind |

`bind` registers the set as `registerSet` would, the term counting from the bind; it never reverts on a price (an unpriced loan token counts zero) and a source removed meanwhile still binds, stake-only. If the source contradicts Obligation (another creator or obligor), or the set is already registered or contingent, the bind is skipped and the commitment becomes reclaimable. A set already finished is registered and resolved at once.

**External stakes.** `registerExternal(adapter, externalId, stake, longStop)` lets the obligor an adapter names stake BOND on a debt `BondProtocol` cannot read directly, such as cover written through [insurance](insurance-and-bills.md) (`ContingentBondAdapter`). It never mints. `resolveExternal` burns the stake on a shortfall, and returns it on settlement or once `longStop` (at most 5 years ahead) has passed. An adapter that fails to answer reads as still open, so it can delay a return until `longStop`, no further.

## The listing power

`BondListing`'s owner, on the testnet the deploying account, decides which loan tokens, priced assets and sources **future** registrations count:

- additions (`queueLoanToken`, `queuePriced`, `queueSource`, `queueMarket`, `queueMintCap`) take effect 7 days after they are queued, with no second transaction, and can be cancelled (`cancel`) until then;
- removal (`remove`) is immediate and final: a removed address can never be listed again. It applies to new registrations, commitments and observations only;
- a mint cap or `MAX_PRICE` can be lowered at once (`lowerMintCap`, `lowerMaxPrice`) but raised only through the queue. **A lowered cap applies to existing registrations' later mints**, the one act that reaches a registration already made;
- **the power ends at `LISTING_END`, which on the testnet is 14 November 2027**, or earlier if the owner calls `renounceOwnership`. After that the owner can do nothing, and the lists stand as they are.

The owner cannot move funds, pause anything, touch a stake or commitment, or change a registration's counted principal, rate or term. To check: `LISTING_END()` (1826202129, 14 November 2027 14:22:09 UTC), `endedAt()` (earlier if renounced) and `owner()` on `BondListing`.

**Because this is an admin power, BOND stands outside the [base layer](../concepts/base-layer-and-trust.md) while it lives.**

## Owners, pauses and the vault

`BondProtocol`, `BondCurve` and `BondToken` have **no owner, no setter and no pause**. `BondCurve` has one writer, `BondProtocol`, fixed at deployment. `BondListing` is owned as above and has no pause.

**`BondProtocol` is not a vault consumer.** It never reserves, locks or moves anything in the vault, and needs no `approveConsumer`. It holds stakes and commitments as its own `BondToken` balance, and only reads Obligation, the Lien, the sources and, through `BondListing`, the vault's lock legs and the spot markets.

## Things to know

- **The term counts from registration**, so a late registration forgoes the part before it. A commitment, once bound, counts from the bind.
- **Claiming the collateral ends minting.** `Lien.claim` burns the obligor position, which reads as a change of holder, so an obligation resolved afterwards mints nothing; one resolved after `Lien.release` and before `Lien.claim` still mints. A sale of the position (including one in progress on `ObligorPositionMarket`) stops minting the same way; the stake still returns or burns with the set.
- **A cold curve mints nothing.** The counted rate is fixed at registration; if the loan token's curve is zero then, the registration never mints.
- **The cap is first come.** A borrower can find the room gone at repayment, and the owner can lower it while the power lives.
- **Prices rest on the spot markets' clearings.** Anyone can trade on those markets. A higher print is capped by `MAX_PRICE` and needs five clearings to count; a low print lowers or zeroes the collateral counted in the next registrations.

## Reading it

| View | Returns |
|---|---|
| `getRegistration(setId)` | Registrant, stake, term start, whether it may mint and is secured, status (`ACTIVE`, `RETURNED`, `BURNED`), counted principal and rate, and what is resolved |
| `estimateMint(obligationId)` | What `resolveObligation` would mint now, after the cap |
| `commitment(source, ref)` | Borrower, amount, bound, reclaimed |
| `curveRate(token, term)`, `minted(token)`, `isSource(source)`, `getExternal(adapter, id)` | On `BondProtocol` |
| `loanToken`, `mintCap`, `pricedOf`, `assetPrice`, `value`, `originatorOf`, `pendingMintCap`, `pendingMarket` | On `BondListing` |
| `curveBucket(token, bucket)`, `bucketOf(term)` | On `BondCurve` |

Key events: `SetRegistered`, `CommitmentBound`, `BondMinted`, `MintSkipped` (with its reason), `BondStakeReturned` and `BondStakeBurned` on `BondProtocol`; each queued addition (with its effect time), `ListingRemoved` and `ListingPowerRenounced` on `BondListing`.

## Next

- [Bullet loan](bullet-loan.md): the two sources BOND counts.
- [Spot market](spot-market.md): the clearings BOND prices from.
- [The base layer and what you trust](../concepts/base-layer-and-trust.md): the listing power beside the emergency surface.
