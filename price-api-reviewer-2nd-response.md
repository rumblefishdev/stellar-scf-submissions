# Prices API — Response to Second-Round Reviewer Questions

This document responds directly to the four questions raised in the second-round review of
the Prices API submission. All changes referenced here are reflected in the updated design
document `prices-api-design-after-2nd-review.md`.

---

## Question 1: Detailed Backfill Plan — Compute Cost, Processing Rate, Milestones in Tranches 2 and 3, Progress Endpoint

### Two-Stream Design

The two price data sources live in completely different places in the ledger, which drives a
two-stream backfill approach with different methods, timelines, and costs:

| Stream | What it is | Where historical data lives | Backfill method |
|--------|-----------|---------------------------|----------------|
| **SDEX trades** | Native order book fills recorded in `OperationResult → offersClaimed[]` | All 57M ledgers in Stellar public history archives | ECS Fargate task reading archives |
| **Soroban AMM swaps** | CAP-67 events emitted by Soroswap, Aquarius, Phoenix contracts | Already parsed and stored in Block Explorer `soroban_events` table | Read-only query to Block Explorer RDS |

SDEX and Soroban events are fundamentally distinct data paths in `LedgerCloseMeta` — a SDEX
trade never produces a CAP-67 event, and a Soroban AMM swap never produces an `offersClaimed`
entry. This makes the two streams independently backfillable.

### Why only the SDEX stream cannot complete within 13 weeks

The Soroban AMM stream is resolved quickly in Tranche 1 by querying the Block Explorer's
already-indexed data. The SDEX stream is the long-running backfill.

A cost-sustainable SDEX archive backfill runs at approximately **150,000 ledgers/hour**, limited
by Prices API RDS write throughput (db.t4g.small during backfill). At this rate:

```
57,000,000 ledgers ÷ 150,000 ledgers/hour = ~380 hours (~16 days) of pure compute time
```

With RDS write variance, backfill task restarts, and rollup worker overhead, the full SDEX
backfill is expected to take **4–6 weeks of continuous operation** after it starts in Tranche 1.
Since Tranche 3 is at Week 13 and the task starts at Week 2, approximately 11 weeks are
available — leaving a tail of 4–6 weeks post-delivery.

### Architecture

**Stream 1 — Soroban AMM (Tranche 1, fast):**

The Soroban AMM backfill task queries the Block Explorer's `soroban_events` table with a
read-only connection (shared VPC). It filters by the known contract IDs for Soroswap, Aquarius,
and Phoenix, extracts swap amounts and token pairs from the decoded JSONB `topics`/`data`
fields, and writes OHLCV candles into the Prices RDS. This runs for a few hours and completes
during Tranche 1.

After the DB-sourced pass, a gap-detection check verifies contiguous OHLCV coverage from Soroban
activation to present. Any ledger ranges missing from the Block Explorer (e.g. gaps at startup)
trigger a targeted archive read for those specific ranges only.

**Stream 2 — SDEX (continuous, through and past Tranche 3):**

A separate ECS Fargate task (Rust binary, 2 vCPU / 4 GB RAM) reads `LedgerCloseMeta` files
from Stellar's public history archives, extracts `offersClaimed[]` from `ManageSellOffer` and
`ManageBuyOffer` operation results, and writes 1-min OHLCV candles. It updates a heartbeat in
the `backfill_progress` table every 15 minutes.

### Compute Cost of the Backfill

**Stream 1 — Soroban AMM:**

| Component | Configuration | One-time cost |
|-----------|--------------|--------------|
| ECS Fargate backfill task | 1 vCPU / 2 GB RAM, a few hours (Tranche 1) | ~$2 |
| Block Explorer RDS read access | Read-only, no additional RDS cost | $0 |
| **Soroban AMM total** | | **~$2** |

**Stream 2 — SDEX:**

| Component | Configuration | One-time cost |
|-----------|--------------|--------------|
| ECS Fargate backfill task | 2 vCPU / 4 GB RAM, ~13 weeks continuous (2,184 hours) | ~$216 |
| RDS during backfill | db.m6g.large ($131/mo, non-burstable) for ~3 months; downgrade to db.t4g.micro after. The `t4g` burstable family exhausts CPU credits under sustained writes and throttles to ~20% of one vCPU — unsuitable for a weeks-long backfill | ~$393 |
| S3 reads from Stellar public archives | ~57M file reads at minimal S3 request pricing | ~$25 |
| **SDEX total** | | **~$634** |

**Grand total one-time backfill compute: ~$636**, appearing as a separate budget line item
in Tranche 1.

### Milestones in Tranches 2 and 3

| Tranche | Week | Stream | Milestone | Concrete Validation |
|---------|------|--------|-----------|---------------------|
| **1** | 4 | Soroban AMM | Full AMM history from Soroban activation (Nov 2023) | `soroban_amm.status: "completed"`; OHLCV data for Soroswap pairs verifiable for Nov 2023 dates |
| **1** | 4 | SDEX | Archive backfill running; recent 6 months covered | `sdex.task_healthy: true`; `sdex.earliest_data_available` ~6 months ago |
| **2** | 9 | SDEX | 4+ years of SDEX history (back to Jan 2022) | `sdex.earliest_data_available` ≤ 2022-01-01; reviewer spot-checks XLM/USDC OHLCV for specific 2022 dates |
| **3** | 13 | SDEX | 8+ years of SDEX history (back to Jan 2018) | `sdex.earliest_data_available` ≤ 2018-01-01; `sdex.task_healthy: true`; `estimated_hours_to_completion` shows credible remaining estimate |
| **Post-delivery** | ~Week 17–19 | SDEX | Full all-time SDEX history | `sdex.status: "completed"`; Stellar notified |

**The Tranche 3 review does not require the SDEX backfill to be complete.** It validates that:
1. The SDEX backfill is running and healthy
2. The processing rate is stable (variation <20% over 24 hours)
3. The estimated completion date is concrete and credible
4. The Soroban AMM backfill completed in Tranche 1 as expected

### `GET /backfill/status` — Progress and Health Endpoint

This endpoint is live from Tranche 1 onwards. It exposes both streams independently.

```json
{
  "realtime_tip_ledger": 57234198,
  "sdex": {
    "status": "running",
    "current_ledger": 34891234,
    "start_ledger": 1,
    "target_ledger": 57234198,
    "progress_pct": 61.2,
    "ledgers_remaining": 22342964,
    "rate_ledgers_per_hour": 148200,
    "estimated_hours_to_completion": 150.7,
    "task_healthy": true,
    "last_heartbeat": "2026-06-15T14:29:55Z",
    "earliest_data_available": "2019-08-22T00:00:00Z"
  },
  "soroban_amm": {
    "status": "completed",
    "completed_at": "2026-04-14T08:23:11Z",
    "earliest_data_available": "2023-11-01T00:00:00Z"
  }
}
```

**Health monitoring:** If `sdex.last_heartbeat` falls more than 10 minutes behind the current
time, a CloudWatch alarm fires (SNS → email + Slack). This alarm is active for the full SDEX
backfill duration, including post-delivery.

The endpoint is queryable by Stellar without an API key (rate-limited to 60 requests/hour per
IP) so they can monitor progress independently at any time.

---

## Question 2: Explicit List of Architecture Components Shared with the Funded Block Explorer

The following table lists every shared component and confirms no double-billing.

| Component | Block Explorer budget | Prices API budget | How the sharing works |
|-----------|----------------------|-------------------|-----------------------|
| **Galexie ECS Fargate task** | Yes — 1 task, 1 vCPU / 2 GB, continuous | **$0 — not billed** | The Prices API Ledger Processor Lambda is registered as a **second S3 event notification target** on the `stellar-ledger-data/` bucket. When Galexie writes a ledger file, S3 fans out the notification to both the Block Explorer's Ledger Processor and the Prices API Ledger Processor. No second Galexie instance is needed |
| **S3 bucket `stellar-ledger-data/`** | Yes — bucket owned and managed by Block Explorer | **$0 — not billed** | Same files, second Lambda reads them. S3 storage cost is trivial and already covered |
| **VPC (us-east-1a), subnets, security groups, route tables** | Yes | **$0 — not billed** | Prices API RDS instance and Lambda functions deploy into the existing VPC and private subnets |
| **NAT Gateway** | Yes — one NAT Gateway for Block Explorer egress | **$0 — not billed** | Single NAT Gateway handles egress for both services |
| **ECS Fargate cluster** | Yes | **$0 — not billed** | Prices API historical backfill task runs as a separate task definition in the same cluster |
| **`stellar-xdr` XDR parsing crate** | Yes — Rust workspace crate written for Block Explorer | **$0 — not billed** | Same crate compiled into the Prices API Ledger Processor. No duplication of parsing logic |
| **Block Explorer `soroban_events` table (read-only)** | Yes — Block Explorer indexes all CAP-67 events including Soroswap, Aquarius, Phoenix swaps | **$0 — not billed** | The Soroban AMM backfill task connects read-only to Block Explorer RDS (same VPC) to extract decoded swap event history. Eliminates re-reading ~8.5M Soroban-era ledgers from archives. The Prices API does not write to Block Explorer data. This is a one-time read during Tranche 1; after the AMM backfill completes the connection is closed |

### What the Prices API does bill for

The following are **exclusive to the Prices API** and are fully funded by the Prices API grant:
- Prices API RDS PostgreSQL instance (separate schema: OHLCV, oracle, current prices)
- Prices API Lambda functions (Ledger Processor, Rollup, Price Updater, Oracle Fetcher, Asset Discovery, Cleanup, API handlers)
- Prices API API Gateway + usage plans
- Prices API EventBridge rules
- Prices API Secrets Manager entries
- Prices API CloudWatch dashboards and alarms
- Historical backfill ECS task (separate task definition, separate IAM role)

The Prices API does not read from the Block Explorer's RDS. The only shared data path is the
`stellar-ledger-data/` S3 bucket — both services read but never write to each other's data.

---

## Question 3: Budget Reductions Accounting for Infrastructure Overlap

### Removed Items (compared to pre-review budget)

| Budget item removed | Monthly value | Reason |
|--------------------|--------------|--------|
| Galexie ECS Fargate task | ~$36/mo | Shared with Block Explorer — already operational and funded |
| NAT Gateway | ~$35/mo | Shared with Block Explorer |
| **Total removed** | **~$71/mo** | |

### Revised Monthly Running Cost (post-backfill, low traffic)

| Service | Before sharing | After sharing |
|---------|---------------|--------------|
| Galexie ECS Fargate | ~$36 | **$0** |
| NAT Gateway | ~$35 | **$0** |
| RDS PostgreSQL (db.t4g.micro) | ~$12 | ~$12 |
| Lambda — API handlers | ~$20 | ~$20 |
| Lambda — Ingestion workers | ~$10 | ~$10 |
| API Gateway | ~$50 | ~$50 |
| CloudWatch + X-Ray | ~$20 | ~$20 |
| Secrets Manager + S3 | ~$5 | ~$5 |
| **Total** | **~$188/mo** | **~$117/mo** |

The removal of Galexie and NAT Gateway from the Prices API budget represents a **38% reduction
in monthly infrastructure cost** relative to the prior design.

### Development Savings (reflected in reduced effort estimates)

The shared Rust workspace means the following is not built twice:

| Shared artifact | Development days saved |
|----------------|----------------------|
| `stellar-xdr` XDR parsing logic | ~5–7 days |
| `sqlx` database patterns + migration tooling | ~2–3 days |
| CDK VPC + networking constructs | ~3–4 days |
| CloudWatch dashboard and alarm patterns | ~1–2 days |
| **Total** | **~11–16 days** |

These savings are reflected in the Prices API effort estimates: the infrastructure and ingestion
pipeline sections are smaller than they would be for a greenfield project.

---

## Question 4: Separate Budget Items for Historical Backfill; Tranche 3 Review Scope

### New Budget Line Items for Historical Backfill

The backfill is a distinct deliverable with two separate budget lines: the Soroban AMM stream
(fast, Tranche 1) and the SDEX archive stream (long-running, through and past Tranche 3).

| Budget item | Stream | Tranche | One-time cost | Notes |
|-------------|--------|---------|--------------|-------|
| ECS Fargate — Soroban AMM backfill task | Soroban AMM | 1 | ~$2 | 1 vCPU / 2 GB, a few hours; queries Block Explorer RDS |
| ECS Fargate — SDEX archive backfill task | SDEX | 1 | ~$216 | 2 vCPU / 4 GB, 13 weeks continuous |
| RDS during SDEX backfill | SDEX | 1 | ~$393 | db.m6g.large ($131/mo, non-burstable) for ~3 months; t4g burstable family is unsuitable for sustained writes |
| S3 archive reads (Stellar public history) | SDEX | 1 | ~$25 | |
| **Total backfill compute** | | | **~$636** | One-time, during project |

### Tranche 3 Review — Backfill Validation vs. Completion

The Tranche 3 review does **not** require the backfill to be complete. The schedule is
structured so that Tranche 3 can proceed on the designated week regardless of backfill status.

**What Tranche 3 reviews for backfill:**
1. `GET /backfill/status` returns `status: running` and `backfill_task_healthy: true`
2. `earliest_data_available` ≤ 2018-01-01 (8+ years of history available)
3. `rate_ledgers_per_hour` shows a stable, non-zero processing rate
4. `estimated_hours_to_completion` provides a concrete, credible remaining estimate
5. The CloudWatch alarm for backfill heartbeat is configured and in OK state

**What happens after Tranche 3:**
- The backfill task continues running autonomously
- No additional development work is required from the Rumble Fish team
- Stellar can monitor progress via `GET /backfill/status` at any time
- When `status` transitions to `completed`, the endpoint records `completed_at` and the
  `earliest_data_available` field shows the inception date of the earliest-indexed asset
- The Rumble Fish team will notify Stellar when the backfill completes (email + public endpoint)

This structure allows the Tranche 3 review to proceed on schedule while ensuring the reviewer
can confirm the backfill is actively progressing and will complete autonomously.
