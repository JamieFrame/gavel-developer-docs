# Gavel Developer Docs

Documentation for **building on The Gavel Protocol** — a permissionless, oracle-free, auction-based lending protocol on Arbitrum One. If you want to integrate with the protocol, build a custom frontend or curation layer, consume its market data, or understand where it's heading, you're in the right place.

This repository is the *build-on-it* companion to [The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol), which is the *use-it* repository — the canonical contracts, deployed addresses, verified source, and the direct-access guide live there. These docs assume that as the source of truth for addresses and contract source, and link to it rather than duplicating it.

## What's here

- **v1 build details** — how the live contracts work and how to build against them: architecture, a full contract reference, integration flows, deploying your own instance, building a curation layer, and building a frontend.
- **Data access** — how to consume the Aletheia API and how to read the protocol's data directly on-chain.
- **v2 vision** — where the protocol is going: the substrate/primitive/composition architecture and what it will let you build.

These docs cover how to *consume* the protocol and its data API. They deliberately do **not** cover Aletheia's proprietary data internals (the pipeline, indicator computation, or infrastructure) — the API is documented as a product, not as an implementation.

## Contents

### `v1/` — building on the live protocol

- [Architecture](v1/architecture.md) — the model: layers, the two stacks, access paths, the loan lifecycle, and the properties you can rely on. **Start here.**
- [Contract Reference](v1/contracts-reference.md) — function-level reference for every contract: signatures, structs, events, and errors.
- [Integration Guide](v1/integration.md) — worked ethers v6 flows for borrowing, lending, and trading positions, plus the operator pattern.
- [Deployment Guide](v1/deployment.md) — deploy your own instance of the contracts, including the proxy model and wiring.
- [Build Your Own Curation Layer](v1/build-your-own-curation-layer.md) — run an independent whitelist/router over the permissionless core.
- [Frontend Guide](v1/frontend.md) — build a UI against the contracts and the public API.

### `data/` — accessing protocol data

- [Aletheia API](data/aletheia-api.md) — the REST API for the yield curve, credit surface, and indicators: endpoints, access, and examples.
- [On-Chain Data](data/on-chain-data.md) — read the raw data directly via RPC and events, or through the public MCP server.

### `vision/` — where v2 is going

- [v2 Roadmap & Vision](vision/v2-roadmap.md) — the vault substrate, the credit and spot primitives, the committed path to native BTC, and the open composition catalogue.

## Quick orientation

- **Just want to call the contracts?** [Architecture](v1/architecture.md) → [Integration Guide](v1/integration.md), with the [Direct Access Guide](https://github.com/JamieFrame/The-Gavel-Protocol/blob/main/docs/direct-access-guide.md) for the no-code path.
- **Building a product on top?** [Architecture](v1/architecture.md) → [Build Your Own Curation Layer](v1/build-your-own-curation-layer.md), and the operator pattern in the [Integration Guide](v1/integration.md).
- **Building a frontend or analytics?** [Frontend Guide](v1/frontend.md), [Aletheia API](data/aletheia-api.md), and [On-Chain Data](data/on-chain-data.md).
- **Planning for v2?** [v2 Roadmap & Vision](vision/v2-roadmap.md).

## Licence

MIT.
