# Technical Architecture: X402 Facilitator with Bazaar Discovery Support for Stellar

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
dependencies. Two compliant paths: (a) call an externally-hosted channel-account service over the
network rather than linking its code directly, which does not trigger AGPL's copyleft obligations,
or (b) implement a permissively-licensed channel-account pool directly — the underlying pattern is
a well-documented Stellar technique, not proprietary to any one implementation.

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
one-per-argument-value. The facilitator may report cataloging outcomes via a base64-encoded JSON
`EXTENSION-RESPONSES` header (`{"bazaar":{"status":"success"|"processing"|"rejected", ...}}`), but
this is best-effort — its absence carries no signal and should not be relied upon by clients.

### 4.2 Discovery API

- `GET /discovery/resources` — filtered catalog listing (`type`, `payTo`, `network`, `extensions`,
  `limit`, `offset`). Returns each resource's `accepts` terms, its `extensions.bazaar.info`, and
  `lastUpdated`.
- `GET /discovery/search` — natural-language query with ranked results (see §4.3), covering both
  HTTP endpoints and MCP tools.

The index is offchain by default, consistent with keeping settlement (onchain, minimal, auditable)
separate from discovery (offchain, mutable, iterative).

### 4.3 Search ranking

Ranking combines three signal categories, following the pattern established by the existing x402
reference implementation:

1. **Retrieval relevance** — hybrid text/semantic match against the query.
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
`scheme_upto_evm.md`, coordinated through the x402 Technical Steering Committee — Stellar's
representation on the x402 Foundation's Governing Board provides a direct coordination channel for
this. `batch-settlement` and `auth-capture` variants are explicitly out of scope for this proposal,
consistent with how the base `upto` specification scopes multi-settlement/streaming patterns out of
its first version.

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
- **Conformance testing** is performed at the wire level against unmodified canonical x402 clients,
  not just internal unit tests, to catch protocol-level divergence from the spec rather than only
  implementation bugs.
- **SSRF defenses** are built into Bazaar metadata validation (§4.1) for any user-supplied URLs.

## References

- [x402 Specification v2](https://github.com/coinbase/x402/blob/main/specs/x402-specification-v2.md)
- [`scheme_exact_stellar.md`](https://github.com/coinbase/x402/blob/main/specs/schemes/exact/scheme_exact_stellar.md)
- [`scheme_upto.md`](https://github.com/coinbase/x402/blob/main/specs/schemes/upto/scheme_upto.md) /
  [`scheme_upto_evm.md`](https://github.com/coinbase/x402/blob/main/specs/schemes/upto/scheme_upto_evm.md)
- [Bazaar (Discovery Layer) specification](https://docs.x402.org/extensions/bazaar)
- [Stellar Docs: Authorization (`require_auth_for_args`)](https://developers.stellar.org/docs/learn/fundamentals/contract-development/authorization)
- [SEP-0041: Stellar Token Interface](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
- [OpenZeppelin `relayer-plugin-channels`](https://github.com/OpenZeppelin/relayer-plugin-channels)
