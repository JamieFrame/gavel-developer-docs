# Build Your Own Curation Layer

The reference `ListingService` is not special. It reaches the core through the same public operator mechanism any contract can use, and the core grants it no privileges. That means **anyone can build an independent curation layer** — your own whitelist, your own fee model, your own brand and rules — over the same permissionless core, with no permission required and no custody risk. Multiple curation layers can coexist over one core; the reference one is simply the first.

This guide shows how. Read [Integration](integration.md) for the operator pattern first; this builds directly on it.

## What a curation layer is

A curation layer is an **operator** that routes auctions (and optionally listings) to the core on behalf of users who have approved it. It adds whatever value you want — a vetted token set, a fee, jurisdiction filters, extra safety checks, a curated UI — while custody stays entirely in the core. It never holds user collateral.

The core enforces two things that make this safe:

- **It uses the user's own deposited collateral.** `createAuctionFor` debits `collateralBalances[user][token]` — collateral the user deposited to the core directly. Your layer never touches it.
- **It checks the user opted in.** `createAuctionFor` reverts `Unauthorized` unless the caller is the user or an approved operator of the user. Your layer can only act for users who explicitly approved it.

So a curation layer can curate and route, but it cannot move a non-approving user's funds, cannot take custody, and cannot bypass any core rule.

## A minimal curation layer

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Ownable}    from "@openzeppelin/contracts/access/Ownable.sol";
import {IERC20}     from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20}  from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ILoanProtocol} from "./interfaces/ILoanProtocol.sol";

contract MyCurationLayer is Ownable {
    using SafeERC20 for IERC20;

    ILoanProtocol public immutable loanProtocol;
    address public treasury;
    uint256 public feeBps; // 0 = no fee

    mapping(address => bool) public collateralWhitelist;
    mapping(address => bool) public loanTokenWhitelist;

    error TokenNotWhitelisted();

    event Whitelisted(address indexed token, bool collateral, bool allowed);
    event CuratedAuctionCreated(uint256 indexed auctionId, address indexed borrower);

    constructor(address _loanProtocol, address _treasury) Ownable(msg.sender) {
        loanProtocol = ILoanProtocol(_loanProtocol);
        treasury = _treasury;
    }

    // --- Admin: define your curated set and fee ---
    function setCollateralWhitelist(address token, bool ok) external onlyOwner {
        collateralWhitelist[token] = ok;
        emit Whitelisted(token, true, ok);
    }
    function setLoanTokenWhitelist(address token, bool ok) external onlyOwner {
        loanTokenWhitelist[token] = ok;
        emit Whitelisted(token, false, ok);
    }
    function setFee(uint256 _feeBps) external onlyOwner { feeBps = _feeBps; }

    // --- User entrypoint ---
    // Preconditions (the user does these on the CORE, once):
    //   1. loanProtocol.depositCollateral(collateralToken, collateralAmount)
    //   2. loanProtocol.setOperatorApproval(address(this), true)
    function createCuratedAuction(
        address collateralToken, uint256 collateralAmount,
        address loanToken, uint256 loanAmount, uint256 maxRepayment,
        uint256 loanDuration, uint256 auctionDuration, uint256 bidStep
    ) external returns (uint256 auctionId) {
        if (!collateralWhitelist[collateralToken]) revert TokenNotWhitelisted();
        if (!loanTokenWhitelist[loanToken])       revert TokenNotWhitelisted();

        // Optional fee, pulled from the user (requires their approval of THIS contract
        // for the fee token — separate from the collateral deposited to the core).
        if (feeBps > 0) {
            uint256 fee = (collateralAmount * feeBps) / 10_000;
            if (fee > 0) IERC20(collateralToken).safeTransferFrom(msg.sender, treasury, fee);
        }

        // Route to the core. It debits the user's pre-deposited collateral and
        // verifies operatorApprovals[msg.sender][address(this)] == true.
        auctionId = loanProtocol.createAuctionFor(
            msg.sender,   // borrower
            msg.sender,   // collateralFrom (must equal borrower)
            collateralToken, collateralAmount,
            loanToken, loanAmount, maxRepayment,
            loanDuration, auctionDuration, bidStep
        );
        emit CuratedAuctionCreated(auctionId, msg.sender);
    }
}
```

## The user flow

From the user's side, using your layer is three steps — the first two on the core, the third on your layer:

```js
// 1. Deposit collateral to the CORE (approve the core first)
await wbtc.approve(loanProtocol.target, amount);
await loanProtocol.depositCollateral(WBTC, amount);

// 2. Approve your curation layer as an operator on the CORE
await loanProtocol.setOperatorApproval(MY_CURATION_LAYER, true);

// 3. (If you charge a fee) approve your layer for the fee token, then route
await wbtc.approve(MY_CURATION_LAYER, feeAmount);
await myCurationLayer.createCuratedAuction(/* params */);
```

A user can approve several curation layers at once, or none and go direct. Approval is revocable at any time with `setOperatorApproval(layer, false)`.

## What you can customise

The operator model is the only fixed part; everything above it is yours:

- **Selection** — your own criteria for which tokens (or NFT collections) you list, and how you vet them.
- **Fee model** — none, flat, percentage, tiered; in any token; to any treasury. (The reference layer takes a fee in the collateral token, atomically, before delegating — if the fee transfer fails, no auction is created.)
- **Extra checks** — per-token caps, allowlists/denylists of users, jurisdiction filters, rate guards.
- **A curated marketplace** — the same pattern works for secondary trading via `listPositionFor`, so you can offer a curated position market too.
- **Brand and UX** — your own frontend and identity over the shared liquidity of the core.

## Peer, not privileged

This is the architectural point worth internalising: a curation layer holds no special status. The core treats your layer exactly as it treats any operator — it can act only for users who approved it, only on their own deposited collateral, and only within the core's unchangeable rules. There can be many curation layers over one core at once, competing on selection, fees, and UX, all sharing the same immutable settlement and the same liquidity. Curating is permissionless; it is not a gatekeeping role granted by anyone.

> **Note on v2.** This on-chain curation-layer pattern is a v1 feature. In v2 there is no on-chain curation contract at all — curation becomes a purely off-chain concern (a frontend plus an off-chain curated-asset list) over the same permissionless primitives, since the protocol takes no fees and an on-chain layer never actually restricts what a determined user can do. See the [v2 Roadmap](../vision/v2-roadmap.md). If you're building curation for v2, build a frontend and asset list, not a contract.

## NFT stack

The same applies to NFT-collateralised lending. Build against `NFTLoanProtocol` instead, whitelist **collections** rather than tokens, and call `NFTLoanProtocol.createAuctionFor` with `(collateralNFT, collateralTokenId, …)`. The operator and custody guarantees are identical.

## Next

- [Frontend Guide](frontend.md) — putting a UI on your layer or on the core.
- [Contract Reference](contracts-reference.md) — exact signatures for `createAuctionFor`, `listPositionFor`, and the operator functions.
