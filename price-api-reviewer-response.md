# Response to Reviewer Feedback — Prices API Design

This document addresses each of the four requested revisions raised by the reviewer.

---

## 1. Replacing Horizon API + Soroban RPC with Direct Ledger Processing

### The Problem

Our original design used Horizon's `trade_aggregations` endpoint as the primary source of SDEX trade data, and Soroban RPC event streaming for SEP-41/AMM activity. The reviewer correctly notes that Horizon is deprecated — the SDF is moving away from it in favor of the Composable Data Platform (CDP) tooling. Relying on Horizon creates a single point of failure and a scaling ceiling tied to the SDF's own Horizon operation.

### New Architecture: Galexie → S3 → Lambda Processor

We will replace the Horizon and Soroban RPC ingestion lanes with a **self-hosted Galexie process** running on **ECS Fargate**, writing `LedgerCloseMeta` files to our own **S3 bucket** in the same AWS account. A Lambda function processes each file as it arrives.

```
Stellar Network (mainnet peers)
        │
        ▼ (Captive Core / ledger stream)
┌──────────────────────────────────┐
│  Galexie — ECS Fargate (1 task)  │
│  Continuously running            │
│  Exports one file per ledger     │
│  (~1 file every 5–6 seconds)     │
└──────────────┬───────────────────┘
               │ LedgerCloseMeta XDR (compressed)
               ▼
┌──────────────────────────────────┐
│  S3 Bucket: stellar-ledger-data/ │
│  path: ledgers/{seq_start}-      │
│              {seq_end}.xdr.zstd  │
└──────────────┬───────────────────┘
               │ S3 PutObject event notification
               ▼
┌──────────────────────────────────────────────────────┐
│  Lambda "Ledger Processor"                           │
│  Triggered per file (~every ledger close)            │
│  1. Download + decompress XDR                        │
│  2. Parse LedgerCloseMeta via @stellar/stellar-sdk   │
│  3. Extract SDEX trades from OperationResult:        │
│     ManageSellOfferResult / ManageBuyOfferResult     │
│     → offersClaimed[] (price, volume, asset pair)   │
│  4. Extract Soroban swap events (CAP-67):            │
│     Soroswap, Aquarius, Phoenix — all in one stream  │
│  5. Write 1-min OHLCV candles to RDS                 │
└──────────────────────────────────────────────────────┘
               │
               ▼
       RDS PostgreSQL (unchanged)
```

### What This Eliminates

| Removed dependency | Replaced by |
|--------------------|-------------|
| Horizon `trade_aggregations` | Ledger Processor parsing SDEX offer results from `LedgerCloseMeta` |
| Soroban RPC event streaming (Soroswap/Aquarius/Phoenix swaps) | CAP-67 Soroban events from same `LedgerCloseMeta` stream |
| Soroswap API (for price/trade data) | Ledger events (API still used for pool metadata/discovery only) |
| Aquarius API (for price/trade data) | Ledger events (API still used for pool metadata/discovery only) |

### What External APIs Remain (and Why)

| External dependency | Purpose | Replaceability |
|--------------------|---------|----------------|
| Reflector Oracle (Soroban RPC `simulateTransaction`) | Cross-reference price sanity check only — **not a primary source** | Could switch to any other oracle |
| Soroswap API | Pool metadata and pair discovery | Low-frequency, non-critical |
| Aquarius API | Pool metadata and pair discovery | Low-frequency, non-critical |
| Stellar network peers | Galexie Captive Core upstream | Inherent to any Stellar node |

### Galexie Operation

- **Runtime:** ECS Fargate task, 1 vCPU / 2 GB RAM, continuously running
- **Recovery:** If the task restarts, Galexie resumes from the last exported ledger sequence (checkpoint-aware)
- **Lag monitoring:** CloudWatch alarm fires if S3 file timestamps fall more than 60 seconds behind current ledger close time
- **Data format:** `LedgerCloseMeta` XDR, zstd-compressed, parsed via `@stellar/stellar-sdk` XDR types in Node.js Lambda

### Why This Is More Robust

- No dependency on SDF-operated Horizon availability or its rate limits
- All price data flows from the canonical ledger — same source used by Stellar Core itself
- Single ingestion stream covers SDEX + all Soroban AMMs simultaneously
- Aligns with SDF's stated direction for ecosystem data infrastructure

---

## 2. Component Ownership — What We Host vs. External Dependencies

The following table demarcates every component in the system by who owns and operates it.

### Components Hosted by Rumble Fish (in AWS sub-account)

| Component | Service | Role |
|-----------|---------|------|
| Galexie process | ECS Fargate (1 task, continuous) | Streams ledger data from Stellar network to S3 |
| S3 bucket `stellar-ledger-data` | AWS S3 | Stores `LedgerCloseMeta` XDR files from Galexie |
| Lambda — Ledger Processor | AWS Lambda (event-driven, per S3 file) | Parses ledger XDR, extracts trades and Soroban events, writes OHLCV candles to RDS |
| Lambda — Oracle Fetcher | AWS Lambda (EventBridge, every 5 min) | Reads Reflector oracle prices as cross-reference; writes to `oracle_prices` table |
| Lambda — OHLCV Rollup | AWS Lambda (EventBridge, every 15 min) | Rolls up 1m candles → 15m / 1h / 4h / 1d |
| Lambda — Asset Discovery | AWS Lambda (EventBridge, every 1 hour) | Scans ledger contract deployment events for new SEP-41 tokens; classic assets from account entries in `LedgerCloseMeta` (issuer account, `homeDomain` field, trustlines) |
| Lambda — Current Price Updater | AWS Lambda (EventBridge, every 1 min) | Reads latest candles from `price_ohlcv`, computes weighted average across sources, writes to `current_prices` table |
| Lambda — Cleanup Worker | AWS Lambda (EventBridge, daily 02:00 UTC) | Retention policy: deletes fine-grained data, drops old partitions, creates new ones |
| Lambda — API handlers | AWS Lambda (per API Gateway route group) | Serves public API requests |
| RDS PostgreSQL | AWS RDS (db.t4g.micro → upgradeable) | Primary data store |
| API Gateway | AWS API Gateway | Public REST API entry point, caching, rate limiting, API keys |
| EventBridge Scheduler | AWS EventBridge | Cron/rate triggers for scheduled Lambdas |
| S3 bucket `api-docs` | AWS S3 + CloudFront | Hosts OpenAPI spec and self-service onboarding portal |
| Secrets Manager | AWS Secrets Manager | DB credentials, oracle/API keys |
| CloudWatch + X-Ray | AWS CloudWatch | Logs, metrics, alarms, distributed tracing |
| CI/CD pipeline | GitHub Actions → AWS CDK | Infrastructure-as-code deploy from open-source repo |

### External Services Consumed (read-only, no data written)

| External service | Data used for | Failure impact |
|-----------------|--------------|----------------|
| Stellar network peers (via Galexie Captive Core) | Ledger data source | Critical — Galexie cannot ingest without network access; mitigated by connecting to multiple peers |
| Reflector Oracle (Soroban RPC) | Price cross-reference sanity check only | Non-critical — API continues serving aggregated prices; oracle column shows `null` or last known value |
| Soroswap API | Pool pair metadata and discovery | Non-critical — existing pairs continue working; new pair discovery pauses until restored |
| Aquarius API | Pool pair metadata and discovery | Non-critical — same as above |

### What Stellar / SDF Provides

| Item | Nature |
|------|--------|
| AWS sub-account (via infrastructure grant) | Account for Rumble Fish to deploy into |
| Stellar mainnet network access | Public — Galexie connects as any node would |
| Open-source re-deployability | Full CDK stack is public; Stellar can fork and redeploy independently |

---

## 3. Blend Exploit Analysis and Price Manipulation Defense Strategy

### What Happened (February 2026)

The Blend YieldBlox pool suffered a ~$10.8M exploit. The attacker identified USTRY — a tokenized US Treasury bond issued by Etherfuse — as an asset with very thin SDEX liquidity, and executed trades that moved its price from ~$1.05 to $100+ (a 100x spike). Reflector oracle reads the SDEX mid-price every 5 minutes without time-weighting or liquidity gating, so the inflated price was published and treated as valid. The attacker then deposited USTRY as collateral and borrowed ~61M XLM and ~1M USDC against it.

Crucially, this was **not a flash loan attack**. The attacker held the manipulated SDEX position across two consecutive Reflector windows (5 minutes apart), waited 9+ minutes in total, and then executed the borrow. Real capital was at risk during the manipulation window — which points to just how thin USTRY's market was.

### The Core Issue: Liquidation Feasibility Was Never Checked

The reviewer asks us to address price manipulation robustly. We believe the most important insight from this exploit is that **oracle manipulation was a secondary symptom**. The primary failure was that **YieldBlox accepted millions of dollars of USTRY as collateral without verifying whether those millions could ever be recovered through liquidation**.

USTRY appears to have been traded exclusively on SDEX, with no meaningful liquidity pools on Soroswap or Aquarius. Its 24h volume at the time of the exploit was a small fraction of the total collateral position accepted. This created an impossible situation for the liquidator: even at the correct $1.05 price, liquidating a large USTRY position by selling it back on SDEX would crater the price far below $1.05, making the collateral worth far less than the borrowed amount.

A simple rule — **do not accept more collateral of a given asset than a fixed percentage of that asset's 24h trading volume** — would have blocked this attack regardless of what any oracle reported. If the daily volume of USTRY was, say, $20,000, then accepting $5M of USTRY collateral is indefensible at any price.

### What Our API Already Provides for This

Our design (as described in `prices-api-design.md`) already exposes the fields a lending protocol needs to enforce this rule. A protocol like YieldBlox should rely on the following from our `GET /assets/{asset_identifier}/price` response:

**`volume_24h_usd`** — the total USD-denominated trading volume for the asset across all tracked sources in the past 24 hours. This is the primary signal for liquidation feasibility. A protocol should cap total accepted collateral for any single asset at some percentage of this figure (e.g., 10–20% of `volume_24h_usd`).

**`vwap_24h`** — our volume-weighted average price over 24 hours. This is computed by our system as:
```
vwap_24h = Σ(source_price × source_volume_24h) / Σ(source_volume_24h)
```
across SDEX, Soroswap, and Aquarius. A 24h VWAP is structurally harder to manipulate than a 5-minute spot price: moving it meaningfully requires sustained volume over the full day, which costs real capital and leaves a visible trail in our historical OHLCV data. Protocols should use `vwap_24h` for collateral valuation rather than any spot price or short-window oracle reading.

**`sources`** — the per-source breakdown already in our design:
```json
{
  "price_usd": "1.05",
  "vwap_24h": "1.048",
  "volume_24h_usd": "18400.00",
  "sources": {
    "sdex":      { "price": "1.05", "volume_24h": "18400" },
    "soroswap":  null,
    "aquarius":  null
  },
  "updated_at": "2026-02-22T00:14:30Z"
}
```
The absence of Soroswap and Aquarius entries immediately tells a consuming protocol that SDEX is the only market. A protocol can use this to apply a stricter collateral cap, or to refuse the asset as collateral entirely if it is single-source.

**OHLCV data (`GET /assets/{asset_identifier}/ohlcv`)** — historical candles expose `trade_count` per period. A protocol can inspect whether the asset has consistent, organic trading activity over time versus sporadic or one-off bursts. Sudden high volume on a normally quiet asset is a red flag.

### Our Responsibility vs. Consuming Protocol's Responsibility

It is not the job of a price API to prevent lending protocols from making poor risk decisions. Our responsibility is to provide accurate, timely data with enough context for consuming systems to make informed decisions. The fields above — `volume_24h_usd`, `vwap_24h`, and the `sources` breakdown — are already in our design and directly address the information gap that enabled this exploit.

The responsibility for enforcing collateral caps, rejecting single-market assets, or requiring minimum liquidity before accepting a position lies entirely with the consuming protocol. We will make this explicit in our API documentation and onboarding guide, recommending that any protocol using our price data for collateral valuation implement a volume-based collateral cap as a minimum safeguard.

---

## 4. Expanded Deliverables with Concrete Acceptance Criteria

Each milestone below specifies what "done" means, making review and payment tranches unambiguous.

### Milestone 1 — Infrastructure Foundation (Weeks 1–2)

**Work:**
- AWS CDK stack provisioned: ECS Fargate cluster (Galexie), S3 buckets (ledger data, API docs), Lambda execution roles, API Gateway, RDS, EventBridge, CloudWatch, Secrets Manager
- RDS PostgreSQL running with full schema (all tables in Section 3 of the design doc, all partitions for current + next 2 months, all indexes)
- Galexie ECS Fargate task running and actively writing `LedgerCloseMeta` files to S3
- GitHub Actions CI/CD pipeline: lint → test → `cdk diff` → `cdk deploy` on merge to main
- CloudWatch alarm: Galexie lag > 60s triggers SNS notification (email / Slack webhook)

**Acceptance criteria (verifiable by reviewer):**
1. `cdk deploy` from a clean AWS account produces the full stack with no manual steps
2. S3 bucket `stellar-ledger-data/` contains at least 1,000 consecutive ledger files (covering ~90 minutes of mainnet history) with timestamps matching mainnet ledger close times
3. RDS schema matches the spec in `prices-api-design.md`: all tables exist, partitions exist for current and next 2 months, all indexes present (verifiable via `\d+` psql output)
4. CI/CD pipeline link provided; at least one successful deploy run visible in GitHub Actions

---

### Milestone 2 — Data Ingestion (Weeks 3–4)

**Work:**
- Ledger Processor Lambda: S3 event-triggered, parses `LedgerCloseMeta` XDR, extracts SDEX trades and Soroban swap events (Soroswap, Aquarius, Phoenix), writes 1-min OHLCV candles to `price_ohlcv`
- OHLCV Rollup Lambda: rolls 1m → 15m, 1h, 4h, 1d via EventBridge schedule (every 15 min)
- Asset Discovery Lambda: detects new SEP-41 contract deployments and classic assets from ledger; populates `assets` table
- Oracle Fetcher Lambda: reads Reflector prices every 5 min, writes to `oracle_prices`; failures log to CloudWatch but do not block other ingestion
- Current Price Updater Lambda: runs every 1 min via EventBridge, reads latest source candles from `price_ohlcv`, writes aggregated price to `current_prices` table (basic version — full VWAP formula wired in during Milestone 4)

**Acceptance criteria (verifiable by reviewer):**
1. After 24 hours of live operation: `price_ohlcv` contains continuous 1-min candles for at least 20 major assets (XLM, USDC, EURC, AQUA, yXLM, BTC, ETH and their SAC equivalents), covering the full 24h period with no gaps >2 candles
2. 15m, 1h, 4h, 1d rollup candles present and mathematically correct (high = max of constituent 1m highs, etc.) — verified via provided test script
3. `oracle_prices` table populated with Reflector readings every 5 min; at least 200 records per major asset (XLM, USDC, EURC) over the 24h period (Reflector at 5-min intervals yields ~288 per asset per day)
4. Simulated new asset deployment on testnet → appears in `assets` table within 1 hour
5. Processing lag: Lambda processes each S3 file within 10 seconds of the S3 PutObject event (CloudWatch `Duration` metric confirms)

---

### Milestone 3 — Public API v1 (Weeks 5–6)

**Work:**
- All core API endpoints implemented and deployed behind API Gateway:
  - `GET /assets` (paginated, sortable, filterable)
  - `GET /assets/{asset_identifier}` (single asset)
  - `GET /assets/{asset_identifier}/price` (current price with `sources` breakdown per the design doc)
  - `GET /assets/{asset_identifier}/ohlcv` (OHLCV with timeframe/granularity params)
- API Gateway: response caching (TTLs per endpoint: `/assets` 60s, `/ohlcv` 60s, `/price` 15s), usage plans, API key issuance, throttling
- Input validation: asset identifier format enforced, param ranges validated, 400 errors on invalid input

**Acceptance criteria (verifiable by reviewer):**
1. All 4 endpoint groups return correct, schema-valid responses for at least 20 major assets
2. Load test (k6 or Locust, script provided): 100 req/s sustained for 5 minutes on `GET /assets/{id}/price` → p95 latency <200ms, error rate <0.1%
3. Cache confirmed: consecutive identical requests within TTL window return `X-Cache: Hit` header from API Gateway
4. Invalid inputs (bad asset identifier, out-of-range dates, unknown granularity) return `400` with descriptive error message — not 500
5. API key required: requests without a key return `403`

---

### Milestone 4 — Oracle Integration, VWAP, and Polish (Weeks 7–8)

**Work:**
- Aquarius integration: Aquarius swap events already arrive via the Galexie ledger stream (same as Soroswap/Phoenix); this milestone validates Aquarius pool metadata from the Aquarius API and qualifies it as a named VWAP source
- VWAP calculation in Current Price Updater Lambda: implements the formula from the design doc — `Weighted Price = Σ(source_price × source_volume_24h) / Σ(source_volume_24h)` — across SDEX, Soroswap, Aquarius; sources below the configurable `min_volume_usd` threshold are excluded
- Outlier detection: before a source's price is included in the VWAP, it is compared against the inter-source median; sources deviating by more than a configurable percentage are excluded from that update cycle
- `POST /prices/batch` endpoint (multi-asset current price in one call)
- `GET /oracles/{asset_identifier}` endpoint (Reflector oracle cross-reference data; schema supports additional oracles)

**Acceptance criteria (verifiable by reviewer):**
1. Aquarius appearing as a named source in `price_usd` `sources` breakdown for assets that have Aquarius pools (e.g., AQUA/XLM)
2. VWAP calculation verifiable: for any asset, `price_usd` in `current_prices` matches `Σ(source_price × source_volume_24h) / Σ(source_volume_24h)` computed against the same raw rows — verified via provided SQL query
3. `?min_volume_usd=` query param: sources below the specified threshold absent from the `sources` breakdown and excluded from `price_usd`
4. Outlier exclusion: inject a test record where one source's price deviates >X% from the others → verify via DB that the source is excluded from that VWAP cycle
5. `POST /prices/batch` returns correct current prices for all requested assets in a single response
6. `GET /oracles/{asset}` returns Reflector price records for at least XLM, USDC, EURC; oracle data does not appear in the `price_usd` field

---

### Milestone 5 — Documentation, Testing, and Hardening (Week 9)

**Work:**
- OpenAPI 3.0 specification covering all endpoints with request/response schemas, error codes, field descriptions
- Self-service onboarding portal (hosted on S3 + CloudFront): API key request form, quickstart guide, example queries
- Integration test suite (automated, runs in CI): covers all 6 endpoint groups with real data assertions
- Load test report: k6 or Locust, documented test plan, results at 100/s, 500/s, 1000/s
- Partition manager Lambda: creates next 2 months of partitions on the 1st of each month

**Acceptance criteria (verifiable by reviewer):**
1. OpenAPI spec passes `openapi-validator` lint with no errors
2. Swagger UI deployed and accessible at documented URL; all endpoints callable from the UI
3. Onboarding portal accessible at documented URL; self-service API key request flow functional
4. Integration test suite: all tests pass on CI (GitHub Actions link provided). Tests cover: correct OHLCV math, pagination cursor correctness, cache TTL behavior, error handling
5. Load test report provided as PDF/Markdown: shows p50/p95/p99 at each tested load level; p95 <100ms at 100 req/s confirmed

---

### Milestone 6 — Production Launch (Week 10)

**Work:**
- Security review checklist completed (IAM least-privilege, no secrets in env vars, input sanitization, HTTPS-only, VPC isolation for RDS)
- X-Ray tracing enabled end-to-end (API Gateway → Lambda → RDS)
- CloudWatch dashboards: API latency, error rate, ingestion lag (Galexie S3 file freshness), DB CPU/connections, current price staleness per asset
- GitHub repository made public with README, architecture docs, local dev setup guide, and deploy instructions
- Public announcement and handoff documentation to Stellar team

**Acceptance criteria (verifiable by reviewer):**
1. Security checklist provided and signed off — minimum: Lambda functions have no wildcard IAM permissions, RDS has no public endpoint, all secrets in Secrets Manager (not environment variables), all inputs validated against regex/schema
2. X-Ray service map shows end-to-end traces for API Gateway → Lambda → RDS; latency breakdown visible
3. GitHub repository public; `cdk deploy` from README instructions works in a fresh AWS account (verified by reviewer or documented in video)
4. CloudWatch dashboard accessible to Stellar team (read-only IAM role provided); all alarms in OK state at launch
5. 7-day post-launch monitoring report provided: uptime %, error rate, p95 latency, Galexie ingestion lag

---

*This document supersedes the condensed milestone table in `notes/prices-api-design.md` (Section 9). The technical design sections of that document remain valid, updated for the Galexie ingestion architecture described in Section 1 above.*
