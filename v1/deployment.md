# Deployment Guide

The Gavel Protocol is MIT-licensed and permissionless. This guide shows how to **deploy your own instance** of the contracts — for a private market, a testnet, or a fork. It is generic builder guidance; for the addresses of the canonical Arbitrum One deployment, see the protocol repo's [deployed contracts](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/deployed-contracts.md).

## Prerequisites

- [Foundry](https://book.getfoundry.sh/) installed.
- An RPC endpoint and a funded deployer account on your target chain.
- The contracts from `src/` (compile with `forge build`).

## The proxy model

Each contract is an implementation deployed behind a minimal **ERC1967 proxy**. The implementations use OpenZeppelin's upgradeable base contracts, which means they use an `initialize(...)` function instead of a constructor — but they are **not** UUPS and there is **no proxy admin**, so once deployed the logic cannot be upgraded. This is deliberate: upgrades are done by deploying a new instance ("redeploy-as-v2"), never in place. If you deploy your own instance, it inherits the same immutability.

## Deploy order and the circular wiring

Within a stack, the core and the position NFT reference each other, and the reference is fixed at initialisation with no setter. The init signatures are:

```
PositionNFT.initialize(address _loanProtocol)
LoanProtocol.initialize(address _positionNFT)
ListingService.initialize(address _loanProtocol)
```

(The NFT stack is analogous: `NFTPositionNFT.initialize(_loanProtocol)`, `NFTLoanProtocol.initialize(_positionNFT)`, `NFTListingService.initialize(_loanProtocol)`.)

Because each of the core and the position NFT needs the other's address, you resolve the cycle by deploying the position-NFT proxy first **without** initialising it, deploying the core initialised with the position-NFT address, then initialising the position NFT with the core address:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {PositionNFT}    from "../src/PositionNFT.sol";
import {LoanProtocol}   from "../src/LoanProtocol.sol";
import {ListingService} from "../src/ListingService.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();

        // 1. Implementations
        PositionNFT    posImpl  = new PositionNFT();
        LoanProtocol   coreImpl = new LoanProtocol();
        ListingService lsImpl   = new ListingService();

        // 2. PositionNFT proxy — deployed UNINITIALISED (empty init data)
        ERC1967Proxy posProxy = new ERC1967Proxy(address(posImpl), "");

        // 3. LoanProtocol proxy — initialised with the PositionNFT proxy address
        bytes memory coreInit =
            abi.encodeCall(LoanProtocol.initialize, (address(posProxy)));
        ERC1967Proxy coreProxy = new ERC1967Proxy(address(coreImpl), coreInit);

        // 4. Now initialise PositionNFT with the LoanProtocol proxy address
        PositionNFT(address(posProxy)).initialize(address(coreProxy));

        // 5. (Optional) Curation layer — initialised with the core
        bytes memory lsInit =
            abi.encodeCall(ListingService.initialize, (address(coreProxy)));
        ERC1967Proxy lsProxy = new ERC1967Proxy(address(lsImpl), lsInit);

        vm.stopBroadcast();
    }
}
```

Run with:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast
```

Always interact with the **proxy** addresses, never the implementation addresses.

## Post-deploy configuration (curation layer)

The core needs no configuration — it accepts any token immediately. The optional Curation Layer is where you define your curated set:

```solidity
// Whitelist collateral and loan tokens (loan tokens require a non-zero min bid step)
ListingService(lsProxy).setCollateralWhitelist(WBTC, true);
ListingService(lsProxy).setLoanTokenWhitelist(USDC, true, MIN_BID_STEP);

// NFT stack: whitelist collections instead
NFTListingService(nftLsProxy).setCollectionWhitelist(COLLECTION, true);
```

`setLoanTokenWhitelist` reverts (`MinBidStepRequired`) if you pass a zero min bid step when whitelisting — set a meaningful floor for the token's decimals.

## Ownership and decentralisation

After deploy, each contract's owner is the deployer. The owner's powers are narrow: `pause`/`unpause` on the core, and whitelist/fee configuration on the curation layer. There is no upgrade power. Transfer ownership to a multisig:

```solidity
PositionNFT(posProxy).transferOwnership(SAFE);
LoanProtocol(coreProxy).transferOwnership(SAFE);
ListingService(lsProxy).transferOwnership(SAFE);
```

`Ownable` here is one-step and irreversible — verify the target address before transferring. To fully decentralise, the owner can later renounce ownership, after which even pause is gone and only the immutable logic remains.

## Verification

Verify each implementation on the block explorer so the source is publicly inspectable:

```bash
forge verify-contract <IMPLEMENTATION_ADDR> src/LoanProtocol.sol:LoanProtocol \
  --chain arbitrum --watch
```

Explorers will then surface the proxy as a proxy and link it to the verified implementation.

## Notes

- **Test on a testnet first.** The position↔core linkage is permanent; a mis-wired deployment cannot be fixed in place — you would redeploy.
- **Immutability is a feature, not an oversight.** Don't add an upgrade mechanism expecting to match the reference; the design is non-upgradeable by intent.
- **Decimals and bid steps** are per token — choose `minBidStep` appropriately (e.g. a 6-decimal stablecoin and an 18-decimal token need different floors).
- For the canonical addresses and verified source, see the [protocol repo](https://github.com/JamieFrame/The-Gavel-Protocol).
