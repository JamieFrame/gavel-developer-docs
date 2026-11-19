# Gavel Developer Docs

Documentation for **building on The Gavel Protocol**: permissionless, oracle-free settlement infrastructure for collateralised credit and spot exchange, on Arbitrum.

In a phrase, Gavel is *a public record of promises*. Assets are held in a shared vault, promises are recorded as obligations, collateral is pledged under liens that enforce by rules fixed in advance, and prices and rates come from open auctions rather than price feeds. Loans, bills, insurance and futures are arrangements of those same building blocks, and anyone can build a new one on them.

## Two generations

| | Network | Status |
|---|---|---|
| **V2**, the building blocks | Arbitrum Sepolia (testnet) | Deployed for testing. **Not audited.** The tokens are test tokens with no value. |
| **v1**, the auction-based lending protocol | Arbitrum One (mainnet) | Live and audited ([Sherlock, April 2026](https://github.com/JamieFrame/The-Gavel-Protocol/tree/main/docs/audit)). |

These docs lead with V2. The v1 pages are kept, unchanged in substance, under [`v1/`](v1/) and [`data/`](data/).

V2 is deployed beside v1, not over it: v1's contracts stay where they are and keep running under their own rules.

## Who runs what

- **The Gavel Protocol** is code. It has no operating entity and no treasury, and its core charges no fee. Anyone may call its contracts. Its owner keys hold narrow emergency powers, set out in [the base layer and what you trust](v2/concepts/base-layer-and-trust.md).
- **The data API and the MCP server** described in these docs are operated by **Aletheia Analytics SASU**, a separate company that indexes the protocol's public on-chain record and serves it as data. They are conveniences, not part of the protocol: everything they serve can be read from the chain directly.

## Contents

### V2: what Gavel is for

- [Vision](v2/vision.md): what the protocol is, what it is for, the commitments it is built to, and where it stops. **Start here.**

### V2: concepts

- [Overview](v2/concepts/overview.md): the four layers, the building blocks and what is deployed, and one loan followed from start to finish.
- [The vault](v2/concepts/vault.md): accounts, reservations and locks; consumers and your approval of them.
- [Obligation](v2/concepts/obligation.md): debts and their lifecycle; modules for resolved amounts, triggers and exercise.
- [The Lien](v2/concepts/lien.md): security over collateral, released on payment and enforced on breach.
- [Claims and obligor positions](v2/concepts/claims-and-positions.md): the two tokens every set creates.
- [The engines](v2/concepts/engines.md): the five auction engines the products run on.
- [The base layer and what you trust](v2/concepts/base-layer-and-trust.md): the base layer's rule, the four promises, and the emergency surface.

### V2: products

- [Bullet loan](v2/products/bullet-loan.md) · [Spot market](v2/products/spot-market.md) · [Secondary market](v2/products/secondary-market.md) · [Cash-settled future](v2/products/cash-settled-future.md) · [Insurance and bills](v2/products/insurance-and-bills.md) · [Liquidation cover](v2/products/liquidation-cover.md) · [BOND](v2/products/bond.md)

### V2: building

- [Contracts reference](v2/build/contracts-reference.md): what each contract does, the functions you call, its events and errors.
- [Reading data](v2/build/reading-data.md): from the chain directly, or from the testnet API.
- [Writing](v2/build/writing.md): transactions from your own wallet, and acting for a user through operator delegation.
- [Composing a product](v2/build/composing-a-product.md): building your own product on the core.
- [Agents](v2/agents.md): the MCP server for V2.
- [Deployments](v2/deployments.md): networks, addresses and verified source.

### v1: the live protocol on Arbitrum One

- [Architecture](v1/architecture.md) · [Contract Reference](v1/contracts-reference.md) · [Integration Guide](v1/integration.md) · [Deployment Guide](v1/deployment.md) · [Build Your Own Curation Layer](v1/build-your-own-curation-layer.md) · [Frontend Guide](v1/frontend.md)
- Data: [Aletheia API](data/aletheia-api.md) · [On-Chain Data](data/on-chain-data.md)

## Quick orientation

- **New to Gavel?** [Vision](v2/vision.md), then the [concepts overview](v2/concepts/overview.md).
- **Calling the V2 contracts?** [Writing](v2/build/writing.md), with the [contracts reference](v2/build/contracts-reference.md) and the [deployments](v2/deployments.md).
- **Building a product of your own?** [Composing a product](v2/build/composing-a-product.md), after the concept pages.
- **Reading data or building analytics?** [Reading data](v2/build/reading-data.md).
- **Working with v1 on mainnet?** [v1 architecture](v1/architecture.md), and [The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol) for addresses, verified source, known issues and the audit.

## Companion repository

[The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol) is the *use-it* repository: deployed addresses for both generations, v1's source, known issues and the audit. These docs link to it rather than repeat what it holds.

These docs cover how to use and build on the protocol and how to consume its data. They do not describe how the data API computes its products.

## Licence

MIT.
