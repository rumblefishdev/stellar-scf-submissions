# Technical Architecture: X402 Facilitator with Bazaar Discovery Support for Stellar

*Revised 2026-10-12: incorporates reviewer-requested detail on Bazaar discovery-layer specifics,
search ranking and quality evaluation, Stellar-specific facilitator behavior, and the conformance
testing approach.*

## 1. Overview

This document describes the technical architecture for a production x402 payment facilitator on
Stellar, extended with a Bazaar discovery layer, an MCP-native agent interface, and a usage-based
(`upto`) settlement scheme. It is written to be read independently of any grant application: it
describes what will be built and why each design decision was made, referencing the x402
specification and Stellar/Soroban's authorization model throughout.

x402 turns HTTP's `402 Payment Required` status code into a machine-native payment handshake: a
client requests a resource, the server names its price, the client authorizes payment, and a
facilitator verifies and settles that payment on-chain. Stellar's low settlement cost
(~0.0023 XLM per transaction) makes per-request micropayments practical at a scale most chains
cannot match. Stellar already has a working facilitator and an `@x402/stellar` package covering
the `exact` payment scheme (fixed-price payments). This document covers everything needed to turn
that into a full, discoverable, agent-native payment marketplace.

## 2. System Architecture

Four logical components, deployed as extensions of a single facilitator service rather than
separate microservices — following the same pattern used by the existing x402 reference
implementation (Coinbase's CDP Facilitator), where `/verify`, `/settle`, `/supported`,
`/discovery/resources`, and `/discovery/search` all live behind one service:

```
                     ┌───────────────────────────────────────────┐
                     │              Facilitator                  │
                     │                                           │
  Resource  ───402──▶│  /verify    /settle    /supported         │
  Server    ◀────────│                                           │
                     │  ┌─────────────────────────────────────┐  │
                     │  │  Bazaar Discovery Layer             |  │
                     │  │  /discovery/resources               |  │
                     │  │  /discovery/search                  |  │
                     │  │  (cataloging triggered on settle)   |  │
                     │  └─────────────────────────────────────┘  │
                     └───────────────┬───────────────────────────┘
                                      │
                              ┌───────┴────────┐
                              │  MCP Discovery │
                              │  Server        │
                              └───────┬────────┘
                                      │
                                 Agent Runtime
```

The facilitator is the trust anchor for both payment settlement and service discovery: discovery
entries are created as a side effect of real settlements, not a separate registration process,
which keeps the catalog aligned with services that actually work and actually get paid.

## 3. Facilitator

### 3.1 Core endpoints

Standard x402 v2 surfaces: `verify`, `settle`, `supported`, on both `stellar:testnet` and
`stellar:pubnet`. Built on the existing Apache-2.0 `@x402/stellar` package rather than
reimplementing verify/settle logic.

Payment flow for the `exact` scheme (fixed-price payments):

1. Client requests a resource; server responds `402` with `PaymentRequired`, naming price, asset,
   and recipient.
2. Client builds a Soroban `invokeHostFunction` transaction calling the SEP-41 token's
   `transfer(from, to, amount)`, simulates it to discover required authorization entries, and
   signs **only those authorization entries** (`sorobanCredentialsAddress` credential type) — not
   a full transaction. Expiry is a ledger number (`signatureExpirationLedger`), computed as
   `currentLedger + ceil(maxTimeoutSeconds / estimatedLedgerSeconds)`.
3. The facilitator verifies transaction structure (exactly one `invokeHostFunction` operation,
   correct function name and arguments, no facilitator-owned addresses anywhere in the auth
   entries), re-simulates against current ledger state, and confirms the resulting balance changes
   match exactly what's expected — nothing more.
4. At settlement, the facilitator rebuilds the transaction with **its own account as source**
   (see §3.2), copies the client's operations and auth entries unchanged, signs, and submits.

Any SEP-41 token is accepted, with USDC as the default. The facilitator is non-custodial by
design — funds move directly from payer to payee inside the signed contract invocation; the
facilitator never holds client funds.

### 3.1.1 Auth-entry validation, strict per field

Step 3 above is not a single check but seven independent ones, each validated before a payment is
accepted:

- **Signature** — cryptographically verified against the declared signer. Validation is
  signer-type-agnostic: a Soroban custom-account (smart wallet) signer exposing its own
  `__check_auth(context, signature_payloads)` entry point is verified through the network's own
  authorization/simulation flow, the same path a plain Ed25519-keypair account takes — the
  facilitator never hand-rolls signature checks that would only work for one signer type. The
  facilitator also accepts both the legacy Soroban address-credential format and CAP-71's V2
  (`SOROBAN_CREDENTIALS_ADDRESS_V2`) format: Stellar Protocol 28 (testnet since 2026-08-27, mainnet
  vote 2026-09-16) makes `stellar-sdk` v17 clients emit V2-signed auth entries by default, so
  accepting only the legacy format would silently reject a growing share of valid payments as
  adoption rolls out.
- **Declared call** — exactly one `invokeHostFunction` operation, function name and contract
  address matching what the payment terms specified, with no substitution.
- **Asset** — the SEP-41 token contract matches the asset named in `PaymentRequirements`.
- **Amount** — matches exactly for `exact`; for `upto` (§6), matches against the signed ceiling
  rather than a fixed amount.
- **Recipient** — the `to` argument matches the seller's `payTo` address.
- **Replay** — the native Soroban auth-entry nonce (or, for `upto`, the proxy contract's own
  nonce) has not been consumed before.
- **Expiry** — `signatureExpirationLedger` has not already passed at verify time, and is re-
  simulated against current ledger state rather than trusted as submitted; it is also capped
  against a configured maximum future window, closing the risk of a stale-but-not-yet-expired
  signature being replayed long after it was issued.

### 3.1.2 Trustline onboarding

Classic Stellar assets wrapped as SEP-41 contracts (as opposed to natively Soroban-issued tokens)
require the buyer to hold a trustline before they can receive or hold the asset. Pre-flight
simulation detects a missing trustline before payment is attempted and returns an actionable
error, and the buyer-side SDK surfaces this as an explicit "establish trustline" step rather than a
generic failure — preserving the goal of letting a developer deploy and test against a
discoverable endpoint in under an hour, even starting from a fresh wallet.

### 3.2 Fee sponsorship

Buyers never need to hold XLM. Because the client signs only authorization entries (not a full
transaction), the facilitator can wrap those entries into a transaction where **its own account is
the source** — paying the base network fee and supplying the sequence number — without touching
the payment amount itself. This is a meta-transaction pattern built on Stellar's native split
between "who authorizes a contract call" (auth entries) and "who is the transaction's source
account" (fee payer), requiring no special relayer contract.

Safety is enforced by verification, not trust: the facilitator's address must never appear as the
`from` address or in any client-signed auth entry, and re-simulation before submission must show
*only* the expected payer-decrease/recipient-increase balance changes. This guarantees the
facilitator can sponsor the fee without risk of being tricked into moving more than that.

### 3.3 Throughput: channel accounts

A single facilitator source account serializes on its own sequence number, becoming a bottleneck
under concurrent settlement load. The standard mitigation is a pool of channel accounts, each with
an independent sequence number, rotated across concurrent settlements.

This capability already exists in the ecosystem (OpenZeppelin's `relayer-plugin-channels`), but is
AGPL-3.0 licensed, which conflicts with a permissive-license requirement for facilitator
dependencies. Rather than depend on it — either directly, which would trigger AGPL's copyleft
obligations, or via OpenZeppelin's hosted channel-account service, which would add an operational
dependency on a third party's uptime for a core piece of settlement infrastructure — this design
**reimplements a channel-account pool under a permissive license**. The underlying pattern is a
well-documented Stellar technique, not proprietary to any one implementation, so this is
reimplementing known behavior rather than novel design work. The full dependency tree is re-audited
at each delivery milestone, not just checked once at the start, to keep this guarantee intact as
the codebase grows.

## 4. Bazaar Discovery Layer

### 4.1 Cataloging mechanism

A seller declares a resource as discoverable by including a `bazaar` extension block in their
route's `PaymentRequired` response:

```json
"extensions": {
  "bazaar": {
    "info": {
      "input":  { "type": "http", "method": "GET", "queryParams": { "...": "..." } },
      "output": { "type": "json", "example": { "...": "..." } }
    }
  }
}
```

For MCP tools, `info.input.type` is `"mcp"`, with `toolName`, `inputSchema`, and `transport`
fields. Optional enrichment fields (`serviceName`, `tags`, `iconUrl`) are validated with a
**soft-drop** policy — malformed optional fields are silently discarded rather than rejecting the
whole listing; only structurally required fields (input/output type, absolute URLs) cause
rejection. `iconUrl` is restricted to absolute `https`/`http` URLs with no IP literals or loopback
addresses, as an explicit SSRF defense.

The client's signed `PaymentPayload` must echo the `bazaar` extension for cataloging to occur.
Cataloging is triggered on **settle**, not verify — a payment that only reaches verification never
gets indexed, by design, so the catalog only ever reflects services with a real, completed
transaction behind them.

Parameterized routes (e.g. `/users/[id]`) are normalized to a `routeTemplate` (`/users/:userId`)
so repeated calls to the same pattern consolidate into one catalog entry instead of spamming
one-per-argument-value. Before a `routeTemplate` is used as a catalog key, it is percent-decoded
and then re-encoded; a mismatch between that result and the original is rejected outright. This
closes a listing-spoofing path where a crafted percent-encoding could look distinct from an
existing legitimate entry's template while decoding to the same normalized path — otherwise able
to shadow, collide with, or be confused for another seller's entry in `GET /discovery/resources`
results. MCP tools have no HTTP route to normalize this way, so they are keyed instead by the
composite of `resource.url` and `input.toolName`, which together uniquely identify a tool entry.

The facilitator may report cataloging outcomes via a base64-encoded JSON
`EXTENSION-RESPONSES` header (`{"bazaar":{"status":"success"|"processing"|"rejected", ...}}`), but
this is best-effort — its absence carries no signal and should not be relied upon by clients.

### 4.2 Discovery API

- `GET /discovery/resources` — filtered catalog listing on six filters: `type`, `payTo`, `network`,
  `extensions`, `limit`, `offset`. Returns each resource's `accepts` terms, its
  `extensions.bazaar.info`, and `lastUpdated`.
- `GET /discovery/search` — natural-language query with ranked results (see §4.3), covering both
  HTTP endpoints and MCP tools. Pagination is cursor-based (an opaque `cursor` token, passed back
  on the next request) rather than offset-based, since the underlying ranked set can shift between
  queries as new settlements land. Every response also carries a `partialResults: boolean` field,
  `true` whenever the quality-score recompute pass (§4.3) hasn't finished for the full candidate
  set at query time, so a caller can distinguish "these are all the matches" from "more may still
  be indexing."

The index is offchain by default, consistent with keeping settlement (onchain, minimal, auditable)
separate from discovery (offchain, mutable, iterative).

### 4.3 Search ranking

Ranking combines three signal categories, following the pattern established by the existing x402
reference implementation:

1. **Retrieval relevance** — **hybrid lexical + dense retrieval, fused with Reciprocal Rank Fusion
   (RRF)**, run on the ClickHouse cluster already operated for the team's other Stellar services
   (Prices API, Block Explorer), so no new datastore is introduced for the Bazaar catalog.
   - *Lexical*: a token-frequency (TF-IDF-style) relevance score over each resource's
     `serviceName`, `tags`, and `info` description fields, computed via a materialized per-token
     document-frequency table with `ngrambf_v1`-indexed candidate filtering.
   - *Dense*: vector similarity (`cosineDistance()`, native on `Array(Float32)` columns) between
     the query embedding and each resource's description embedding, computed with a small
     self-hosted sentence-transformer model (e.g. `all-MiniLM-L6-v2`) — avoiding a live external
     inference API as a dependency for a facilitator that has to hit 99%+ uptime. At the catalog's
     expected scale (thousands of listings), an unindexed `ORDER BY cosineDistance(...) LIMIT k`
     scan is cheap on ClickHouse's columnar engine; a native vector index is a scaling path if the
     catalog later grows enough to need one.
2. **Objective quality score** — buyer reach (distinct buyers served in a trailing window),
   transaction volume, recency of activity, metadata completeness (penalizing placeholder
   descriptions), and a hosting penalty for resources on shared/tunneling infrastructure.
3. **Curation** — an optional manual boost for verified, health-checked resources.

New listings with no settlement history are exempt from any minimum-activity filter (so they're
not invisible on day one) but receive no quality-score boost until real usage accumulates. No
published, reusable open-source implementation of this ranking function exists anywhere in the
x402 ecosystem today (checked against both the Coinbase and GoPlausible/Algorand reference
implementations) — this scoring function is designed from first principles for Stellar, informed
by the signal categories above.

**Quality evaluation.** Ranking quality is measured by **nDCG@10** (normalized discounted
cumulative gain at rank 10) — appropriate here because result relevance is graded, not binary —
against a fixed, human-labeled set of 50 natural-language queries over 25 seeded resources, each
query-result pair graded 0–3 before any ranking code is evaluated against it, to avoid biasing the
labels toward whatever the algorithm already returns. Ranking must reach **nDCG@10 ≥ 0.75** against
this fixed set before it ships beyond curation-only mode; short of that bar, curated ordering is
used instead. The evaluation is re-run at every delivery milestone, then monthly for the first 6
months post-launch, since the discovery spec and its ranking signal set are both still evolving
under the x402 Foundation.

### 4.4 Catalog storage

Catalog entries live in a ClickHouse `ReplacingMergeTree` table, keyed on the normalized
`routeTemplate` (HTTP resources) or the `resource.url` + `input.toolName` pair (MCP tools) and
versioned by settlement timestamp — the engine's native "latest version wins" merge semantics give
current-state read behavior from an append-only insert stream, without needing in-place row
updates. This fits how the data actually arrives: the only writer to the catalog is a settlement
event (§4.1). The buyer-reach/volume/recency quality-score inputs (above) are themselves OLAP
aggregations — `COUNT(DISTINCT buyer)`, `COUNT(*)`, `MAX(timestamp)` over rolling windows — against
that same settlement-event log, a workload ClickHouse's columnar engine is well suited to.

## 5. MCP Discovery Server

A Model Context Protocol server exposing search and paid-call tools, so an agent runtime can
search the Bazaar and complete a payment natively as part of its normal tool-use loop — without an
intermediate HTTP client layer. Inputs and outputs are structured and deterministic, and errors
are surfaced as machine-readable codes rather than free-text messages, so agent runtimes can
handle failure paths (insufficient funds, expired authorization, resource not found) programmatically.

## 6. The `upto` Settlement Scheme

### 6.1 Problem

The `exact` scheme authorizes a single fixed amount — useful for pay-per-call pricing, but not for
metered use cases (LLM tokens generated, bytes transferred, compute consumed) where the true cost
is only known after the request completes. The `upto` scheme addresses this: the client authorizes
a **ceiling**, and the actual settlement amount — determined by the seller after the request — must
be less than or equal to that ceiling.

On EVM chains, `upto` is built on Uniswap's Permit2 witness pattern rather than reusing `exact`'s
EIP-3009 mechanism, because EIP-3009 hard-codes the exact amount into the signed payload — there is
no way to authorize a ceiling and settle for less with it. Stellar's `exact` scheme has the
identical limitation: Soroban's `require_auth()` binds a signature to the literal invocation being
authorized (contract, function, and argument values), so a signed `transfer(from, to, amount)` call
only ever authorizes that one amount.

### 6.2 Design

Soroban's `Address::require_auth_for_args(args)` provides the same escape hatch Permit2's witness
pattern provides on EVM: a contract can authorize against a **custom argument list of its own
choosing**, decoupled from the literal invocation's parameters. The contract and function are still
pinned to the real call site (a client can't be tricked into signing for a different contract), but
the values being signed can be a purpose-built payload rather than the function's actual arguments.

The design combines this with Soroban's existing allowance primitive:

1. A canonical `UptoProxy` contract exposes
   `settle_upto(from, to, facilitator, max_amount, actual_amount, nonce, valid_after, deadline)`.
   Internally it calls
   `from.require_auth_for_args(vec![to, facilitator, max_amount, nonce, valid_after, deadline])` —
   deliberately excluding `actual_amount` from the signed payload. The client signs only the
   ceiling, recipient, nonce, and validity window; the actual settlement amount is a runtime
   parameter, checked in contract logic against the signed ceiling (`actual_amount <= max_amount`),
   never part of the signature itself.
2. Separately, the client grants the proxy contract a SEP-41 allowance —
   `approve(from, spender=ProxyAddress, amount=max_amount, expiration_ledger)` — itself only ever a
   ceiling, consistent with the "sign a max, not an exact amount" property required throughout.
3. At settlement, the proxy — itself the `spender` on that allowance — calls
   `token.transfer_from(spender=ProxyAddress, from, to, actual_amount)`. Because the proxy is
   calling this as itself, Soroban's rule that a contract's own calls on its own address
   auto-satisfy authorization (no signature required for a contract authorizing its own direct
   action) means no second client signature is needed for this inner transfer.

This satisfies all five properties the x402 specification requires of an `upto` implementation:

| Property | How this design satisfies it |
|---|---|
| Single-use authorization | Nonce consumed on settlement; rechecked against `valid_after`/`deadline` |
| Time-bound | `valid_after`/`deadline` are part of the signed payload |
| Recipient binding | `to` is part of the signed payload — cannot be redirected post-signature |
| Maximum enforcement | `actual_amount <= max_amount` checked in contract logic before transfer |
| Phase-dependent settlement | `max_amount` fixed at signing; `actual_amount` supplied only at settlement |

### 6.3 Upstream contribution

This design will be proposed as `scheme_upto_stellar.md`, alongside the existing
`scheme_upto_evm.md` and `scheme_upto_svm.md`, coordinated through the x402 Technical Steering
Committee — Stellar's representation on the x402 Foundation's Governing Board provides a direct
coordination channel for this. `batch-settlement` and `auth-capture` variants are explicitly out
of scope for this proposal, consistent with how the base `upto` specification scopes
multi-settlement/streaming patterns out of its first version.

### 6.4 Post-grant spec tracking

The discovery and settlement-scheme specs are both still evolving under the x402 Foundation, so
upkeep matters as much as the initial implementation. For the first 6 months after mainnet launch,
upstream spec changes (the `x402-foundation/x402` repo's `specs/` directory and Discussions
category) are reviewed on a monthly cadence, with a compatibility patch filed within 2 weeks of any
breaking change — run alongside the same-cadence ranking-quality re-measurement in §4.3. Changes
and resulting patches are logged in the project's own public changelog.

## 7. SDKs

**TypeScript** — thin ergonomic wrappers over the facilitator, Bazaar, and `upto` scheme, following
the existing `@x402/stellar` ecosystem conventions (seller middleware, buyer/agent helpers).

**Rust** — a net-new Stellar scheme plugin for the Rust x402 ecosystem. No existing Rust crate
(`rust-x402`, `x402-axum`) implements Stellar support; both are limited to Base/Avalanche on
EIP-3009. This crate implements the Soroban auth-entry construction and signing logic `@x402/stellar`
provides for TypeScript, targeting Rust web frameworks via `rust-x402`'s or `x402-axum`'s modular
scheme architecture where possible. Stellar's own canonical SDKs (`soroban-sdk`, `stellar-xdr`) are
Rust-first, which should make this implementation a more direct fit than the JavaScript port that
`@x402/stellar` had to build on.

## 8. Licensing

The facilitator, Bazaar, MCP server, and SDKs are released under a permissive OSI-approved license
with license-compatible dependencies — no AGPL, consistent with the requirement to preserve
ecosystem-wide reuse without copyleft obligations. This directly shapes the channel-account
implementation decision in §3.3, since the one existing reference implementation of that pattern is
AGPL-licensed.

## 9. Security Considerations

- **Facilitator self-protection.** The facilitator never signs for or holds client funds; its only
  exposure as fee-sponsor is the network fee itself, bounded by the verification checks in §3.2.
- **`upto` proxy contract** is new, first-of-its-kind code and requires independent security review
  before mainnet deployment, in addition to the facilitator's own audit.
- **Conformance testing** is performed at the wire level against an unmodified canonical client
  from the `x402-foundation/x402` repo's own end-to-end test suite, not just internal unit tests,
  to catch protocol-level divergence from the spec rather than only implementation bugs. Stellar's
  `exact` scheme is already a first-class mechanism in that suite's client, server, and facilitator
  roles, so this does not require contributing new Stellar test fixtures to the harness; it does
  require standing up a thin **external facilitator proxy** — a mechanism the suite already
  documents (`e2e/facilitators/external-proxies/`) for pointing the canonical client at a real,
  externally-hosted facilitator rather than one of its own bundled reference implementations —
  configured against this facilitator's testnet deployment. A passing run produces a real settled
  transaction hash per scheme, recorded as the conformance artifact.
- **SSRF defenses** are built into Bazaar metadata validation (§4.1) for any user-supplied URLs.

## References

- [x402 Specification v2](https://github.com/x402-foundation/x402/blob/main/specs/x402-specification-v2.md) —
  canonical repo is now under the x402 Foundation (`x402-foundation/x402`); `coinbase/x402` is a
  lagging fork
- [`scheme_exact_stellar.md`](https://github.com/x402-foundation/x402/blob/main/specs/schemes/exact/scheme_exact_stellar.md)
- [`scheme_upto.md`](https://github.com/x402-foundation/x402/blob/main/specs/schemes/upto/scheme_upto.md) /
  [`scheme_upto_evm.md`](https://github.com/x402-foundation/x402/blob/main/specs/schemes/upto/scheme_upto_evm.md) /
  [`scheme_upto_svm.md`](https://github.com/x402-foundation/x402/blob/main/specs/schemes/upto/scheme_upto_svm.md) —
  no `scheme_upto_stellar.md` exists yet
- [`@x402/stellar` package](https://github.com/x402-foundation/x402/tree/main/typescript/packages/mechanisms/stellar) —
  existing TypeScript client/server/facilitator implementation this design builds on; its
  changelog documents the CAP-71 V2 credential patch referenced in §3.1.1
- [E2E test suite / external facilitator proxies](https://github.com/x402-foundation/x402/tree/main/e2e) —
  canonical conformance harness referenced in §9
- [Bazaar (Discovery Layer) specification](https://docs.x402.org/extensions/bazaar)
- [Stellar Docs: Authorization (`require_auth_for_args`)](https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization)
- [SEP-0041: Stellar Token Interface](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
- [OpenZeppelin `relayer-plugin-channels`](https://github.com/OpenZeppelin/relayer-plugin-channels)
