# gavel-developer-docs

Builder documentation for developers **building on** The Gavel Protocol — custom
frontends, orchestrators, curation layers, bots/indexers, and composition
products. If you want to *use* the protocol, start with the protocol repo below;
if you want to *build* on it, you're in the right place.

## Relationship to the protocol repo

The canonical source of truth is
**[The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol)** — it
holds the contracts, deployed addresses, and the direct-access guide. That repo
is *"use the protocol."* This repo is *"build on the protocol."* Where the two
overlap (ABIs, addresses, deployment), the protocol repo is authoritative.

## Scope

This documentation covers:

- **v1 build details** — the current v1 contracts and reference frontend: how to
  integrate against them and how to deploy your own instance.
- **Consuming the Aletheia API** — the Aletheia API documented as a product:
  endpoints, auth, and request/response shapes. It is described as a black box —
  inputs, outputs, and auth, not its implementation.
- **Accessing on-chain data directly** — the free, permissionless path: reading
  protocol state and events over RPC, and the public MCP at `mcp.thegavel.io`.
- **v2 vision** — a roadmap of the coming expansions (CreditPrimitive,
  vault-agnostic collateral, SpotAuction, composition).

**Out of scope:** Aletheia's proprietary data internals — the data pipeline,
indicator computation, yield-curve fitting specifics, market-making/bot logic,
the internal MCP, and any database/infrastructure detail. The API is documented
as a black box, never its implementation.

## Planned structure

The repository is being populated incrementally. Landed sections are linked;
forthcoming sections are listed so the intended shape is clear.

- **`v1/`** — build details for the current v1 protocol
  - architecture *(forthcoming)*
  - contract reference *(forthcoming)*
  - [integration](v1/integration.md) — interacting with the contracts programmatically *(landed)*
  - [deployment](v1/deployment.md) — deploying your own instance *(landed)*
  - frontend build *(forthcoming)*
  - build-your-own-curation-layer *(forthcoming)*
- **`data/`** — accessing protocol data
  - Aletheia API (consumption) *(forthcoming)*
  - on-chain data access *(forthcoming)*
- **`vision/`** — what's coming
  - v2 roadmap *(forthcoming)*

## Licence

[MIT](LICENSE), matching
[The-Gavel-Protocol](https://github.com/JamieFrame/The-Gavel-Protocol).
