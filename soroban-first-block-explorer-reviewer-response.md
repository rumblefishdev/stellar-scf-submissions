# Response to Reviewer Feedback — Soroban-first Block Explorer

This document addresses each of the three requested revisions raised by the reviewer.

---

## 1. Replacing Horizon with Direct Ledger Processing (Galexie)

### The Problem

Our original design used Stellar Horizon as the indexing layer. The reviewer correctly
identifies that Horizon is deprecated — the SDF is moving toward the Composable Data
Platform (CDP) tooling. Horizon also has a structural gap for this specific RFP: its REST
API does not expose Soroban events in a queryable form; CAP-67 events live inside
`result_meta_xdr` and are not decoded into searchable fields. Relying on Horizon would
limit our event coverage and tie us to a deprecated system.

### New Architecture: Galexie → S3 → Lambda Processor → Own RDS

We will replace the Horizon-based design with a **self-hosted Galexie process** running on
**ECS Fargate**, writing `LedgerCloseMeta` XDR files to our own **S3 bucket**. A Lambda
function processes each file as it arrives, extracts all block explorer data, and writes it
to a **dedicated PostgreSQL database** that we own entirely.

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
               │ LedgerCloseMeta XDR (zstd-compressed)
               ▼
┌──────────────────────────────────┐
│  S3: stellar-ledger-data/        │
│  ledgers/{seq_start}-{seq_end}   │
│                    .xdr.zstd     │
└──────────────┬───────────────────┘
               │ S3 PutObject event notification
               ▼
┌─────────────────────────────────────────────────────────┐
│  Lambda "Ledger Processor"  (event-driven, per file)    │
│  1. Download + decompress XDR                           │
│  2. Parse LedgerCloseMeta via @stellar/stellar-sdk      │
│  3. Extract ledger header (sequence, close_at, proto)   │
│  4. Extract all transactions: hash, source, fee,        │
│     success/failure, envelope XDR, result XDR           │
│  5. Extract operations: type, details per operation     │
│  6. Extract Soroban invocations (INVOKE_HOST_FUNCTION): │
│     contract ID, function name, args, return value      │
│  7. Extract CAP-67 events (SorobanTransactionMeta       │
│     .events): all contract events in one stream         │
│  8. Extract contract deployments (new C-addresses)      │
│  9. Produce human-readable summaries for known          │
│     patterns (swaps, transfers, mints)                  │
│ 10. Write all above to RDS PostgreSQL                   │
└─────────────────────────────────────────────────────────┘
               │
               ▼
       ┌───────────────────┐
       │  RDS PostgreSQL   │  ← block explorer's own schema
       └─────────┬─────────┘
                 │
                 ▼
       ┌───────────────────┐     ┌─────────────────────┐
       │  Lambda — NestJS  │────>│  React Frontend      │
       │  REST API         │     │  (CloudFront CDN)    │
       └───────────────────┘     └─────────────────────┘
```

### What `LedgerCloseMeta` Contains

The `LedgerCloseMeta` XDR produced by Galexie contains the complete ledger close: the
ledger header, every transaction envelope and result, operation-level metadata, and the full
`SorobanTransactionMeta` for each Soroban transaction. Specifically:

| Data needed by block explorer | Where it lives in LedgerCloseMeta |
|-------------------------------|-----------------------------------|
| Ledger sequence, close time, protocol version | `LedgerHeader` |
| Transaction hash, source account, fee, success | `TransactionEnvelope` + `TransactionResult` |
| Operation type and details | `OperationMeta` per transaction |
| Soroban contract invocation (function, args, return value) | `InvokeHostFunctionOp` in envelope |
| CAP-67 contract events (type, topics, data) | `SorobanTransactionMeta.events` |
| Contract deployment (new C-address, WASM hash) | `LedgerEntryChanges` (CONTRACT type) |
| Account balance changes | `LedgerEntryChanges` (ACCOUNT type) |

Everything a block explorer needs is present in the XDR. There is no data gap requiring
Horizon or Soroban RPC to be a primary source.

### Historical Backfill

For historical data, Galexie can be pointed at Stellar's **public history archives** (the
same archives used by Horizon's `db reingest` command). This lets us process any ledger
range — including full history from genesis — by reading `LedgerCloseMeta` files from the
archives without running a full Horizon stack. We will backfill from at least the Soroban
mainnet activation ledger (late 2023) to give users a complete picture of all Soroban
activity.

The backfill process runs as a separate ECS Fargate task in batch mode, writing files to
the same S3 bucket, which triggers the same Ledger Processor Lambda. No separate code path
is required.

### Component Ownership

**Hosted by Rumble Fish (AWS sub-account):**

| Component | Service | Role |
|-----------|---------|------|
| Galexie process | ECS Fargate (1 task, continuous) | Streams ledger data from Stellar network to S3 |
| Historical backfill task | ECS Fargate (batch, runs once during phase 1) | Processes history archives into S3 |
| S3 bucket `stellar-ledger-data` | AWS S3 | Stores `LedgerCloseMeta` XDR files |
| Lambda — Ledger Processor | AWS Lambda (event-driven, per S3 file) | Parses XDR, writes all block explorer data to RDS |
| Lambda — Event Interpreter | AWS Lambda (EventBridge, every 5 min) | Post-processes new events to generate human-readable summaries |
| Lambda — NestJS API handlers | AWS Lambda (per API Gateway route) | Serves all public API requests |
| RDS PostgreSQL | AWS RDS | Block explorer database — ledgers, transactions, contracts, events |
| API Gateway | AWS API Gateway | REST API, rate limiting, API keys |
| CloudFront CDN | AWS CloudFront | Serves React frontend |
| EventBridge Scheduler | AWS EventBridge | Cron triggers for background workers |
| Secrets Manager | AWS Secrets Manager | DB credentials |
| CloudWatch + X-Ray | AWS CloudWatch | Logs, metrics, alarms, distributed tracing |
| CI/CD pipeline | GitHub Actions → AWS CDK | Infrastructure-as-code deploy |

**External services consumed (read-only):**

| External service | Purpose | Failure impact |
|-----------------|---------|----------------|
| Stellar network peers (Galexie Captive Core) | Live ledger data source | Critical — mitigated by connecting to multiple peers |
| Stellar public history archives | Historical backfill (one-time) | Non-critical after backfill completes |

No external APIs (Horizon, Soroswap, Aquarius, Soroban RPC) are required for the block
explorer's core functionality. All data flows from the canonical ledger.

### Why This Is More Robust

- **No deprecated software** — no Horizon dependency anywhere in the stack
- **No event coverage gap** — CAP-67 events are parsed natively from XDR; they are not
  filtered or decoded by an intermediary
- **Single ingestion stream** — ledger headers, transactions, operations, Soroban
  invocations, and events all come from the same `LedgerCloseMeta` file; no joins across
  different APIs
- **Self-contained recovery** — if Galexie restarts, it resumes from the last exported
  ledger sequence (checkpoint-aware); if the Ledger Processor fails, it replays from S3
- **Aligns with SDF's stated direction** — Galexie is the SDF-endorsed CDP exporter;
  using it positions this block explorer as a reference implementation for modern Stellar
  infrastructure

---

## 2. Budget Justification

### Rate Breakdown

Our $131,200 request covers direct development labour for a team of 4 developers and
1 designer over 12 weeks:

| Role | Rate | Hours | Total |
|------|------|-------|-------|
| 4 × Developer | $65/hr | 40 hrs/week × 12 weeks = 1,920 hrs each | $124,800 |
| 1 × Designer | $40/hr | 160 hrs (20 days) | $6,400 |
| **Total** | | | **$131,200** |

$65/hr is a standard rate for senior engineers at a specialised Web3 dev shop in Central
Europe. This covers engineers with Stellar/Soroban knowledge; general contractors without
domain expertise would require a longer ramp-up that would exceed this budget.

### Why the Scope Justifies the Budget

The revised Galexie-based architecture **adds engineering complexity** relative to a
Horizon-proxy approach — and that complexity is precisely what the reviewer is asking for:

| Component | Work involved |
|-----------|---------------|
| XDR parsing pipeline | Full `LedgerCloseMeta` deserialization using `@stellar/stellar-sdk` XDR types; extracting and normalising all 8 data types listed in Section 1 |
| Custom database schema | Owning the full schema for ledgers, transactions, operations, contracts, events, event_interpretations — not inheriting Horizon's schema |
| Historical backfill | Batch ECS task reading from history archives; gap detection; replay-safe Ledger Processor |
| Event interpretation | Heuristic logic to identify swap, transfer, mint, and burn patterns across known protocol contracts |
| Full-stack delivery | NestJS REST API + React SPA with all pages (Home, Transactions, Ledgers, Accounts, Contracts, Search) |
| Infrastructure as code | Full AWS CDK stack: ECS, Lambda, RDS, API Gateway, CloudFront, EventBridge, S3, CloudWatch |

A Horizon-only approach would have been simpler (and cheaper), but it would not deliver
the robust, maintainable system the RFP requires. The budget reflects the cost of building
infrastructure that does not depend on deprecated services.

### Infrastructure Costs

AWS hosting costs (~$188–$465/month depending on RDS size) are **not** included in this
budget and will be covered separately through the Stellar infrastructure grant process.

---

## 3. Custom Indexing, Maintenance, and Operations

### Operational Design of the Galexie Pipeline

**Normal operation (live):**

```
Galexie (ECS Fargate) → S3 (one file per ledger, ~every 5–6 s)
                      → Lambda Ledger Processor (triggered per S3 PutObject)
                      → RDS PostgreSQL (write, ~<10 s from ledger close to DB)
```

**Recovery from Galexie restart:** Galexie is checkpoint-aware. On restart it reads the
last exported ledger sequence from its local state file and resumes from there. No manual
intervention required.

**Recovery from Ledger Processor failure:** S3 PutObject event notifications are retried
by Lambda automatically. If a file was written to S3 but processing failed, the Lambda
retries up to the configured limit. For permanent failures, the file remains in S3 and can
be replayed manually by re-triggering the Lambda with the S3 key.

**Historical backfill (one-time, during Phase 1):** A second ECS Fargate task reads from
Stellar's public history archives in configurable ledger-range batches, writes the same
`LedgerCloseMeta` format to the same S3 bucket, and triggers the same Ledger Processor
Lambda. Backfill is fully parallelisable across ledger ranges.

**Schema migrations:** All schema changes are versioned and managed via AWS CDK custom
resources or a migration tool (e.g. `node-pg-migrate`). Migrations run as part of the
CI/CD pipeline before deploying new Lambda code.

### Monitoring and Alerting

| Alarm | Threshold | Action |
|-------|-----------|--------|
| Galexie ingestion lag | S3 file timestamp >60 s behind ledger close time | SNS → Slack/email |
| Ledger Processor error rate | >1% of Lambda invocations in error | SNS → Slack/email |
| RDS CPU | >70% sustained for 5 min | SNS → on-call; evaluate upgrade |
| RDS free storage | <20% remaining | SNS → expand storage or archive |
| API Gateway 5xx rate | >0.5% of requests | SNS → Slack/email |

CloudWatch dashboards will expose: Galexie S3 file freshness, Ledger Processor duration
and error rate, API latency (p50/p95/p99), RDS connections and CPU, and current highest
indexed ledger sequence vs. network tip.

### Long-term Maintainability

**Protocol upgrades:** When Stellar introduces a new CAP, `LedgerCloseMeta` structure may
change. We will track Stellar Core releases and update `@stellar/stellar-sdk` XDR types
accordingly. Protocol upgrades are infrequent and well-announced.

**Open-source re-deployability:** The full CDK stack is public; Stellar or any third party
can fork the repository and redeploy the entire system in a fresh AWS account by following
the README. This is verified as part of the mainnet launch acceptance criteria (see below).

**Data retention:** The block explorer stores full history indefinitely for ledgers and
transactions (essential for a block explorer). Event and invocation tables will use
PostgreSQL native range partitioning by month, enabling instant partition drops if storage
constraints require pruning older fine-grained data while keeping ledger and transaction
summaries intact.

---

## Revised Deliverables

The deliverables below supersede the original submission. Each milestone includes
acceptance criteria that are objectively verifiable by the reviewer without access to our
internal systems.

---

**[Deliverable 1] Indexing Pipeline & Core Infrastructure**

• Galexie ECS Fargate task running on mainnet, writing `LedgerCloseMeta` XDR files to S3
  every ~5–6 seconds. Lambda Ledger Processor triggered per file, parsing and writing
  ledgers, transactions, operations, Soroban invocations, and CAP-67 events to a dedicated
  RDS PostgreSQL database. Historical backfill from Soroban mainnet activation ledger
  (late 2023) to genesis of new architecture. NestJS API scaffolding with core modules
  (Transactions, Ledgers, Network stats, Contracts). OpenAPI specification. AWS CDK
  infrastructure-as-code. CI/CD pipeline (GitHub Actions → `cdk deploy`). CloudWatch
  dashboards and ingestion lag alarms.

• How to measure completion:
  1. S3 bucket contains consecutive `LedgerCloseMeta` files with timestamps matching
     mainnet ledger close times (verifiable by inspector script provided in repo)
  2. RDS `ledgers` table contains all ledgers from backfill start through current tip with
     no gaps (verified by: `SELECT COUNT(*) ... WHERE sequence BETWEEN X AND Y`)
  3. RDS `soroban_events` table contains CAP-67 events for known Soroswap/Aquarius/Phoenix
     transactions (spot-checked by transaction hashes provided by reviewer)
  4. `cdk deploy` from a clean AWS account produces the full working stack with no manual
     steps (README instructions verified)
  5. CloudWatch dashboard accessible; Galexie lag alarm fires when S3 file is artificially
     delayed in staging

• Budget: $26,240 (20% of total)

---

**[Deliverable 2] Complete API + Frontend**

• All REST API endpoints live and serving mainnet data: transactions (list + detail),
  ledgers (list + detail), accounts (detail + history), contracts (detail + invocations +
  events), search (exact match + prefix). Human-readable event interpretation for known
  patterns (swaps, transfers, mints) on transaction detail pages. React SPA deployed via
  CloudFront with all pages: Home, Transaction List, Transaction Detail, Ledger Detail,
  Account Detail, Contract Detail, Search. Rate limiting (per-IP and API key tiers) and
  response caching configured on API Gateway.

• How to measure completion:
  1. All API endpoints return schema-valid responses for a set of mainnet entity IDs
     provided by the reviewer (or taken from public Stellar mainnet)
  2. Soroban invocations on Contract Detail page show function name, arguments, and return
     value (not raw XDR) for at least 3 known Soroswap/Aquarius contract transactions
  3. CAP-67 events appear on Transaction Detail page under Events tab with decoded topics
     and data fields (not raw XDR)
  4. Global search returns correct entity type and redirects to detail page for an exact
     transaction hash, account ID, and contract ID provided by the reviewer
  5. React frontend is publicly accessible at staging URL; all pages render live mainnet
     data without errors in browser console

• Budget: $39,360 (30% of total)

---

**[Deliverable 3] Mainnet Launch**

• Production deployment on mainnet at public URL. Unit and integration tests covering XDR
  parsing correctness, API endpoint responses, and event interpretation logic. Load test
  results documented (1M baseline, 10M stress). Security audit checklist (OWASP Top 10,
  CORS, IAM least-privilege, no public RDS endpoint). Monitoring dashboards and alerting
  active and accessible to Stellar team. Full API reference documentation published. GitHub
  repository made public with architecture docs, local dev setup, and deploy instructions.
  Professional user testing completed. 7-day post-launch monitoring report.

• How to measure completion:
  1. Block explorer publicly accessible at announced production URL, showing live mainnet
     data with ledger sequences matching network tip within 30 seconds
  2. GitHub repository public; `cdk deploy` from README works in a fresh AWS account
     (verified by reviewer or documented in screen recording)
  3. CloudWatch dashboard accessible to Stellar team (read-only IAM role); all alarms in
     OK state at launch; Galexie ingestion lag shown to be <30 s from network tip
  4. Load test report provided: p95 <200 ms at 1M requests/month equivalent load; error
     rate <0.1%
  5. Security checklist signed off: no wildcard IAM permissions, RDS has no public
     endpoint, all secrets in Secrets Manager, all API inputs validated
  6. 7-day post-launch monitoring report: uptime %, API error rate, p95 latency, Galexie
     ingestion lag per day

• Budget: $52,480 (40% of total + professional user testing)
