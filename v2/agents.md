# Agents

AI agents can work with Gavel V2 through an MCP (Model Context Protocol) server: a set of tools that read the testnet's record and prepare transactions for a user's own wallet to sign. The server is operated by **Aletheia Analytics SASU**. The protocol itself has no operator. The server's read tools serve the [testnet API](build/reading-data.md)'s data; its write tools read the chain for their checks.

| | |
|---|---|
| Network | Gavel V2 on **Arbitrum Sepolia** (chain id 421614), not audited |
| Tools | 37: 18 read, 19 write |
| Endpoint | `https://mcp-testnet.thegavel.io/mcp` |

**Nothing on the server signs or broadcasts a transaction, holds keys or takes custody of funds.** A write tool returns calldata; the user's wallet decides whether to sign it.

## Read tools

| Area | Tools |
|---|---|
| Deployment and activity | `get_deployment`, `get_tape`, `get_contract_events`, `get_calendar` |
| Accounts | `get_account` |
| Auctions | `list_auctions`, `get_auction_bids` |
| Bullet loans and rates | `list_bullet_loans`, `get_curve` |
| Spot market | `list_spot_markets`, `get_spot_market` |
| Cash-settled future | `list_future_offers`, `list_futures` |
| Insurance and bills | `list_cover_offers`, `list_policies`, `get_bill_series` |
| Secondary market | `list_secondary_sales`, `get_secondary_sale_bids` |

Every answer carries its **provenance**. That includes the testnet API's own fields (the block the data is indexed to, the source, the ABI set events were decoded with, and its account marking), as described in [reading data](build/reading-data.md); the server adds the chain, the deployment records it uses and the routes it read. Answers also carry a **sample block** that the server computes, typically: how many rows it holds, whether it was truncated, how many distinct addresses appear, and the share involving accounts in the provider's published register, or why that share is not given. Some tools report a reduced or per-section form.

Each answer also carries a disclosure: that this is a test network, that the tokens have no value, that the contracts are not audited, and how much of the activity involves accounts in the provider's register.

Some data is not served yet. The answers where it would appear carry a structured absence, naming what is missing and why, rather than an empty result:

- vault balances;
- the state of an obligation, a set or a lien by its id;
- BOND's listing;
- liquidation-cover accounts.

All of it can be read from the chain; see [reading data](build/reading-data.md) and the [contracts reference](build/contracts-reference.md).

## Write tools

| Area | Tools |
|---|---|
| Test tokens | `prepare_test_token_mint` |
| Vault | `prepare_vault_deposit`, `prepare_vault_withdraw`, `prepare_vault_product_approval` |
| Spot market | `prepare_spot_order`, `prepare_spot_cancel` |
| Bullet loan | `prepare_bullet_request_open`, `prepare_bullet_request_bid`, `prepare_bullet_offer_open`, `prepare_bullet_offer_bid`, `prepare_bullet_cancel`, `prepare_bullet_entry_release`, `prepare_bullet_finalise` |
| Obligation | `prepare_obligation_pay`, `prepare_obligation_collect`, `prepare_obligation_redeem` |
| Lien | `prepare_lien_payoff`, `prepare_lien_release`, `prepare_lien_claim` |

Each write tool returns an **unsigned blueprint**:

- **The transaction:** `{ to, data, value, chain_id }`, with `value` zero and `chain_id` 421614.
- **Prerequisites, at exact amounts:** the ERC-20 approval, vault deposit or vault consumer approval the transaction needs first, listed only where the chain shows one is missing, each with its own transaction and a line saying why it is needed. An approval is never for more than the amount. The steps are the ones set out in [writing](build/writing.md).
- **A simulation:** when no prerequisite is outstanding, the transaction is run with `eth_call` from the user's address; otherwise the answer says it was not run. A simulation reflects the state at that moment, and the state can change before the transaction is sent.
- **An EIP-681 link**, where every argument is a scalar; otherwise `null`, with the reason.
- **A wallet-agnostic signing link**, `null` for now, with the reason.

A call that would fail returns no blueprint. Before encoding, the server reads the chain and refuses calls it can see would revert, naming the contract's error (`PrincipalCannotBid`, for example). A revert found by the simulation is returned as an error naming the contract's error (`MustPayWhole` and so on), or `UnknownError` where the error is not recognised.

The tools cover the core and the bullet loan, the spot market and the test tokens. The other products' transactions can be built from their signatures in the [contracts reference](build/contracts-reference.md).

## v1

The v1 MCP server, for the protocol on Arbitrum One, is described in [on-chain data](../data/on-chain-data.md#programmatic-access-for-ai-agents-public-mcp).

## Next

- [Writing](build/writing.md): the same transactions, built by hand.
- [Deployments](deployments.md): what is on the testnet.
