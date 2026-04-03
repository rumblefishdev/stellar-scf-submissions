# Prices API — Technical Design Document (Post-2nd-Review)

> This document supersedes `prices-api-design-after-review.md`. It incorporates changes
> required by second-round reviewer feedback: shared infrastructure with the funded Soroban
> Block Explorer is explicitly catalogued and removed from the Prices API budget; the
> technology stack is updated to Rust (axum + sqlx) to match the block explorer codebase;
> a concrete historical backfill plan with milestones in Tranches 2 and 3 is added; and a
> `GET /backfill/status` API endpoint is introduced to allow progress monitoring.

---

## 0. Deployment & AWS Account

The service will be deployed to the **same dedicated AWS sub-account** as the already-funded
and operational Soroban Block Explorer (Rumble Fish, awarded March 2026). Both services share
core infrastructure — Galexie ECS, S3 ledger bucket, VPC, and NAT Gateway — eliminating
duplicate operational costs. See Section 11 for a full accounting of shared components.

Infrastructure is managed via AWS CDK and deployed through a shared CI/CD pipeline (GitHub
Actions). The codebase is fully open source — Stellar retains the ability to fork and redeploy
to their own infrastructure at any time if needed.

---

## 1. Architecture Overview

### 1.1 API Layer

```
                         ┌────────────────────────────────┐
                         │       AWS API Gateway           │
                         │  (REST API, rate limiting,      │
                         │   API keys, throttling,         │
                         │   built-in response caching)    │
                         └────────────┬───────────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │      AWS Lambda           │
                         │  (API handler functions)  │
                         │   Rust / axum             │
                         └────────────┬─────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │     RDS PostgreSQL        │
                         │  (db.t4g.micro, Single-AZ)│
                         └──────────────────────────┘
```

### 1.2 Data Ingestion Layer

```
┌─────────────────────────────────────────────────────────────┐
│  SHARED WITH SOROBAN BLOCK EXPLORER (no additional cost)    │
│                                                             │
│  Stellar Network (mainnet peers)                            │
│          │                                                  │
│          ▼ (Captive Core / ledger stream)                   │
│  ┌──────────────────────────────────┐                       │
│  │  Galexie — ECS Fargate (1 task)  │                       │
│  │  Continuously running            │                       │
│  │  Exports one file per ledger     │                       │
│  │  (~1 file every 5–6 seconds)     │                       │
│  └──────────────┬───────────────────┘                       │
│                 │ LedgerCloseMeta XDR (zstd-compressed)     │
│                 ▼                                           │
│  ┌──────────────────────────────────┐                       │
│  │  S3: stellar-ledger-data/        │                       │
│  │  ledgers/{seq_start}-            │                       │
│  │         {seq_end}.xdr.zstd       │                       │
│  └──────────────┬───────────────────┘                       │
└─────────────────┼───────────────────────────────────────────┘
                  │ S3 PutObject event notification
                  │ (second notification target, alongside
                  │  Block Explorer's Ledger Processor)
                  ▼
┌──────────────────────────────────────────────────────┐
│  Lambda "Prices Ledger Processor" (per file, Rust)   │
│  1. Download + decompress XDR                        │
│  2. Parse LedgerCloseMeta via stellar-xdr crate      │
│  3. Extract SDEX trades (ManageSellOfferResult /     │
│     ManageBuyOfferResult → offersClaimed[])          │
│  4. Extract Soroban swap events (CAP-67):            │
│     Soroswap, Aquarius, Phoenix — all in one stream  │
│  5. Write 1-min OHLCV candles to Prices RDS          │
└──────────────────────────────────────────────────────┘
               │
               ▼
       ┌───────────────┐      ┌───────────────────────┐
       │  Prices RDS   │◄─────│  EventBridge-triggered │
       │  PostgreSQL   │      │  Lambda workers (Rust): │
       └───────────────┘      │  - OHLCV Rollup        │
                              │  - Current Price Upd.  │
                              │  - Oracle Fetcher      │
                              │  - Asset Discovery     │
                              │  - Cleanup Worker      │
                              └───────────────────────┘

       ┌────────────────────────────────────────────────┐
       │  External read-only (metadata / cross-ref)     │
       │  Reflector Oracle (Soroban RPC simulate)       │
       │  Soroswap API   (pool pair metadata only)      │
       │  Aquarius API   (pool pair metadata only)      │
       └────────────────────────────────────────────────┘
```

---

## 2. AWS Services Breakdown

### 2.1 Components Hosted by Prices API (Prices API budget)

| Service | Role | Details |
|---------|------|---------|
| **Lambda — Prices Ledger Processor** | Primary ingestion | Event-driven (per S3 file, same bucket as Block Explorer). Rust binary. Parses XDR via `stellar-xdr` crate, extracts SDEX trades and Soroban swap events, writes 1-min OHLCV candles to Prices RDS |
| **Lambda — Current Price Updater** | Price aggregation | EventBridge rate(1 min). Reads latest candles from `price_ohlcv`, computes VWAP across sources, writes to `current_prices` |
| **Lambda — OHLCV Rollup** | Candle aggregation | EventBridge rate(15 min). Rolls 1m → 15m, 1h, 4h, 1d, 1w, 1M |
| **Lambda — Oracle Fetcher** | Oracle cross-reference | EventBridge rate(5 min). Reads Reflector via Soroban RPC `simulateTransaction`. Writes to `oracle_prices`. Failures do not block primary ingestion |
| **Lambda — Asset Discovery** | Asset registry | EventBridge rate(1 hour). Detects new SEP-41 contract deployments and classic asset issuances |
| **Lambda — Cleanup Worker** | Data retention | EventBridge cron(02:00 UTC daily). Deletes expired fine-grained candles, drops old partitions, creates upcoming partitions |
| **Lambda — API handlers** | Public API | Individual functions per route group. Rust / axum via `lambda_runtime`, 256–512 MB, 15s timeout |
| **RDS PostgreSQL** | Primary data store | `db.t4g.micro` (2 vCPU burstable, 1 GB RAM), Single-AZ to start. Native range partitioning. Upgraded to `db.t4g.small` during backfill period |
| **API Gateway** | Public API entry point | REST API, usage plans, API key auth, rate limiting (100 req/s per key), request validation. Built-in response cache (0.5 GB) with per-endpoint TTLs |
| **EventBridge Scheduler** | Scheduled triggers | Cron/rate rules for all periodic Lambda workers |
| **Secrets Manager** | Credentials | DB password, Soroswap/Aquarius API keys, oracle contract address |
| **CloudWatch + X-Ray** | Observability | API latency, error rates, ingestion lag, Lambda duration/concurrency, backfill progress |
| **S3** (API docs) | Documentation hosting | OpenAPI spec + self-service onboarding portal, served via CloudFront |

### 2.2 External Services Consumed (read-only)

| External service | Purpose | Failure impact |
|-----------------|---------|----------------|
| Reflector Oracle (Soroban RPC) | Price cross-reference only — not a primary source | Non-critical — oracle column shows `null` or last known value |
| Soroswap API | Pool pair metadata and discovery | Non-critical — existing pairs continue; new pair discovery pauses |
| Aquarius API | Pool pair metadata and discovery | Non-critical — same as above |

### 2.3 Components Shared with Soroban Block Explorer (no additional charge)

These components are funded by the Soroban Block Explorer grant and already operational. They
are listed here to confirm there is no double-billing.

| Component | Block Explorer context | Prices API usage |
|-----------|----------------------|-----------------|
| **Galexie ECS Fargate task** | 1 task, continuous, 1 vCPU / 2 GB, writes to `stellar-ledger-data/` S3 every ~5–6 s | Prices API adds its Ledger Processor Lambda as a **second S3 event notification target** on the same bucket. No second Galexie needed |
| **S3 bucket `stellar-ledger-data/`** | Owned and funded by Block Explorer | Prices API Lambda reads the same files; no additional S3 storage cost |
| **VPC (us-east-1a)** | Owned by Block Explorer | Prices API RDS and Lambda functions deploy into the same VPC and private subnets |
| **NAT Gateway** | Block Explorer's single NAT Gateway, billed to Block Explorer | Prices API Lambda egress traffic routed through the same gateway; no second NAT Gateway provisioned |
| **ECS Fargate cluster** | Block Explorer's cluster | Prices API historical backfill tasks run in the same ECS cluster (separate task definition, no cluster fee) |
| **GitHub Actions CI/CD patterns** | Shared pipeline structure and CDK conventions | Prices API pipeline reuses the same CDK deployment pattern |
| **Block Explorer `soroban_events` table (read-only)** | Contains all decoded CAP-67 events (Soroswap, Aquarius, Phoenix swaps) from Soroban activation to present | Prices API backfill task connects read-only to Block Explorer RDS (same VPC) to extract Soroban AMM swap history. Eliminates ~8.5M ledger archive reads for the Soroban AMM stream. This creates a schema dependency; documented in Section 11 |

**Confirmed: none of the components in 2.3 appear in the Prices API budget.**

---

## 3. Database Schema (PostgreSQL with Native Range Partitioning)

### 3.1 Assets Table
```sql
CREATE TABLE assets (
    id              SERIAL PRIMARY KEY,
    asset_code      VARCHAR(12) NOT NULL,
    asset_type      VARCHAR(10) NOT NULL CHECK (asset_type IN ('classic', 'soroban')),
    issuer_address  VARCHAR(56),          -- G-address, NULL for XLM
    contract_address VARCHAR(56),         -- C-address (SAC or native contract)
    home_domain     VARCHAR(255),         -- classic assets only, nullable
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE (asset_code, issuer_address, contract_address)
);

CREATE INDEX idx_assets_contract ON assets(contract_address);
CREATE INDEX idx_assets_code ON assets(asset_code);
```

### 3.2 Price Snapshots (OHLCV) — Native Range Partitioning
```sql
-- Parent table: partitioned by month on timestamp
CREATE TABLE price_ohlcv (
    asset_id        INT NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    granularity     VARCHAR(5) NOT NULL,   -- '1m', '15m', '1h', '4h', '1d', '1w', '1M'
    open            NUMERIC(28,14) NOT NULL,
    high            NUMERIC(28,14) NOT NULL,
    low             NUMERIC(28,14) NOT NULL,
    close           NUMERIC(28,14) NOT NULL,
    volume_base     NUMERIC(28,14) NOT NULL DEFAULT 0,
    volume_quote_usd NUMERIC(28,14) NOT NULL DEFAULT 0,
    vwap            NUMERIC(28,14),
    trade_count     INT DEFAULT 0,
    source          VARCHAR(20),           -- 'sdex', 'soroswap', 'aquarius', 'aggregated'

    PRIMARY KEY (timestamp, asset_id, granularity)
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (managed by Cleanup Worker Lambda)
CREATE TABLE price_ohlcv_2026_01 PARTITION OF price_ohlcv
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE price_ohlcv_2026_02 PARTITION OF price_ohlcv
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... new partitions created 2 months ahead by the cleanup-worker Lambda

-- Index for typical query: asset + granularity within a time range
CREATE INDEX idx_ohlcv_asset_gran ON price_ohlcv (asset_id, granularity, timestamp DESC);
```

**Why native partitioning works well here:**
- Queries with `WHERE timestamp > X` only scan relevant monthly partitions (partition pruning)
- Retention = `DROP TABLE price_ohlcv_2025_01` — instant, no vacuum needed
- No extension dependencies — works on plain AWS RDS PostgreSQL
- Monthly partitions keep each partition at a manageable size
- Backfill writes into historical partitions (pre-2026) alongside live writes into current
  partitions, with no locking conflicts

### 3.3 Current Prices (Materialized/Cached)
```sql
CREATE TABLE current_prices (
    asset_id        INT PRIMARY KEY REFERENCES assets(id),
    price_usd       NUMERIC(28,14) NOT NULL,
    price_xlm       NUMERIC(28,14),
    change_24h_pct  NUMERIC(10,4),
    change_7d_pct   NUMERIC(10,4),
    volume_24h_usd  NUMERIC(28,14),
    market_cap_usd  NUMERIC(28,14),
    vwap_24h        NUMERIC(28,14),
    sources         JSONB,                 -- {"sdex": 1.02, "soroswap": 1.01, "aquarius": 1.015}
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.4 Oracle Prices (Partitioned)
```sql
CREATE TABLE oracle_prices (
    asset_id        INT NOT NULL,
    oracle_name     VARCHAR(30) NOT NULL,  -- 'reflector', 'chainlink', 'redstone', 'band'
    price_usd       NUMERIC(28,14) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    raw_data        JSONB,

    PRIMARY KEY (timestamp, asset_id, oracle_name)
) PARTITION BY RANGE (timestamp);

-- Monthly partitions, same pattern as price_ohlcv
CREATE TABLE oracle_prices_2026_01 PARTITION OF oracle_prices
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE oracle_prices_2026_02 PARTITION OF oracle_prices
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE INDEX idx_oracle_asset ON oracle_prices (asset_id, oracle_name, timestamp DESC);
```

### 3.5 Backfill Progress Tracking
```sql
CREATE TABLE backfill_progress (
    id              SERIAL PRIMARY KEY,
    task_name       VARCHAR(50) NOT NULL UNIQUE, -- 'historical_all_time'
    start_ledger    BIGINT NOT NULL,
    target_ledger   BIGINT NOT NULL,
    current_ledger  BIGINT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'running',
    -- 'running', 'paused', 'completed', 'error'
    rate_per_hour   BIGINT,               -- ledgers/hour, updated every 15 min
    eta_hours       NUMERIC(10,1),        -- estimated hours to completion
    last_heartbeat  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
```

### 3.6 Retention Policy (Cleanup Worker Lambda)
```
Fine-grained data retention (DELETE within partitions):
  1m  granularity → keep 7 days   (DELETE WHERE granularity='1m' AND timestamp < now()-7d)
  15m granularity → keep 30 days  (DELETE WHERE granularity='15m' AND timestamp < now()-30d)

Coarse-grained data (1h, 4h, 1d, 1w, 1M) → keep forever

Partition lifecycle (DROP entire old partitions):
  Partitions older than 13 months → DROP TABLE (all fine-grained data already cleaned)
  New partitions → CREATE 2 months ahead of current date

Both handled by the cleanup-worker Lambda (daily at 02:00 UTC).
```

---

## 4. API Endpoints Design

**Base URL:** `https://api.prices.stellar.example.com/v1`

### 4.1 Assets

#### `GET /assets`
List all tracked assets with metadata and current price.

**Query parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | all | Filter: `classic`, `soroban`, or `all` |
| `search` | string | — | Search by asset code (prefix match) |
| `sort` | string | `volume_24h` | Sort by: `price`, `volume_24h`, `change_24h`, `code` |
| `order` | string | `desc` | `asc` or `desc` |
| `cursor` | string | — | Pagination cursor (Base64-encoded, see below) |
| `limit` | int | 50 | Max 200 |

**Cursor pagination mechanism:**

The cursor is a Base64-encoded JSON object containing the sort column value and the asset ID of the
last returned row (ID breaks ties when sort values are equal):

```
cursor = base64({ "volume_24h": 1523400.50, "id": 42 })
       → "eyJ2b2x1bWVfMjRoIjoxNTIzNDAwLjUwLCJpZCI6NDJ9"
```

On the first request (no cursor), the query is:
```sql
SELECT * FROM current_prices JOIN assets ON assets.id = current_prices.asset_id
ORDER BY volume_24h DESC, id DESC
LIMIT 51;  -- limit + 1 to determine has_more
```

On subsequent requests, the server decodes the cursor and uses a **keyset condition**:
```sql
SELECT * FROM current_prices JOIN assets ON assets.id = current_prices.asset_id
WHERE (volume_24h, id) < (1523400.50, 42)  -- decoded from cursor
ORDER BY volume_24h DESC, id DESC
LIMIT 51;
```

`has_more` is determined by fetching `limit + 1` rows.

**Response:**
```json
{
  "data": [
    {
      "asset_code": "USDC",
      "asset_type": "classic",
      "issuer_address": "GA5ZSE...XYZ",
      "contract_address": "CABC...DEF",
      "home_domain": "centre.io",
      "price_usd": "1.0001",
      "change_24h_pct": "-0.02",
      "change_7d_pct": "0.01",
      "volume_24h_usd": "1523400.50",
      "vwap_24h": "1.0002",
      "sources": {
        "sdex": "1.0001",
        "soroswap": "1.0002",
        "aquarius": "1.0001"
      },
      "updated_at": "2026-02-10T12:00:00Z"
    }
  ],
  "cursor": "eyJpZCI6NTB9",
  "has_more": true
}
```

#### `GET /assets/{asset_identifier}`
Get single asset details. `asset_identifier` can be:
- `{code}:{issuer}` for classic assets (e.g. `USDC:GA5ZSE...XYZ`)
- `{contract_address}` for Soroban tokens (e.g. `CABC...DEF`)
- `native` for XLM

### 4.2 Prices / OHLCV

#### `GET /assets/{asset_identifier}/ohlcv`
Historical OHLCV candlestick data.

**Query parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `timeframe` | string | `24h` | `1h`, `24h`, `7d`, `30d`, `1y`, `all` |
| `granularity` | string | auto | `1m`, `15m`, `1h`, `4h`, `1d`, `1w`, `1M`. Auto-selected from timeframe if omitted |
| `start` | ISO8601 | — | Custom range start (overrides timeframe) |
| `end` | ISO8601 | — | Custom range end |
| `base_currency` | string | `USD` | Quote currency: `USD` or `XLM` |

**Auto-selected granularity mapping:**
| Timeframe | Default Granularity | Max Data Points |
|-----------|-------------------|-----------------|
| `1h` | `1m` | ~60 |
| `24h` | `15m` | ~96 |
| `7d` | `1h` | ~168 |
| `30d` | `4h` | ~180 |
| `1y` | `1d` | ~365 |
| `all` | `1d` | variable |

When `timeframe=all` is requested but the backfill has not yet reached the asset's inception
date, the response includes a `backfill_note` field indicating how far back data is available:

```json
{
  "asset": "USDC:GA5ZSE...XYZ",
  "granularity": "1d",
  "base_currency": "USD",
  "backfill_note": "Historical data available from 2022-01-01. Backfill in progress — see GET /backfill/status.",
  "data": [...]
}
```

**Response:**
```json
{
  "asset": "USDC:GA5ZSE...XYZ",
  "granularity": "15m",
  "base_currency": "USD",
  "data": [
    {
      "timestamp": "2026-02-10T11:00:00Z",
      "open": "1.0001",
      "high": "1.0005",
      "low": "0.9998",
      "close": "1.0003",
      "volume_base": "125000.00",
      "volume_quote_usd": "125037.50",
      "vwap": "1.0003",
      "trade_count": 47
    }
  ]
}
```

#### `GET /assets/{asset_identifier}/price`
Current real-time price (latest snapshot from `current_prices`).

**Response:**
```json
{
  "asset": "USDC:GA5ZSE...XYZ",
  "price_usd": "1.0001",
  "price_xlm": "8.33",
  "vwap_24h": "1.0002",
  "volume_24h_usd": "1523400.50",
  "change_24h_pct": "-0.02",
  "sources": {
    "sdex":      { "price": "1.0001", "volume_24h": "800000" },
    "soroswap":  { "price": "1.0002", "volume_24h": "500000" },
    "aquarius":  { "price": "1.0001", "volume_24h": "223400" }
  },
  "updated_at": "2026-02-10T12:00:30Z"
}
```

### 4.3 Batch / Multi-asset

#### `POST /prices/batch`
Fetch current prices for multiple assets in one call.

**Request body:**
```json
{
  "assets": [
    "native",
    "USDC:GA5ZSE...XYZ",
    "CABC...DEF"
  ]
}
```

### 4.4 Oracle Data

#### `GET /oracles/{asset_identifier}`
Oracle-specific prices for an asset. Oracle data is exposed here for reference only and does not
feed the `price_usd` field in any other endpoint.

**Response:**
```json
{
  "asset": "USDC:GA5ZSE...XYZ",
  "oracles": [
    {"name": "reflector", "price_usd": "1.0000", "updated_at": "2026-02-10T11:55:00Z"},
    {"name": "redstone",  "price_usd": "1.0001", "updated_at": "2026-02-10T11:58:00Z"}
  ]
}
```

### 4.5 Backfill Progress

#### `GET /backfill/status`
Returns the current state of the historical all-time backfill. This endpoint is the primary
mechanism for the Tranche 3 reviewer to validate that backfill is progressing correctly.

A CloudWatch alarm fires if `last_heartbeat` falls more than 10 minutes behind the current
time (indicating the backfill task has stalled or crashed).

The backfill is split into two independent streams with different sources and timelines (see
Section 5.6). The response reflects both.

**Response:**
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

| Field | Description |
|-------|-------------|
| `sdex.status` | `running`, `paused`, `completed`, or `error` — SDEX archive backfill |
| `sdex.current_ledger` | Oldest ledger processed so far by the SDEX backfill task |
| `sdex.rate_ledgers_per_hour` | Rolling 15-min average processing rate |
| `sdex.estimated_hours_to_completion` | `ledgers_remaining / rate_per_hour` |
| `sdex.task_healthy` | `false` if no heartbeat in past 10 minutes → CloudWatch alarm fires |
| `sdex.earliest_data_available` | Timestamp of oldest SDEX OHLCV record in the database |
| `soroban_amm.status` | Typically `completed` from Tranche 1 onwards |
| `soroban_amm.earliest_data_available` | Soroban activation date (~Nov 2023) once complete |

---

## 5. Data Ingestion Pipeline

### 5.1 Galexie Operation

Galexie is the SDF's Composable Data Platform exporter. It runs as a shared ECS Fargate task
(funded under the Block Explorer grant) that connects to Stellar mainnet peers via Captive Core
and writes one `LedgerCloseMeta` XDR file to S3 on every ledger close (~every 5–6 seconds).

The Prices API does not deploy or fund a second Galexie instance. Instead, the Prices API
Ledger Processor Lambda is registered as a second S3 event notification target on the shared
`stellar-ledger-data/` bucket, alongside the Block Explorer's Ledger Processor.

| Property | Value |
|----------|-------|
| Runtime | ECS Fargate, 1 vCPU / 2 GB RAM, continuously running (Block Explorer funded) |
| Output | `s3://stellar-ledger-data/ledgers/{seq_start}-{seq_end}.xdr.zstd` |
| Format | `LedgerCloseMeta` XDR, zstd-compressed |
| Recovery | Checkpoint-aware — resumes from last exported sequence on restart |
| Lag monitoring | CloudWatch alarm fires if S3 file timestamps fall >60s behind current ledger |

### 5.2 Prices Ledger Processor (Rust)

The Prices Ledger Processor is triggered by S3 PutObject events. It is implemented in Rust
using the `stellar-xdr` crate (official SDF Rust XDR types) and deployed as a Lambda function
using the `lambda_runtime` crate. It downloads the file, parses it, and extracts:
- **SDEX trades** from `OperationResult` → `ManageSellOfferResult` / `ManageBuyOfferResult` →
  `offersClaimed[]` (price, base volume, quote volume, asset pair)
- **Soroban AMM swap events** (CAP-67) from `SorobanTransactionMeta` → `events` — this covers
  Soroswap, Aquarius, and Phoenix in a single stream

The XDR parsing logic is implemented as a shared Rust workspace crate, compiled into both the
Block Explorer's Ledger Processor and the Prices Ledger Processor, eliminating duplication.

### 5.3 Ingestion Workers

| Worker | Trigger | Source | Data |
|--------|---------|--------|------|
| **Prices Ledger Processor** | S3 PutObject event (per ledger, ~every 5–6 s) | `LedgerCloseMeta` from S3 | SDEX trades + all Soroban AMM swap events → 1-min OHLCV candles |
| **Oracle Fetcher** | EventBridge rate(5 min) | Reflector Oracle (Soroban RPC `simulateTransaction`) | Oracle reported prices → `oracle_prices` |
| **Asset Discovery** | EventBridge rate(1 hour) | Ledger account entries in `LedgerCloseMeta` | New classic asset issuances; new SEP-41 contract deployments |
| **OHLCV Rollup** | EventBridge rate(15 min) | Internal DB | Roll up 1m candles → 15m, 1h, 4h, 1d, 1w, 1M |
| **Current Price Updater** | EventBridge rate(1 min) | Internal DB (after ingestion) | VWAP across sources → `current_prices` table |
| **Cleanup Worker** | EventBridge cron(02:00 UTC) | Internal DB | Delete expired fine-grained data, drop old partitions, create upcoming partitions |
| **Backfill Task** | ECS Fargate, runs continuously during project | Stellar public history archives | Historical SDEX trades + Soroban swaps, written into historical `price_ohlcv` partitions |

### 5.4 EventBridge Scheduler Rules

```
prices-ledger-processor:  S3 event (PutObject) → Lambda "prices-ledger-processor"
oracle-ingest:             rate(5 minutes)       → Lambda "oracle-worker"
asset-discovery:           rate(1 hour)          → Lambda "discovery-worker"
ohlcv-rollup:              rate(15 minutes)      → Lambda "rollup-worker"
price-update:              rate(1 minute)        → Lambda "price-updater"
retention-cleanup:         cron(0 2 * * *)       → Lambda "cleanup-worker"
```

Note: the Prices Ledger Processor is event-driven (S3 notification), not schedule-driven. All
other workers remain on EventBridge schedules.

### 5.5 VWAP Calculation Logic

```
Weighted Price = Σ(source_price × source_volume_24h) / Σ(source_volume_24h)

Where sources = [SDEX, Soroswap, Aquarius, ...]
Only include sources where volume_24h > configurable_min_threshold_usd (e.g. $100)
```

Volume threshold is configurable per-request via `?min_volume_usd=` query param or defaults to
the system setting.

**Outlier detection:** before a source's price is included in the VWAP, it is compared against the
inter-source median. Sources deviating by more than a configurable percentage are excluded from
that update cycle.

### 5.6 Historical All-Time Backfill Plan

#### Scope and Two-Stream Design

The two price data sources are fundamentally different in where their historical data lives,
which drives a two-stream backfill design:

| Stream | Data location | Era | Method |
|--------|-------------|-----|--------|
| **SDEX trades** | `OperationResult → offersClaimed[]` in `LedgerCloseMeta` XDR | All-time (2015 → present, ~57M ledgers) | Archive reads via ECS Fargate task |
| **Soroban AMM swaps** (Soroswap, Aquarius, Phoenix) | `SorobanTransactionMeta.events` (CAP-67) — already parsed and stored in Block Explorer `soroban_events` table | Soroban activation (Nov 2023) → present (~8.5M ledgers) | Read-only query to Block Explorer RDS |

The Soroban AMM stream is handled first (Tranche 1) by querying the Block Explorer's already-indexed
`soroban_events` table. This completes quickly (hours, not weeks) and gives full AMM price history
from Soroban activation. The SDEX stream requires reading all 57 million ledgers from Stellar's
public history archives and is the long-running backfill that extends beyond the project duration.

#### Architecture

```
STREAM 1 — Soroban AMM (fast, Tranche 1)
─────────────────────────────────────────
Block Explorer RDS (read-only)
  soroban_events WHERE contract_id IN
  (Soroswap, Aquarius, Phoenix contracts)
        │
        ▼
┌──────────────────────────────────────────┐
│  Soroban AMM Backfill Task (Rust)        │
│  - Queries BE soroban_events by          │
│    contract_id and event topic           │
│  - Extracts token pair + amounts         │
│    from decoded JSONB topics/data        │
│  - Writes OHLCV into historical          │
│    price_ohlcv partitions                │
│  - Marks soroban_amm stream "completed"  │
└──────────────────────────────────────────┘

STREAM 2 — SDEX (slow, runs through and past Tranche 3)
────────────────────────────────────────────────────────
Stellar Public History Archives (S3-compatible)
        │
        ▼ (ledger range reads, oldest→newest)
┌──────────────────────────────────────────┐
│  SDEX Backfill ECS Fargate Task (Rust)   │
│  - Reads LedgerCloseMeta from archives   │
│  - Extracts offersClaimed[] from         │
│    ManageSellOffer/ManageBuyOffer results │
│  - Writes OHLCV into historical          │
│    price_ohlcv partitions                │
│  - Updates backfill_progress heartbeat   │
│    every 15 minutes                      │
└──────────────────────────────────────────┘
               │
               ▼
       Prices RDS PostgreSQL
       (historical price_ohlcv partitions,
        written in parallel with live writes)
```

Neither backfill task conflicts with live ingestion: native range partitioning separates
historical writes (old month partitions) from live writes (current month partition).

**Schema coupling note:** the Soroban AMM backfill task holds a read-only connection to the
Block Explorer's RDS within the shared VPC. It accesses only the `soroban_events` table and
does not write to it. If the Block Explorer's `soroban_events` schema changes, the backfill
task must be updated accordingly. This dependency is documented in Section 11.

#### Processing Rate and Compute Estimates

**Stream 1 — Soroban AMM (Block Explorer DB query):**

| Metric | Value | Notes |
|--------|-------|-------|
| Data source | Block Explorer `soroban_events` table | Already indexed, decoded JSONB |
| Ledger range | ~48.5M–57M (Nov 2023 to present) | ~8.5M ledgers worth of events |
| Estimated runtime | A few hours | DB query + OHLCV write; no archive reads needed |
| ECS task configuration | 1 vCPU / 2 GB RAM | Short-lived; same ECS cluster |
| Expected completion | During Tranche 1 (Week 2–3) | |

**Stream 2 — SDEX (archive reads):**

| Metric | Value | Notes |
|--------|-------|-------|
| Total ledgers | ~57 million | Ledger 1 (Nov 2015) to current tip |
| Task configuration | 2 vCPU / 4 GB RAM, ECS Fargate, continuous | Shared ECS cluster |
| Historical read rate | ~150,000–200,000 ledgers/hour | From public archives; single task |
| Effective sustained rate (DB limited) | ~150,000 ledgers/hour | db.t4g.small during backfill |
| Estimated total hours | ~380 hours (~16 days) of pure compute | 57M / 150,000 |
| Expected completion | ~4–6 weeks after Tranche 3 | Backfill runs continuously post-delivery |

The SDEX backfill task is self-recovering: it tracks `current_ledger` in `backfill_progress`
and resumes from its last checkpoint on restart. Early ledgers (pre-2018) have very few DEX
trades and process faster; the estimate above is conservative.

#### Backfill Milestones

| Tranche | Stream | Milestone | Validation |
|---------|--------|-----------|------------|
| **1** (Week 4) | Soroban AMM | Full AMM history from Soroban activation (Nov 2023) available | `soroban_amm.status: "completed"` in `GET /backfill/status`; OHLCV data for Soroswap pairs verifiable for Nov 2023 dates |
| **1** (Week 4) | SDEX | Archive backfill task running; covers recent 6 months of SDEX data | `sdex.task_healthy: true`; `sdex.earliest_data_available` ~6 months ago |
| **2** (Week 9) | SDEX | 4+ years of SDEX history (back to Jan 2022) | `sdex.earliest_data_available` ≤ 2022-01-01; reviewer spot-checks XLM/USDC OHLCV for 2022 dates |
| **3** (Week 13) | SDEX | 8+ years of SDEX history (back to Jan 2018) | `sdex.earliest_data_available` ≤ 2018-01-01; `sdex.task_healthy: true`; `estimated_hours_to_completion` shows credible remaining estimate |
| **Post-delivery** | SDEX | Full all-time SDEX history (ledger 1 to present) | `sdex.status: "completed"`; Stellar notified |

#### `GET /backfill/status` Alarm

A CloudWatch alarm monitors the `last_heartbeat` field in `backfill_progress`. If the backfill
task fails to update the heartbeat for more than 10 minutes, an SNS alarm fires (email + Slack)
to alert the team. This alarm is active for the full backfill duration, including post-delivery.

---

## 6. Performance & Scaling Strategy

### Target: <100ms p95 API response time

| Layer | Strategy |
|-------|----------|
| **API Gateway caching** | Built-in response cache (0.5 GB). Per-endpoint TTLs: `/assets` list 60s, `/ohlcv` 60s, `/price` 15s, `/backfill/status` 30s. Cache key includes query params. POST `/prices/batch` uncached |
| **API Gateway throttling** | Request throttling (100/s per API key, 1000/s global burst) |
| **Lambda** | Rust binary with `lambda_runtime`. Sub-millisecond cold starts. Stateless, auto-scales to concurrency limit |
| **Database** | Single writer instance to start. Direct Lambda→RDS connections (no proxy needed at low concurrency) |
| **Indexing** | Native range partitioning by month on `timestamp`. Composite index on `(asset_id, granularity, timestamp DESC)` per partition. Partition pruning eliminates irrelevant months |
| **Query optimization** | `current_prices` table avoids real-time aggregation. OHLCV queries hit pre-rolled-up data |

### RDS Sizing — Start Small, Scale Up

**Starting instance:** `db.t4g.micro` (2 vCPU burstable, 1 GB RAM) — ~$12/mo

**During backfill period:** upgraded to `db.m6g.large` (~$131/mo) — a non-burstable
general-purpose instance with 2 dedicated vCPU and 8 GB RAM. The `t4g` burstable family is
unsuitable for sustained backfill writes: CPU credits exhaust within hours under continuous
load and the instance throttles to its baseline fraction (~20% of one vCPU), collapsing write
throughput. After backfill completes, the instance is downgraded back to `db.t4g.micro`.

**Storage:** 20 GB gp3 initially, auto-scaling enabled up to 500 GB (to accommodate full
all-time history once backfill completes)

**Scaling path when traffic grows:**
| Trigger | Action |
|---------|--------|
| CPU credits running out regularly | Upgrade to `db.t4g.small` (~$25/mo) or `db.t4g.medium` (~$50/mo) |
| Sustained >50% CPU | Move to `db.r6g.large` (dedicated, 16 GB RAM, ~$175/mo) |
| Connection count approaching limit | Add RDS Proxy (~$25/mo) |
| Need high availability | Enable Multi-AZ (doubles RDS cost) |
| Read queries bottleneck | Add read replica for API reads |

---

## 7. Security Considerations

- **API keys** via API Gateway usage plans — required for all requests
- **Rate limiting** per API key to prevent abuse
- **Input validation:** Asset identifiers validated against known patterns (G-address: 56 chars
  starting with G, C-address: 56 chars starting with C)
- **Price manipulation protection:**
  - Outlier detection: reject source price data deviating >configurable% from inter-source median
    before including in VWAP
  - Volume-weighted averaging: sources with low 24h volume are down-weighted or excluded via
    `min_volume_usd` threshold
  - Oracle cross-reference: Reflector data available in `/oracles/{asset}` for consumers who
    wish to cross-check; it does not feed primary price fields
- **HTTPS only** (enforced at API Gateway)
- **No PII stored** — only blockchain-public data
- **IAM least-privilege:** each Lambda role scoped to only the resources it needs
- **RDS not publicly accessible:** only reachable from Lambda VPC

---

## 8. Tech Stack Summary

| Component | Technology |
|-----------|-----------|
| Language | Rust (edition 2021) |
| Runtime | AWS Lambda (Rust, custom `provided.al2` runtime via `lambda_runtime` crate) + ECS Fargate for Galexie and backfill task |
| API Framework | `axum` (HTTP router, shared with Block Explorer backend) |
| XDR parsing | `stellar-xdr` crate (official SDF Rust XDR types) — shared workspace crate with Block Explorer |
| Database client | `sqlx` (compile-time verified queries, async) |
| Database | PostgreSQL 16 (native range partitioning, no extensions) |
| Infrastructure | AWS CDK (TypeScript) — shared CDK app with Block Explorer stacks |
| CI/CD | GitHub Actions → `cdk deploy` — shared pipeline with Block Explorer |
| Monitoring | CloudWatch Logs + Metrics + Alarms + X-Ray tracing |
| API Docs | OpenAPI 3.0 spec, auto-generated from axum routes; Swagger UI hosted on S3 + CloudFront |

**Shared with Block Explorer codebase (same Rust workspace):**
- `stellar-xdr` parsing logic — compiled into both the Block Explorer Ledger Processor and the
  Prices Ledger Processor from the same workspace crate
- Database migration tooling (`sqlx migrate`)
- CloudWatch metric and alarm patterns
- CDK stack patterns (VPC, IAM, Lambda configuration)

---

## 9. Delivery Plan — Three Tranches

### Tranche 1 — Infrastructure & Real-time Ingestion (Weeks 1–4)

**Work:**
- AWS CDK stack provisioned: Prices API Lambda execution roles, API Gateway, Prices RDS,
  EventBridge rules, CloudWatch alarms, Secrets Manager entries
- Prices RDS running with full schema from Section 3 (all tables, partitions for current +
  next 2 months, all indexes, including `backfill_progress` table)
- Prices Ledger Processor Lambda deployed and registered as second S3 event notification target
  on the shared `stellar-ledger-data/` S3 bucket; confirmed processing live ledgers
- Asset Discovery Lambda running; `assets` table populated for at least 20 major assets
- Historical backfill ECS Fargate task started; processing from current tip backwards;
  covers approximately 6 months of recent history by end of Tranche 1
- `GET /backfill/status` endpoint live and returning valid progress data
- CloudWatch alarm: backfill heartbeat >10 min stale → SNS notification

**Acceptance criteria:**
1. `cdk deploy` from a clean AWS account (sharing only the existing VPC/S3 bucket from Block
   Explorer) produces the full Prices API stack with no manual steps
2. Prices RDS schema matches Section 3: all tables, partitions for current + next 2 months,
   all indexes present (verifiable via `\d+` psql output)
3. After 24 hours of live operation: `price_ohlcv` contains continuous 1-min candles for at
   least 20 major assets (XLM, USDC, EURC, AQUA, BTC, ETH) with no gaps >2 candles
4. `GET /backfill/status` returns `{"status": "running", "backfill_task_healthy": true}` with
   `backfill_current_ledger` advancing (ledger sequence decreasing toward ledger 1 over time)
5. CloudWatch alarm test: backfill task stopped manually → alarm fires within 15 minutes
6. `earliest_data_available` in `GET /backfill/status` shows a date approximately 6 months ago

**Budget: $XX,XXX (Tranche 1)**

---

### Tranche 2 — Public API (Weeks 5–9)

**Work:**
- Core API endpoints implemented and deployed:
  - `GET /assets` (paginated, sortable, filterable)
  - `GET /assets/{asset_identifier}` (single asset)
  - `GET /assets/{asset_identifier}/price` (current price with `sources` breakdown)
  - `GET /assets/{asset_identifier}/ohlcv` (OHLCV with timeframe/granularity params, with `backfill_note` when history is partial)
  - `POST /prices/batch` (multi-asset current price)
  - `GET /oracles/{asset_identifier}` (Reflector cross-reference data)
  - `GET /backfill/status`
- API Gateway: response caching (TTLs per Section 6), usage plans, API key issuance, throttling
- Full VWAP formula wired into Current Price Updater Lambda (Section 5.5)
- Outlier detection: sources deviating >configurable% from inter-source median excluded
- Aquarius pool metadata integration: Aquarius appearing as a named source in VWAP
- Input validation: asset identifier format enforced, param ranges validated, 400 on invalid input

**Backfill milestone for Tranche 2:**
By the end of Week 9, the backfill task will have been running continuously for approximately
5 weeks. At ~150,000 ledgers/hour sustained, that is ~25.2 million ledgers processed, covering
approximately **January 2022 to present** (4+ years of price history, including all of the
Soroban era plus 2 years of pre-Soroban SDEX data).

**Acceptance criteria:**
1. All 7 endpoint groups return correct, schema-valid responses for at least 20 major assets
2. Load test (k6 or Locust, script provided): 100 req/s sustained for 5 minutes on
   `GET /assets/{id}/price` → p95 latency <200ms, error rate <0.1%
3. Cache confirmed: consecutive identical requests within TTL window return `X-Cache: Hit` header
4. VWAP calculation verifiable against raw `price_ohlcv` rows for at least 3 assets
5. `GET /backfill/status` shows `earliest_data_available` ≤ 2022-01-01
6. OHLCV data for `?timeframe=all` on USDC returns data points from at least January 2022,
   with correct 1d candles verifiable against known USDC price history (spot-check dates
   provided by reviewer)

**Budget: $XX,XXX (Tranche 2)**

---

### Tranche 3 — Production Launch & Validation (Weeks 10–13)

**Work:**
- OpenAPI 3.0 specification covering all endpoints
- Self-service onboarding portal (S3 + CloudFront): API key request form, quickstart guide,
  example queries
- Integration test suite (automated, runs in CI): covers all 7 endpoint groups
- Load test report: k6 or Locust, documented test plan, results at 100/s, 500/s, 1000/s
- Security review checklist: IAM least-privilege, no secrets in env vars, input sanitization
- X-Ray tracing enabled end-to-end
- CloudWatch dashboards: API latency, error rate, ingestion lag, DB CPU, backfill progress
- GitHub repository made public with README, architecture docs, deploy instructions

**Backfill milestone for Tranche 3:**
By the end of Week 13, the backfill will have been running for approximately 9 weeks.
At ~150,000 ledgers/hour sustained, that is ~45 million ledgers processed, covering approximately
**January 2018 to present** (8+ years of price history).

**The Tranche 3 review validates that backfill is progressing correctly, not that it is complete.**
The reviewer should confirm:
- `GET /backfill/status` returns `status: running`, `backfill_task_healthy: true`
- `earliest_data_available` ≤ 2018-01-01
- `estimated_hours_to_completion` shows a reasonable remaining estimate (expected: 4–8 weeks)
- `rate_ledgers_per_hour` is stable (variation <20% over the last 24 hours)
- OHLCV data for `?timeframe=all` on XLM returns data points from 2018 or earlier

The backfill continues running autonomously post-delivery. When `status` transitions to
`completed`, the `GET /backfill/status` endpoint records the `completed_at` timestamp. The team
will share a link to this endpoint with Stellar for post-delivery monitoring.

**Acceptance criteria:**
1. `GET /backfill/status` shows `status: running`, `backfill_task_healthy: true`,
   `earliest_data_available` ≤ 2018-01-01, `estimated_hours_to_completion` present and valid
2. OpenAPI spec passes `openapi-validator` lint with no errors; Swagger UI deployed
3. Onboarding portal accessible; self-service API key request flow functional
4. Integration test suite: all tests pass on CI (GitHub Actions link provided)
5. Load test report: p95 <100ms at 100 req/s confirmed
6. Security checklist signed off: no wildcard IAM, RDS has no public endpoint, all secrets
   in Secrets Manager, all inputs validated
7. GitHub repository public; `cdk deploy` from README works in a fresh AWS account
8. CloudWatch dashboard accessible to Stellar team (read-only IAM role); all alarms OK
9. 7-day post-launch monitoring report: uptime %, error rate, p95 latency, backfill rate

**Budget: $XX,XXX (Tranche 3)**

---

## 10. Cost Estimate (AWS, monthly)

### Monthly Running Cost (low traffic, post-backfill)

| Service | Estimated Cost | Notes |
|---------|---------------|-------|
| RDS PostgreSQL (db.t4g.micro, Single-AZ) | ~$12 | Prices API only — Block Explorer has its own RDS |
| ECS Fargate — Galexie | **$0** | Shared with Block Explorer (see Section 2.3) |
| Lambda — API handlers | ~$20 | Rust binaries, sub-ms cold starts |
| Lambda — Ingestion workers (~500K invocations) | ~$10 | |
| API Gateway (10M requests + 0.5 GB cache) | ~$50 | |
| NAT Gateway | **$0** | Shared with Block Explorer (see Section 2.3) |
| CloudWatch + X-Ray | ~$20 | |
| Secrets Manager + S3 (API docs) | ~$5 | |
| **Total (low traffic, post-backfill)** | **~$117/mo** | Down from ~$188/mo in prior design |

### Backfill Period Additional Costs (one-time, during 13-week project)

| Item | Configuration | One-time Cost |
|------|--------------|--------------|
| ECS Fargate — Soroban AMM backfill task | 1 vCPU / 2 GB RAM, a few hours (Tranche 1) | ~$2 |
| ECS Fargate — SDEX archive backfill task | 2 vCPU / 4 GB RAM, 13 weeks continuous (2,184 hours) | ~$216 |
| RDS during backfill | db.m6g.large ($131/mo, non-burstable dedicated CPU) for ~3 months; downgrade to db.t4g.micro after | ~$393 |
| S3 archive reads (Stellar public history) | ~57M files × minimal read cost | ~$25 |
| **Total one-time backfill compute** | | **~$636** |

### Scaled Up (high traffic)

| Service | Added Cost |
|---------|-----------|
| Upgrade to db.r6g.large + Multi-AZ | +~$350 |
| Add read replica | +~$175 |
| Add RDS Proxy | +~$25 |
| Add Lambda provisioned concurrency | +~$45 |
| **Total at scale** | **~$712/mo** |

---

## 11. Infrastructure Sharing with Soroban Block Explorer

This section explicitly enumerates all shared infrastructure to confirm there is no
double-billing between the two grants.

### 11.1 Shared Components (Block Explorer funded, Prices API does not charge)

| Component | Saving vs. standalone Prices API | How sharing works |
|-----------|----------------------------------|------------------|
| Galexie ECS Fargate task | ~$36/mo | Prices API Lambda registered as second S3 event target on the same bucket. One Galexie serves both services |
| S3 bucket `stellar-ledger-data/` | ~$2/mo | Same files read by both Lambdas; trivial additional S3 read cost |
| VPC (subnets, security groups, route tables) | ~$0 (one-time setup cost eliminated) | Prices API resources deploy into existing VPC |
| NAT Gateway | ~$35/mo | Single NAT Gateway handles egress for both services |
| ECS Fargate cluster | ~$0 | Cluster overhead shared; backfill tasks run in the same cluster |
| Block Explorer `soroban_events` table (read-only) | ~$0 (no extra RDS cost; read-only) | Soroban AMM backfill task queries the Block Explorer RDS to extract decoded swap events. Avoids re-reading ~8.5M ledgers from archives for the AMM stream. **Schema dependency**: documented below |
| **Monthly saving** | **~$71/mo** | |

### 11.2 Development Savings

| Artifact | Shared from Block Explorer | Saving |
|----------|--------------------------|--------|
| `stellar-xdr` Rust parsing crate | Written once, compiled into both Ledger Processors | ~5–7 dev days of XDR parsing logic not duplicated |
| `sqlx` migration patterns + database tooling | Shared Rust workspace and CI patterns | ~2–3 dev days |
| CDK VPC + IAM patterns | Reused CDK constructs | ~3–4 dev days |
| Observability configuration (CloudWatch dashboards, alarm patterns) | Copy-adapted from Block Explorer | ~1–2 dev days |

### 11.3 What Is Not Shared

The following components are **separate** and funded exclusively by the Prices API grant:
- Prices API RDS PostgreSQL instance (different schema: OHLCV, oracle prices)
- Prices API Lambda functions (separate function definitions, separate IAM roles)
- Prices API API Gateway + usage plans
- Prices API EventBridge rules
- Prices API Secrets Manager entries

### 11.4 Cross-Service Dependency and Risk

The Soroban AMM backfill task reads from the Block Explorer's `soroban_events` table
(read-only, within the shared VPC). This is the one point of coupling between the two services:

| Risk | Mitigation |
|------|-----------|
| Block Explorer `soroban_events` schema changes | The Soroban AMM backfill runs only once (Tranche 1) and completes in hours. Schema changes after it completes have no impact |
| Block Explorer DB has coverage gaps (indexing started late, missed some ledgers) | Gap detection: after the DB-sourced backfill, the task checks for contiguous OHLCV coverage from Soroban activation to present. Any gaps trigger a targeted archive-read for the missing ledger ranges |
| Block Explorer DB goes offline during the backfill window | Backfill is retried automatically; typical downtime during Tranche 1 is negligible |

The Prices API never writes to the Block Explorer's database. The Block Explorer never reads
from the Prices API database. Outside the Soroban AMM backfill window, the two services have
no runtime coupling.
