# Writing

This page shows how to send transactions to Gavel V2 on Arbitrum Sepolia (chain 421614): getting test tokens, the approvals each call needs, how assets are named, and worked examples for the vault, the bullet loan and the spot market. It ends with operator delegation, the way to let another address submit transactions for you.

Every example uses ethers v6. Addresses are in the [address list](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.md) (machine-readable: [`v2-testnet-addresses.json`](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/v2-testnet-addresses.json)) and on the [deployments](../deployments.md) page; full signatures are in the [contracts reference](contracts-reference.md). The contracts are **not audited**.

## Setup

```js
import { ethers } from "ethers";

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL); // Arbitrum Sepolia, 421614
const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
const me = await signer.getAddress();

// The bullet-loan example needs two accounts: an auction refuses its principal as a bidder.
const borrower = new ethers.Wallet(process.env.BORROWER_KEY, provider);
const lender = new ethers.Wallet(process.env.LENDER_KEY, provider);
const B = await borrower.getAddress();
const L = await lender.getAddress();

// From the address list:
const VAULT = "0x...";
const OBLIGATION = "0x...";
const LIEN = "0x...";
const BULLET_LOAN_REQUEST = "0x...";
const SPOT_WBTC_USDC = "0x...";   // the mWBTC/mUSDC SpotMarket
const MUSDC = "0x...";            // 6 decimals
const MWBTC = "0x...";            // 8 decimals
```

You also need a little Arbitrum Sepolia ETH for gas.

## Test tokens

The test tokens (`mWBTC`, `mUSDC`, `mUSDT`, `mWETH`, `mcbBTC`) are plain ERC-20s with an open mint. Anyone can mint any amount to any address:

```js
const erc20 = (addr, runner = signer) => new ethers.Contract(addr, [
  "function mint(address to, uint256 amount)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "function balanceOf(address) view returns (uint256)",
], runner);

await (await erc20(MUSDC).mint(me, 50_000n * 10n ** 6n)).wait();  // 50,000 mUSDC
await (await erc20(MWBTC).mint(me, 1n * 10n ** 8n)).wait();       // 1 mWBTC
```

The same contracts also have `burn(address from, uint256 amount)`, callable by anyone against any holder, the vault included. They are worth nothing, and real tokens do not behave this way.

## Asset keys

The vault, Obligation and the Lien name an asset by a 32-byte key, not by its address. An ERC-20's key is `0x01`, then eleven zero bytes, then the token address:

```js
const erc20Key = (token) => ethers.concat(["0x01", ethers.zeroPadValue(token, 31)]);

const USDC_KEY = erc20Key(MUSDC);
const WBTC_KEY = erc20Key(MWBTC);
```

A key with anything but zeros in bytes 1 to 11, or a zero address, is refused (`InvalidAsset`). The vault and the core always take keys. Products take both: some inputs are a token address (a loan's `token`, a sale's `paymentToken`), while collateral, lock parameters and expected assets are keys (`RequestTerms.lockParams`, `OfferTerms.collateral`, `CoverOrchestrator`'s `asset`, `collateralKey` and `Expected.asset`, `ClaimLotMarket`'s `expectedAsset`). Each signature in the [contracts reference](contracts-reference.md) says which. See [the vault](../concepts/vault.md#accounts-and-assets).

## The approvals

Two kinds of approval, for two different things.

| Approval | Given on | Lets | Needed for |
|---|---|---|---|
| `approve(VAULT, amount)` | The token | The vault pull the token from your wallet | `deposit`, and a collateral lock that draws a shortfall from your wallet |
| `vault.approveConsumer(product, true)` | The vault | One contract reserve, lock and move **your** vault balance | Every product you bid, order or open through |
| `vault.approveConsumer(OBLIGATION, true)` | The vault | Obligation take a payment from your vault balance | `obligation.pay(..., VAULT)` |
| `vault.approveConsumer(LIEN, true)` | The vault | The Lien take a payoff from your vault balance | `lien.payoff(..., VAULT)` |
| `approve(OBLIGATION or LIEN, amount)` | The token | A payment pulled from your wallet | `pay` or `payoff` with `WALLET` |

A consumer approval is per contract: approving `BulletLoanRequest` does not approve `BulletLoanOffer` or any `SpotMarket`. It also works only while the vault's owner keeps that contract on the whitelist; check with `isAuthorisedConsumer(contract)` and `isUserApproved(you, contract)`. `approveConsumer(contract, false)` revokes it; commitments already made (a reservation a product has taken, a lock it holds) still complete.

```js
const vault = new ethers.Contract(VAULT, [
  "function deposit(address user, bytes32 asset, uint256 amount, bytes params)",
  "function withdraw(address user, bytes32 asset, uint256 amount, bytes externalDestination) returns (bytes32)",
  "function approveConsumer(address consumer, bool approved)",
  "function balance(address user, bytes32 asset) view returns (uint256)",
  "function isAuthorisedConsumer(address consumer) view returns (bool)",
  "function isUserApproved(address user, address consumer) view returns (bool)",
], signer);
```

## Deposit and withdraw

`deposit(user, asset, amount, params)` must be called by `user` itself; `params` is empty for an ERC-20. `withdraw(user, asset, amount, destination)` sends free balance to `destination`, ABI-encoded as an address, and refuses zero and the vault's own address.

```js
const amount = 10_000n * 10n ** 6n;                                    // 10,000 mUSDC
await (await erc20(MUSDC).approve(VAULT, amount)).wait();
await (await vault.deposit(me, USDC_KEY, amount, "0x")).wait();

const free = await vault.balance(me, USDC_KEY);                        // free balance

const dest = ethers.AbiCoder.defaultAbiCoder().encode(["address"], [me]);
await (await vault.withdraw(me, USDC_KEY, 1_000n * 10n ** 6n, dest)).wait();
```

A deposit that does not arrive in full reverts (`UnexpectedDepositAmount`). Withdrawal of free balance is never paused and needs no one's consent.

## A bullet loan, end to end

A borrower requests 10,000 mUSDC from one lender for 30 days against 0.2 mWBTC; a lender bids; anyone settles; the borrower repays and takes back the collateral. The rules are on the [bullet loan](../products/bullet-loan.md) page.

```js
const TERMS = "tuple(address token, bytes lockParams, uint128 amount, uint128 minAmount, " +
  "uint256 maxRepayment, uint256 bidStep, uint64 term, uint64 duration)";

const request = new ethers.Contract(BULLET_LOAN_REQUEST, [
  `function openRequest(address borrower, ${TERMS} t, bool many) returns (uint256)`,
  "function bidRequest(address lender, uint256 loanId, uint256 quantity, uint256 repayment, " +
    "uint256 minFill, uint256 hintPrev, bytes32 expectedTerms) returns (uint256)",
  "function finalise(uint256 loanId, uint256 maxSteps)",
  "function deadlineOf(uint256 loanId) view returns (uint256 end, uint256 deadline)",
  "function originationOf(uint256 ref) view returns (uint256 setId, uint256 obligationId, " +
    "uint256 principal, uint256 repayment, uint64 dueAt, uint256 lienId)",
  "event LoanOpened(uint256 indexed loanId, uint8 opening, address indexed opener, uint256 auctionId)",
], signer);

const coder = ethers.AbiCoder.defaultAbiCoder();
const requestB = request.connect(borrower), requestL = request.connect(lender);
const vaultB = vault.connect(borrower), vaultL = vault.connect(lender);
```

### 1. The borrower opens a request

The collateral is described as the vault's lock parameters: `abi.encode(depositor, keys[], amounts[])`, the depositor being the borrower, the keys in strictly ascending order. The product locks it at opening, from the borrower's free balance first and then from the wallet, so either deposit it or approve the vault on the token.

```js
// Borrower (holding 0.2 mWBTC in its wallet)
await (await vaultB.approveConsumer(BULLET_LOAN_REQUEST, true)).wait();
await (await erc20(MWBTC, borrower).approve(VAULT, 20_000_000n)).wait();   // pulled at the lock

const terms = {
  token: MUSDC,
  lockParams: coder.encode(["address", "bytes32[]", "uint256[]"], [B, [WBTC_KEY], [20_000_000n]]),
  amount: 10_000n * 10n ** 6n,
  minAmount: 0n,                       // used only by a request to many
  maxRepayment: 10_100n * 10n ** 6n,   // the most the borrower will repay
  bidStep: 10n * 10n ** 6n,            // to one: at most 10% of the amount
  term: 30n * 86400n,                  // 7 days to 30 years
  duration: 86400n,                    // auction: 1 to 7 days, hard close
};

const openTx = await requestB.openRequest(B, terms, false);           // false: to one lender
const rc = await openTx.wait();
const loanId = rc.logs
  .map((l) => request.interface.parseLog(l))
  .find((e) => e?.name === "LoanOpened").args.loanId;
```

### 2. A lender bids

A lender's principal must already be free in the vault: the bid reserves it there. Each bid carries `expectedTerms`, `keccak256(abi.encode(terms, many))`, and reverts `TermsChanged` against any other terms. The contract has no view of the terms; a lender reads them from the opening transaction's calldata (as below, when the borrower or its operator called `openRequest` directly).

```js
// Lender, with at least 10,000 mUSDC free in the vault (deposited as above)
await (await vaultL.approveConsumer(BULLET_LOAN_REQUEST, true)).wait();

const tx = await provider.getTransaction(openTx.hash);     // the borrower's openRequest transaction
const { args } = request.interface.parseTransaction(tx);   // args.t, args.many
const expectedTerms = ethers.keccak256(coder.encode([TERMS, "bool"], [args.t, args.many]));

await (await requestL.bidRequest(
  L, loanId,
  args.t.amount,             // to one: the whole amount
  10_080n * 10n ** 6n,       // the repayment accepted: at least the amount, at most maxRepayment
  0n, 0n,                    // minFill and hintPrev: used only by a request to many
  expectedTerms,
)).wait();
```

A later bid must be lower by at least `bidStep`. When a bid is beaten, the displaced lender's reservation is released in the same transaction.

### 3. Anyone finalises

After the auction's end (`deadlineOf(loanId)` returns it with any pause added), anyone may call `finalise`. For a request to one, `maxSteps` is ignored.

```js
await (await request.finalise(loanId, 0n)).wait();
const o = await request.originationOf(loanId);  // setId, obligationId, principal, repayment, dueAt, lienId
```

In that one transaction Obligation creates the set (the borrower holds the obligor position, the lender 1,000,000 claim units), the collateral lock passes to the Lien, and the lender's principal moves to the borrower's free balance. Past the settlement deadline, `finalise` reverts `Stalled` and the escrow comes back through `release` and `expire`.

### 4. The borrower repays

`pay(obligationId, amount, source)` may be called by anyone. `amount` is a ceiling: only the payoff is charged. A secured set must be paid in full in one payment (`MustPayWhole`). The bullet loan's prepayment is `FULL`, so before the due date the payoff is the full repayment. `source` is `0` (`VAULT`) or `1` (`WALLET`).

```js
const obligation = new ethers.Contract(OBLIGATION, [
  "function pay(uint256 obligationId, uint256 amount, uint8 source)",
  "function payoff(uint256 id) view returns (uint256)",
  "function collect(uint256 obligationId, bytes32 key, address destination, bool toVault) returns (uint256)",
], provider);

// Borrower, paying from the vault: approve Obligation as a consumer first
await (await vaultB.approveConsumer(OBLIGATION, true)).wait();
const due = await obligation.payoff(o.obligationId);              // 10,080 mUSDC
// The principal (10,000) is free in the vault; deposit the rest before paying.
const freeB = await vault.balance(B, USDC_KEY);
if (due > freeB) {
  await (await erc20(MUSDC, borrower).approve(VAULT, due - freeB)).wait();
  await (await vaultB.deposit(B, USDC_KEY, due - freeB, "0x")).wait();
}
await (await obligation.connect(borrower).pay(o.obligationId, due, 0)).wait();   // 0: VAULT
```

Paying from the wallet instead is `approve(OBLIGATION, due)` on the token, then `pay(o.obligationId, due, 1)`.

The lender then collects, to a wallet (`toVault = false`) or to a vault account (`true`):

```js
// Lender
await (await obligation.connect(lender).collect(o.obligationId, USDC_KEY, L, false)).wait();
```

### 5. Release and claim the collateral

Once the set is paid, anyone may `release` the lien; the collateral is credited to whoever holds the obligor position. That holder then calls `claim`, which moves it to their free balance in the vault and burns the position. Nothing is pushed.

```js
const lien = new ethers.Contract(LIEN, [
  "function release(uint256 lienId)",
  "function claim(uint256 lienId)",
], borrower);

await (await lien.release(o.lienId)).wait();          // anyone may release
await (await lien.claim(o.lienId)).wait();            // the obligor position's holder: here the borrower
await (await vaultB.withdraw(B, WBTC_KEY, 20_000_000n, coder.encode(["address"], [B]))).wait();
```

If the loan is not paid by the due date plus 7 days (pause-adjusted), anyone may call `markBreached`, then `lien.payoff` or `lien.enforce`; see [the Lien](../concepts/lien.md).

## A spot order

Each pair is one `SpotMarket`. A sell reserves the base amount; a buy reserves `amountBase × limitPrice / 10^18` of the quote. Prices are raw quote units per raw base unit, scaled by 10^18: on mWBTC/mUSDC, 100,000 mUSDC per mWBTC is `10^21`. See the [spot market](../products/spot-market.md).

```js
const spot = new ethers.Contract(SPOT_WBTC_USDC, [
  "function placeOrder(uint8 side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 hintPrev) returns (uint256)",
  "function findHint(uint8 side, uint256 limitPrice, uint256 fromOrderId, uint256 maxSteps) view returns (uint256)",
  "function cancelOrder(uint256 orderId)",
  "function placeOrderFor(address owner, uint8 side, uint256 amountBase, uint256 limitPrice, uint256 expiresAtSlot, uint256 hintPrev) returns (uint256)",
  "function setOperatorApproval(address operator, bool approved)",
], signer);

const SELL = 1;                                  // BUY = 0
const limit = 10n ** 21n;                        // 100,000 mUSDC per mWBTC
const size = 1_000_000n;                         // 0.01 mWBTC

await (await vault.approveConsumer(SPOT_WBTC_USDC, true)).wait();
// ...and 0.01 mWBTC deposited in the vault

const hint = await spot.findHint(SELL, limit, 0n, 100n);
await (await spot.placeOrder(SELL, size, limit, 0n, hint)).wait();   // expiresAtSlot 0: no expiry
```

The order id is in the `OrderPlaced` event, which the `BatchAuction` engine emits with the order's owner. If the book moved between `findHint` and inclusion, the call reverts `StaleHint` and can be retried. `cancelOrder(orderId)` is never paused.

## Operator delegation

An **operator** is an address you let submit transactions for you: an agent, a relayer, a batching service. Approval is **per contract**: `setOperatorApproval(operator, true)` on one contract gives the operator nothing on any other, and `false` revokes it. `isOperator(user, operator)` reads it.

**The principal keeps the value.** In every delegated call you are named as the principal, and you, not the operator, own the escrow, the order, the loan and the proceeds. The calls that pay out pay only you: `withdrawFor` sends only to your own address, `collectFor` credits only your vault account. An operator cannot approve consumers or other operators for you, and token approvals stay yours to give.

Contracts that offer it, as deployed:

| Contract | Delegated calls |
|---|---|
| `Vault` | `depositFor(user, ...)` (draws on `user`'s wallet, under `user`'s token approval), `withdrawFor(user, asset, amount)` (pays `user` only) |
| `Obligation` | `collectFor(holder, id, key)`, into the holder's vault account. Forgiving debt needs a separate approval, `setReleaseApproval`, for `releaseFor` |
| `BulletLoanRequest`, `BulletLoanOffer` | The principal is the first argument: `openRequest(borrower, ...)`, `bidRequest(lender, ...)`, `openOffer(lender, ...)`, `bidOffer(borrower, ...)`; and `cancel(loanId)` by the opener's operator |
| Each `SpotMarket` | `placeOrderFor(owner, ...)`, `cancelOrderFor(owner, orderId)` |
| `ClaimLotMarket`, `ObligorPositionMarket` | `createAuctionFor(user, ...)`, `bidFor(user, ...)`, `cancelAuctionFor(user, auctionId)` |
| `FutureOrchestrator` | `offerFor(maker, ...)`, `cancelOfferFor`, `takeFor(taker, ...)` |
| `CoverOrchestrator` | `createOfferFor`, `cancelOfferFor`, `insureFor`, `insureViaFor`, `claimUnitsFor`, `transferBeneficiaryFor` |
| `BillPackager` | `depositFor(lender, ...)`, `redeemFor(holder, ...)` |
| `BondProtocol` | `commit` and `bind` for a borrower, by its operator |

Some delegated calls move value by what they do: `transferBeneficiaryFor` moves a policy's beneficiary to an address the operator names, and the secondary market's `createAuctionFor` puts your claim units or obligor position up for sale. An operator approved on `CoverOrchestrator` or on a secondary market can do those things for you.

The Lien has no operator approval: `release`, `payoff` and `enforce` are open to anyone, and `claim` is the obligor position's holder's alone.

```js
// USER approves AGENT once, on this spot market only:
await (await spot.setOperatorApproval(AGENT, true)).wait();

// The agent, later, for the user:
const spotAsAgent = spot.connect(agentSigner);
await (await spotAsAgent.placeOrderFor(USER, SELL, size, limit, 0n, hint)).wait();
```

The order belongs to `USER`, its reservation is on `USER`'s vault balance (which `USER` must have approved the market to use), and its fills settle to `USER`. A call by an address with no approval reverts `NotApprovedOperator`.

## Errors you will meet

| Error | Meaning |
|---|---|
| `ConsumerNotAuthorised(consumer)` | The contract is not on the vault's whitelist |
| `UserHasNotApproved(user, consumer)` | You have not called `approveConsumer` for it |
| `InsufficientBalance(requested, available)` | Not enough free balance in the vault |
| `UnexpectedDepositAmount` | The token delivered less than the amount |
| `TermsChanged` | A bullet loan bid against terms other than the opening's |
| `StaleHint` | The sorted book moved; recompute the hint and retry |
| `MustPayWhole` | A part-payment on a secured set |
| `NotApprovedOperator` | The caller is neither the principal nor its approved operator |

## Next

- [Reading data](reading-data.md): views, events and the rules for reading them.
- [Composing a product](composing-a-product.md): building on the core.
- [The vault](../concepts/vault.md), [Obligation](../concepts/obligation.md), [the Lien](../concepts/lien.md).
