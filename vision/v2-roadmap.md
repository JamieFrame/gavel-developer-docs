# v2 Roadmap & Vision

This is a forward-looking document. It describes where The Gavel Protocol is going in v2, why the architecture changes, and — most importantly for a builder — what the v2 substrate will let you build on top. It is a statement of direction and sequence, not a schedule; specific dates are deliberately absent.

The v1 contracts documented elsewhere in this repository are live and stable today. v2 is in active development on testnet.

## The shift: from a product to a substrate

v1 is a lending protocol: auctions, loans, and a position marketplace in one coherent system. v2 keeps everything v1 does, but reorganises it into a **substrate** that many products — built by Gavel and by anyone else — can sit on.

The architecture has three tiers:

```
┌──────────────────────────────────────────────────────────────┐
│  COMPOSITION  —  built by anyone, on top of the primitives    │
│  leverage · futures · options · BOND · insurance · structured │
├──────────────────────────────────────────────────────────────┤
│  PRIMITIVES  —  durable, independently auditable              │
│  CreditPrimitive · AuctionOrchestrator                        │
│  LenderAuctionOrchestrator · SpotAuction                      │
│  PositionNFT · MarketplaceService                             │
├──────────────────────────────────────────────────────────────┤
│  THE VAULT LAYER  —  the true substrate                       │
│  account-first custody · any collateral · shared by all       │
└──────────────────────────────────────────────────────────────┘
```

The principle: **the substrate is durable and changes rarely; the products on it are diverse and evolve freely.** That separation is the whole point — it lets the protocol grow without ever re-auditing or upgrading the foundation, and it lets builders ship products without coordinating with anyone.

## The vault layer — the foundation

In v1, the core contract holds collateral directly. In v2, custody moves into a dedicated **vault layer**, and the vault becomes the foundation everything else sits on.

Two ideas make it powerful:

**Account-first.** A vault is a persistent user account, not a per-loan escrow. You deposit an asset once; from then on, every Gavel product operates on that balance. Borrowing reserves from it, repayment releases back to it, a marketplace sale credits it, a spot trade debits and credits it. Capital flows between uses without leaving the vault — no withdraw-and-redeposit, and for repeat users, no per-action token approvals.

**Vault-agnostic.** The protocol never passes assets by value. An auction references collateral as `(vault, vaultLoanId)`, and the vault encapsulates whatever the asset actually is. To the orchestrator and the credit primitive, every vault looks identical:

- an ERC-20 vault: "user has 50,000 USDC free, 0.5 WBTC reserved against loan 12345"
- an NFT vault: "user has a CryptoPunk reserved against loan 12346"
- a native-BTC vault: "user has 1.2 BTC free, 0.5 BTC reserved, AVS-attested"
- a multichain vault: "user has 500 SOL free, 100 reserved"

Every vault implements one interface (`IVault`) with account operations (`deposit`, `withdraw`, `balance`, `transfer`, `reserve`, `releaseReservation`, `settleReservation`) plus loan-scoped convenience wrappers. Because every operation is just a balance transition inside a vault, products compose naturally.

**Safe by construction.** Only contracts a user has explicitly approved can touch their vault balance — a two-layer model of a governance-set consumer whitelist plus per-user, revocable approval. The first time you use a new product you approve it once (like an ERC-20 approval, but scoped to vault operations and strictly safer); after that it just works.

## The primitives

On top of the vault sit small, single-purpose, independently auditable contracts:

- **CreditPrimitive** — the loan lifecycle, and nothing else: register, repay, claim default. It has *no fee parameter at all* — zero fees aren't a policy setting here, they're structurally absent.
- **AuctionOrchestrator** — reverse-Dutch rate discovery (borrower posts, lenders bid the rate down), settling by registering a loan against a vault reference. The same auction mechanism as v1, carried forward.
- **LenderAuctionOrchestrator** — the mirror image: lenders post, borrowers bid the rate up. Together the two orchestrators give two-sided rate discovery — a market, not just a venue — and a yield curve cross-validated from both directions.
- **SpotAuction** — the Spot Protocol: spot trading on the same vault substrate, so credit and spot share one account model.
- **PositionNFT and MarketplaceService** — the position tokens and the secondary market, carried forward from v1 with the same patterns (the `loanId × 2` token-ID scheme, the MEV-protected `buyPosition`).

Separating rate discovery, custody, and credit lifecycle means each can be audited, deployed, and replaced on its own — and it means a third party can call any of them directly.

### Curation is not a contract in v2

v1 has an on-chain curation layer (`ListingService`) that fronts the core with a whitelist. **v2 removes it, with no on-chain replacement.** The reasoning sharpens the substrate philosophy: the primitives are permissionless, so an on-chain curation contract never actually restricts what a determined user can do — it only restricts what a particular frontend surfaces. That makes curation honestly a UI concern, not a contract one. And because the protocol's zero-fee position is permanent, there is no fee to collect, so the other reason such a contract existed disappears too.

In v2, curation is therefore the frontend's job — it filters what users see and constructs transactions with sensible parameters — backed by a governance-maintained curated-asset list published off-chain. If external integrators (aggregators, third-party frontends) ever need an on-chain signal, a minimal **read-only** registry (`isCuratedLoanToken(token) → bool`, no delegation, no fees, no wrappers) can be added post-launch; the primitives would remain unaware of it. "Build your own curation" in v2 means running your own frontend and asset list over the same permissionless primitives — no contract to deploy, no permission to seek.

### Acting on a user's behalf — the operator surface

v2 generalises v1's operator pattern into a uniform delegation surface across the primitives: alongside each user action sits a `…For` variant that an approved operator can call on the user's behalf, scoped per contract and revocable at any time. This is what lets AI agents, frontends, and third-party automation act for a user without ever taking custody — the operator initiates, but funds only ever move within the user's own vault accounts. It is the on-chain counterpart to the agent-facing data access described in [On-Chain Data](../data/on-chain-data.md).

## How v2 arrives — sibling deployment, not an upgrade

v2 does **not** upgrade v1. The v1 contracts are immutable by design, and that's a feature here: v2 deploys as a sibling at new addresses, and the transition carries no migration risk.

- v1 contracts stay where they are; existing v1 loans run their full lifecycle under v1 rules.
- New origination routes to v2; v1's `createAuction` is paused via its existing `pause()`.
- The data indexer reads both v1 and v2 and presents a unified view throughout.
- As v1's outstanding loans settle naturally, v1 simply goes cold. No forced cutover, no rollover, no position ever put at risk.

This is the operational dividend of non-upgradeable contracts: the protocol evolves by deploying new code beside the old, and v1's audit remains valid for v1 forever.

## The committed path

The protocol's own direction, in order — each stage gated by the previous one's outcome, dates deliberately omitted:

1. **v1 on Arbitrum One** — live today.
2. **v2 Core + Spot** — the vault layer, CreditPrimitive, AuctionOrchestrator, LenderAuctionOrchestrator, PositionNFT, and MarketplaceService, then SpotAuction on the same substrate. Audited, then deployed to mainnet, with v1 traffic redirected as above.
3. **Native BTC Vault** — gated by a decision point after v2 has run on mainnet and the vault interface is validated by real use.

That is the full committed path. Everything beyond it is *architecturally enabled* but not committed — buildable on the substrate, by Gavel or by anyone.

### Native BTC

The headline extension: **trustless native Bitcoin as collateral**, custodied by an EigenLayer AVS using FROST threshold signatures (a restaked-ETH-secured operator multisig), exposed through the same `IVault` interface as every other vault. Because it's just another vault, shipping it gives *every* existing orchestrator native-BTC support at zero additional integration cost — no change to CreditPrimitive, AuctionOrchestrator, MarketplaceService, or PositionNFT.

## What the substrate enables — for builders

The substrate's value is that the composition layer is open. Gavel publishes designs for these products specifically so that anyone can build them; some Gavel may build, many it won't, and several will only ever exist if the wider ecosystem builds them. The catalogue, grouped by which primitives they compose:

**Spot-only (compose SpotAuction):** a TWAP execution router that splits large orders to reduce price impact; a passive quoting vault that earns from spread capture.

**Credit-only (compose CreditPrimitive + the marketplace):** a **BOND** protocol that tokenises position NFTs into standardised fixed-income instruments; an **insurance** layer underwriting protocol risks like bad-debt and default tranches; bilateral OTC credit.

**Cross-primitive (compose both):** a **LeverageRouter** for atomic levered positions (borrow → swap → re-collateralise); cash- and physically-settled futures referenced to spot clearings; batch-cleared options; basis-trading vaults; structured products built from these.

And the vault layer itself is extensible:

- **Custom vaults** — implement `IVault` for any collateral type (tokenised T-bills, LSTs, exotic assets) and every existing orchestrator can use it immediately.
- **Multichain vaults** — accept attested collateral from non-Arbitrum chains.
- **Non-credit products** — anything that operates on user balances (yield aggregators, strategy vaults, rebalancers) can use the vault accounts directly.

In every case the integration requirement is minimal: be an authorised consumer of a vault, and have each user approve you. You never need Gavel's permission, never pay a protocol fee, and never fork anything.

## Principles that won't change

These hold for the substrate, in v2 and beyond:

- **Permissionless composition.** Anyone can build a vault, an orchestrator, a product on user balances, or a curation frontend over the permissionless primitives. No gatekeeping, no integration approval.
- **Fee-free substrate.** The credit primitive cannot charge a fee — it's architecturally absent. Any fees in the ecosystem belong to the products built on top, not the substrate.
- **No token.** The substrate has no token, now or planned. If a composition product wants one, that's its own decision.
- **No DAO.** The protocol's autonomy is structural, not subject to a governance vote.
- **No privileged intervention.** The emergency multisig exists for pause only; there are no plans to extend its powers.

## A note for integrators

If you're building on v1 today, you're building on stable, immutable contracts that will keep working unchanged — v2 arrives beside them, not on top of them. If you're planning for v2, the interface to design against is `IVault` and the primitives' public functions; the substrate is meant to be the durable part you can rely on while the products above it evolve. As the v2 specifications are finalised, the contract reference and integration guides in this repository will be extended to cover them.
