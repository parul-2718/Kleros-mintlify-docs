---
name: kleros-docs
description: >
  Answer questions about integrating Kleros V2 dispute resolution into smart contracts.
  Use when a developer asks about IArbitrableV2, IArbitratorV2, createDispute, extraData
  encoding, dispute templates, data mappings, KlerosCore addresses, Curate V2, Escrow V2,
  Proof of Humanity V2, Reality V2, Vea cross-chain bridge, or the @kleros/kleros-sdk.
license: MIT
compatibility: Designed for Claude Code and other AI coding assistants working on Kleros integrations
metadata:
  author: kleros
  version: "1.0"
  site: https://docs.kleros.io
---

# Kleros Documentation — AI Context

You are helping with the Kleros developer documentation. Kleros is a decentralized dispute resolution protocol built on Ethereum. The current live version is **Kleros V2 (Neo)**, deployed on **Arbitrum One**.

## Core Concepts

**Kleros Court (V2):** Disputes are resolved by randomly selected jurors who stake PNK tokens. Jurors vote on binary or multi-option questions. Appeals are possible and each round doubles juror count.

**KlerosCore:** The main V2 contract at `0x9C1dA9A04925bDfDedf0f6421bC7EEa8305F9002` (Arbitrum One). It implements `IArbitrator` and manages courts, staking, and rulings.

**IArbitrableV2:** The interface your contract must implement. Key function: `rule(uint256 _disputeID, uint256 _ruling)`. Key event: `DisputeRequest(IArbitratorV2 _arbitrator, uint256 _arbitratorDisputeID, uint256 _externalDisputeID, uint256 _templateId, string _templateUri)`.

**extraData encoding (V2):** `abi.encodePacked(uint96(courtID), uint256(minJurors))` — court ID MUST be `uint96`, not `uint256`. Using `uint256` for court ID was V1 behavior and will encode a different court in V2.

**Ruling 0:** Always means "Refuse to Arbitrate / Invalid". Reserved by the protocol. Your contract must handle it explicitly. Tied votes also default to ruling 0.

**DisputeTemplateRegistry:** Stores dispute templates (what jurors see) and data mappings (how `{{variables}}` in templates are populated from on-chain/IPFS data). Call `setDisputeTemplate()` to register; emit the returned `templateId` in `DisputeRequest`.

**Data mappings types:** `graphql`, `fetch/ipfs/json`, `abi/call`, `abi/event`, `json`. Mappings are resolved sequentially; later ones can use variables from earlier ones.

**extraEvidences:** A dispute template field (`z.array(EvidenceSchema)`) that lets arbitrables surface pre-dispute evidence (requester submissions, challenger objections) directly in the Court UI. Uses Handlebars conditionals in the template.

## Products

| Product | Chain | Key Contract |
|---------|-------|-------------|
| Kleros Court V2 | Arbitrum One | `0x9C1dA9A04925bDfDedf0f6421bC7EEa8305F9002` |
| Escrow V2 | Arbitrum One | see deployments repo |
| Curate V2 | Arbitrum One + Arbitrum Sepolia testnet | see deployments repo |
| Proof of Humanity V2 | Gnosis Chain (home) + cross-chain | see PoH repo |
| Reality V2 | Arbitrum One | uses KlerosCore as arbitrator |

## Curate V2 Specifics

- Registration and removal disputes use **separate** templates with **inverted** answer logic
- `arbitrationParamsIndex` snapshots arbitration params at request time — always read params at the correct index
- Item data format uses column-based JSON schema: `{"columns": [{"label": "...", "type": "..."}], "values": [...]}`
- Challenge period: registration = challenge period; removal = removal challenge period (different durations)

## Kleros Registries

Four live curation registries (not all are subgraph-queryable):
- **Address Tags** — maps `address+chain` → tag string; Envio subgraph available
- **Tokens** — ERC-20 token metadata; Envio subgraph available
- **CDN (Contract Domain Names)** — maps contract address → domain; Envio subgraph available
- **ATQ (Address Tag Query)** — meta-registry of NPM packages; batch-only, NOT subgraph-queryable

## Cross-chain (Vea)

Vea bridges messages from Arbitrum → other chains. Pattern: `SenderGateway → VeaInbox → [bridge] → VeaOutbox → ReceiverGateway`. Goerli-based routes are deprecated (Sept 2024); use Sepolia equivalents. Track status at [veascan.io](https://veascan.io).

## Proof of Humanity V2

- Lives on Gnosis Chain. Cross-chain state via AMB bridge (minutes to hours, up to 24h delay)
- `humanityId` is `bytes20` — store user data by humanity ID, not address
- V1 users: `humanityId == bytes20(address)` (equality check detects legacy)
- `isHuman()` checks both V2 native and V1 legacy through Fork Module

## Audience Routing

Before answering, identify the user type and route accordingly:

| User asks about... | Send to |
|--------------------|---------|
| Staking PNK, becoming a juror, voting | `/court/how-it-works` and `court.kleros.io` |
| Integrating Kleros into a smart contract | `/developers/arbitrable-apps/arbitrable-guide` |
| Curate / registry integration | `/developers/products/curate/` |
| Escrow integration | `/developers/products/escrow/` |
| Proof of Humanity | `/developers/products/poh/` |
| Reality.eth + Kleros | `/developers/products/reality/` |
| Cross-chain arbitration | `/developers/crosschain/vea-bridge` |
| Contract addresses | `/reference/contracts/deployment-addresses` + always verify against `github.com/kleros/kleros-v2/tree/dev/contracts/deployments` |
| Dispute template format | `/reference/data-formats/dispute-templates` |
| SDK usage | `/reference/sdk/kleros-sdk` |
| V1 / legacy / ERC-792 | `/legacy/` — remind user V2 is current |

## Critical Accuracy Constraints

These must never be wrong in any code you generate or review:

1. **`extraData` encoding:** `abi.encodePacked(uint96(courtID), uint256(minJurors))` — court ID is `uint96`, NOT `uint256`. Getting this wrong silently routes to the wrong court.
2. **Ruling 0:** Always reserved for "Refuse to Arbitrate". Every `rule()` implementation must handle `_ruling == 0` explicitly.
3. **Fee fetching:** Never hardcode `msg.value` for arbitration. Always call `arbitrator.arbitrationCost(extraData)` immediately before `createDispute()`.
4. **Juror count:** Always odd (1, 3, 5...). Even numbers cause ties → ruling 0.
5. **ETH transfer:** Use `.call{value:...}("")` with `require(success)`. Never `transfer()` or `send()`.
6. **Contract addresses:** Always direct users to verify from the official deployment JSONs at `github.com/kleros/kleros-v2/tree/dev/contracts/deployments`. Never assert an address is correct without that caveat.
7. **ERC-20 fees:** Only work on KlerosCore directly. Not supported via ForeignGateway.
8. **Testnet:** Arbitrum Sepolia only. Arbitrum Goerli was deprecated September 2024.

- Using `uint256` instead of `uint96` for court ID in `abi.encodePacked`
- Hardcoding arbitration fees (they change via governance)
- Not handling ruling 0 explicitly
- Using even number of jurors (causes ties → ruling 0)
- Using `transfer()` instead of `.call{value:...}("")` for ETH transfers
- Calling `createDispute()` with stale fees (always fetch fresh `arbitrationCost()`)
- ERC-20 fee payment not supported via ForeignGateway (only KlerosCore directly)
- Not pinning IPFS content (disappears without pinning)
- Using testnet addresses on mainnet

## Key Links

- Deployments: `https://github.com/kleros/kleros-v2/tree/dev/contracts/deployments`
- Court UI: `https://court.kleros.io`
- Discord: `https://discord.gg/kleros`
- Integration contact: `integrations@kleros.io`

## Documentation Structure

```
/developers/
  quickstart.mdx              — First integration steps
  arbitrable-apps/
    arbitrable-guide.mdx      — IArbitrableV2 implementation guide
    arbitrable-production.mdx — Pre-deployment checklist
  products/
    curate/                   — Curate V2 integration + registry system
    reality/                  — Reality.eth + Kleros
    poh/                      — Proof of Humanity V2
  crosschain/
    vea-bridge.mdx            — Vea architecture
    vea-getting-started.mdx   — Vea integration guide
  subgraph/                   — TheGraph + subgraph queries
  examples/                   — Full code examples
/reference/
  contracts/                  — Contract ABIs and interfaces
  data-formats/               — Dispute templates, policy format
  sdk/                        — @kleros/kleros-sdk
```
