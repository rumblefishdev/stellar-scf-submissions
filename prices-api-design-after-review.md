# Prices API — Technical Design Document (Post-Review)

> This document supersedes `prices-api-design.md`. It incorporates changes required by reviewer
> feedback: the ingestion layer is rebuilt around self-hosted Galexie (direct ledger processing)
> replacing the deprecated Horizon API, component ownership is explicitly demarcated, and
> milestones are expanded with concrete acceptance criteria.

---

## 0. Deployment & AWS Account

The service will be deployed to a **dedicated AWS sub-account owned by Rumble Fish** within our
AWS Organizations. This keeps the project isolated from other Rumble Fish infrastructure while
giving us full control over deployment, monitoring, and maintenance.

Rumble Fish will **apply for a separate infrastructure grant** from Stellar to cover ongoing
AWS expenses and service maintenance costs. This keeps hosting and operational costs transparent
and decoupled from the development budget.

Infrastructure is managed via AWS CDK and deployed through a CI/CD pipeline (GitHub Actions).
The codebase is fully open source — Stellar retains the ability to fork and redeploy to their
own infrastructure at any time if needed.

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
                         │   Node.js / TypeScript    │
                         └────────────┬─────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │     RDS PostgreSQL        │
                         │  (db.t4g.micro, Single-AZ)│
                         └──────────────────────────┘
```

### 1.2 Data Ingestion Layer

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
│  ledgers/{seq_start}-            │
│         {seq_end}.xdr.zstd       │
└──────────────┬───────────────────┘
               │ S3 PutObject event notification
               ▼
┌──────────────────────────────────────────────────────┐
│  Lambda "Ledger Processor"  (event-driven, per file) │
│  1. Download + decompress XDR                        │
│  2. Parse LedgerCloseMeta via @stellar/stellar-sdk   │
│  3. Extract SDEX trades (ManageSellOfferResult /     │
│     ManageBuyOfferResult → offersClaimed[])          │
│  4. Extract Soroban swap events (CAP-67):            │
│     Soroswap, Aquarius, Phoenix — all in one stream  │
│  5. Write 1-min OHLCV candles to RDS                 │
└──────────────────────────────────────────────────────┘
               │
               ▼
       ┌───────────────┐      ┌───────────────────────┐
       │  RDS           │◄─────│  EventBridge-triggered │
       │  PostgreSQL    │      │  Lambda workers:       │
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

### 2.1 Components Hosted by Rumble Fish (AWS sub-account)

| Service | Role | Details |
|---------|------|---------|
| **ECS Fargate** | Galexie ledger exporter | 1 continuously running task (1 vCPU / 2 GB RAM). Connects to Stellar mainnet peers via Captive Core. Exports `LedgerCloseMeta` XDR to S3 on every ledger close (~every 5–6 s). Checkpoint-aware restart recovery |
| **S3** (ledger data) | Ledger file store | Receives `LedgerCloseMeta` files from Galexie. Triggers Lambda Ledger Processor via S3 event notifications. Short retention (files deleted after processing) |
| **S3** (API docs) | Documentation hosting | OpenAPI spec + self-service onboarding portal, served via CloudFront |
| **Lambda — Ledger Processor** | Primary ingestion | Event-driven (per S3 file). Parses XDR, extracts SDEX trades and Soroban swap events (Soroswap, Aquarius, Phoenix), writes 1-min OHLCV candles to RDS |
| **Lambda — Current Price Updater** | Price aggregation | EventBridge rate(1 min). Reads latest candles from `price_ohlcv`, computes VWAP across sources, writes to `current_prices` |
| **Lambda — OHLCV Rollup** | Candle aggregation | EventBridge rate(15 min). Rolls 1m → 15m, 1h, 4h, 1d, 1w, 1M |
| **Lambda — Oracle Fetcher** | Oracle cross-reference | EventBridge rate(5 min). Reads Reflector via Soroban RPC `simulateTransaction`. Writes to `oracle_prices`. Failures do not block primary ingestion |
| **Lambda — Asset Discovery** | Asset registry | EventBridge rate(1 hour). Detects new SEP-41 contract deployments and classic asset issuances from ledger account entries (`homeDomain` field in `LedgerCloseMeta`) |
| **Lambda — Cleanup Worker** | Data retention | EventBridge cron(02:00 UTC daily). Deletes expired fine-grained candles, drops old partitions, creates upcoming partitions |
| **Lambda — API handlers** | Public API | Individual functions per route group. Node.js 20, 256–512 MB, 15s timeout |
| **RDS PostgreSQL** | Primary data store | `db.t4g.micro` (2 vCPU burstable, 1 GB RAM), Single-AZ to start. Native range partitioning. Scale path in Section 6 |
| **API Gateway** | Public API entry point | REST API, usage plans, API key auth, rate limiting (100 req/s per key), request validation. Built-in response cache (0.5 GB) with per-endpoint TTLs |
| **EventBridge Scheduler** | Scheduled triggers | Cron/rate rules for all periodic Lambda workers |
| **Secrets Manager** | Credentials | DB password, Soroswap/Aquarius API keys, oracle contract address |
| **CloudWatch + X-Ray** | Observability | API latency, error rates, ingestion lag, Lambda duration/concurrency. X-Ray distributed tracing end-to-end |
| **CI/CD** | Deployment | GitHub Actions → `cdk deploy` on merge to main |

### 2.2 External Services Consumed (read-only)

| External service | Purpose | Failure impact |
|-----------------|---------|----------------|
| Stellar network peers (Galexie Captive Core) | Ledger data source | Critical — mitigated by connecting to multiple peers |
| Reflector Oracle (Soroban RPC) | Price cross-reference only — not a primary source | Non-critical — oracle column shows `null` or last known value |
| Soroswap API | Pool pair metadata and discovery | Non-critical — existing pairs continue; new pair discovery pauses |
| Aquarius API | Pool pair metadata and discovery | Non-critical — same as above |

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

### 3.5 Retention Policy (Cleanup Worker Lambda)
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

---

## 5. Data Ingestion Pipeline

### 5.1 Galexie Operation

Galexie is the SDF's Composable Data Platform exporter. We run it as a self-hosted ECS Fargate
task that connects to Stellar mainnet peers via Captive Core and writes one `LedgerCloseMeta` XDR
file to S3 on every ledger close (~every 5–6 seconds).

| Property | Value |
|----------|-------|
| Runtime | ECS Fargate, 1 vCPU / 2 GB RAM, continuously running |
| Output | `s3://stellar-ledger-data/ledgers/{seq_start}-{seq_end}.xdr.zstd` |
| Format | `LedgerCloseMeta` XDR, zstd-compressed |
| Recovery | Checkpoint-aware — resumes from last exported sequence on restart |
| Lag monitoring | CloudWatch alarm fires if S3 file timestamps fall >60s behind current ledger |

The Lambda Ledger Processor is triggered by S3 PutObject events. It downloads the file, parses it
using `@stellar/stellar-sdk` XDR types, and extracts:
- **SDEX trades** from `OperationResult` → `ManageSellOfferResult` / `ManageBuyOfferResult` →
  `offersClaimed[]` (price, base volume, quote volume, asset pair)
- **Soroban AMM swap events** (CAP-67) from `SorobanTransactionMeta` → `events` — this covers
  Soroswap, Aquarius, and Phoenix in a single stream

### 5.2 Ingestion Workers

| Worker | Trigger | Source | Data |
|--------|---------|--------|------|
| **Ledger Processor** | S3 PutObject event (per ledger, ~every 5–6 s) | `LedgerCloseMeta` from S3 | SDEX trades + all Soroban AMM swap events → 1-min OHLCV candles |
| **Oracle Fetcher** | EventBridge rate(5 min) | Reflector Oracle (Soroban RPC `simulateTransaction`) | Oracle reported prices → `oracle_prices` |
| **Asset Discovery** | EventBridge rate(1 hour) | Ledger account entries in `LedgerCloseMeta` | New classic asset issuances (issuer account, `homeDomain`); new SEP-41 contract deployments |
| **OHLCV Rollup** | EventBridge rate(15 min) | Internal DB | Roll up 1m candles → 15m, 1h, 4h, 1d, 1w, 1M |
| **Current Price Updater** | EventBridge rate(1 min) | Internal DB (after ingestion) | VWAP across sources → `current_prices` table |
| **Cleanup Worker** | EventBridge cron(02:00 UTC) | Internal DB | Delete expired fine-grained data, drop old partitions, create upcoming partitions |

### 5.3 EventBridge Scheduler Rules

```
ledger-processor:   S3 event (PutObject) → Lambda "ledger-processor"
oracle-ingest:      rate(5 minutes)       → Lambda "oracle-worker"
asset-discovery:    rate(1 hour)          → Lambda "discovery-worker"
ohlcv-rollup:       rate(15 minutes)      → Lambda "rollup-worker"
price-update:       rate(1 minute)        → Lambda "price-updater"
retention-cleanup:  cron(0 2 * * *)       → Lambda "cleanup-worker"
```

Note: the Ledger Processor is event-driven (S3 notification), not schedule-driven. All other
workers remain on EventBridge schedules.

### 5.4 VWAP Calculation Logic

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

### 5.5 Data Source Integration Details

#### Galexie / Ledger Processor (SDEX + Soroban AMMs)

- **Source:** `LedgerCloseMeta` XDR files written by Galexie to `s3://stellar-ledger-data/`
- **SDEX trades:** parsed from `OperationResult` metadata — `offersClaimed[]` entries in
  `ManageSellOfferResult` and `ManageBuyOfferResult` contain the matched price and volumes
- **Soroban AMM swaps:** parsed from `SorobanTransactionMeta.events` — CAP-67 `swap` and
  `transfer` events emitted by Soroswap, Aquarius, and Phoenix contracts
- **Library:** `@stellar/stellar-sdk` XDR types (Node.js)
- **Latency:** per-ledger processing, ~5–6 seconds from ledger close to DB write

#### Soroswap API (metadata only)

- **Base URL:** `https://api.soroswap.finance`
- **Auth:** API key (free registration)
- **Used for:** pool pair discovery (`getPools` → `token0`, `token1`); trade price data comes from
  ledger events, not this API

#### Aquarius API (metadata only)

- **Used for:** pool pair discovery and LP token metadata; trade price data comes from ledger events

#### Reflector Oracle

- **Type:** On-chain Soroban contract
- **Access:** Soroban RPC `simulateTransaction` (read-only, no fees)
- **Model:** ReflectorPulse (free, 5-min interval)
- **Role:** Cross-reference sanity check only — Reflector data feeds `oracle_prices` and the
  `/oracles/{asset}` endpoint. It does **not** feed `price_usd` or `vwap_24h`
- **Supported assets:** Major Stellar assets (XLM, USDC, EURC, BTC, ETH, etc.)

#### Asset Discovery

- **Classic assets:** Detected from ledger account entries in `LedgerCloseMeta` — new issuer
  accounts and trustline creations surface new classic assets. The `homeDomain` field is read
  directly from the account entry XDR
- **SAC derivation:** Each classic asset has a deterministic C-address via
  `stellar contract id asset`
- **Soroban tokens:** New SEP-41 contract deployments detected from ledger contract deployment
  events in `LedgerCloseMeta`

---

## 6. Performance & Scaling Strategy

### Target: <100ms p95 API response time

| Layer | Strategy |
|-------|----------|
| **API Gateway caching** | Built-in response cache (0.5 GB). Per-endpoint TTLs: `/assets` list 60s, `/ohlcv` 60s, `/price` 15s. Cache key includes query params. POST `/prices/batch` uncached |
| **API Gateway throttling** | Request throttling (100/s per API key, 1000/s global burst) |
| **Lambda** | Stateless functions, auto-scales to concurrency limit. Cold starts mitigated by small bundles (<5 MB), `@aws-sdk` v3 tree-shaking, lazy-loading non-critical deps |
| **Database** | Single writer instance to start. Direct Lambda→RDS connections (no proxy needed at low concurrency) |
| **Indexing** | Native range partitioning by month on `timestamp`. Composite index on `(asset_id, granularity, timestamp DESC)` per partition. Partition pruning eliminates irrelevant months |
| **Query optimization** | `current_prices` table avoids real-time aggregation. OHLCV queries hit pre-rolled-up data |

### RDS Sizing — Start Small, Scale Up

**Starting instance:** `db.t4g.micro` (2 vCPU burstable, 1 GB RAM) — ~$12/mo
- Supports ~80 max connections (enough for ~10 concurrent Lambda workers + API handlers)
- No RDS Proxy needed at this concurrency level

**Storage:** 20 GB gp3 initially, auto-scaling enabled up to 200 GB

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
| Language | TypeScript (Node.js 20+) |
| Runtime | AWS Lambda (Node.js 20.x) + ECS Fargate (Galexie) |
| API Framework | NestJS exposed with `@vendia/serverless-express` |
| Ledger processing | Galexie (SDF CDP exporter) + `@stellar/stellar-sdk` XDR types |
| Database | PostgreSQL 16 (native range partitioning, no extensions) |
| DB Connection | Direct Lambda→RDS (add RDS Proxy later if concurrency grows) |
| ORM | Drizzle ORM (lightweight, tree-shakeable) |
| Infrastructure | AWS CDK (TypeScript) |
| Bundler | esbuild (tree-shaking for minimal Lambda packages) |
| CI/CD | GitHub Actions → `cdk deploy` |
| Monitoring | CloudWatch Logs + Metrics + Alarms + X-Ray tracing |
| API Docs | OpenAPI 3.0 spec, auto-generated Swagger UI (hosted on S3 + CloudFront) |

---

## 9. Milestone Plan (~10 weeks)

### Milestone 1 — Infrastructure Foundation (Weeks 1–2)

**Work:**
- AWS CDK stack provisioned: ECS Fargate cluster (Galexie), S3 buckets (ledger data, API docs),
  Lambda execution roles, API Gateway, RDS, EventBridge, CloudWatch, Secrets Manager
- RDS PostgreSQL running with full schema from Section 3 (all tables, partitions for current +
  next 2 months, all indexes)
- Galexie ECS Fargate task running and actively writing `LedgerCloseMeta` files to S3
- GitHub Actions CI/CD pipeline: lint → test → `cdk diff` → `cdk deploy` on merge to main
- CloudWatch alarm: Galexie lag >60s triggers SNS notification (email / Slack webhook)

**Acceptance criteria:**
1. `cdk deploy` from a clean AWS account produces the full stack with no manual steps
2. S3 bucket `stellar-ledger-data/` contains at least 1,000 consecutive ledger files (~90 min of
   mainnet history) with timestamps matching mainnet ledger close times
3. RDS schema matches `prices-api-design-after-review.md` Section 3: all tables exist, partitions
   exist for current and next 2 months, all indexes present (verifiable via `\d+` psql output)
4. CI/CD pipeline link provided; at least one successful deploy run visible in GitHub Actions

---

### Milestone 2 — Data Ingestion (Weeks 3–4)

**Work:**
- Ledger Processor Lambda: S3 event-triggered, parses `LedgerCloseMeta` XDR, extracts SDEX trades
  and Soroban swap events (Soroswap, Aquarius, Phoenix), writes 1-min OHLCV candles to
  `price_ohlcv`
- OHLCV Rollup Lambda: rolls 1m → 15m, 1h, 4h, 1d via EventBridge schedule (every 15 min)
- Asset Discovery Lambda: detects new SEP-41 contract deployments and classic asset issuances
  from ledger account entries; populates `assets` table
- Oracle Fetcher Lambda: reads Reflector prices every 5 min, writes to `oracle_prices`; failures
  log to CloudWatch but do not block primary ingestion
- Current Price Updater Lambda: runs every 1 min via EventBridge, reads latest source candles
  from `price_ohlcv`, writes aggregated price to `current_prices` table (basic version — full
  VWAP formula wired in during Milestone 4)

**Acceptance criteria:**
1. After 24 hours of live operation: `price_ohlcv` contains continuous 1-min candles for at least
   20 major assets (XLM, USDC, EURC, AQUA, yXLM, BTC, ETH and their SAC equivalents), covering
   the full 24h period with no gaps >2 candles
2. 15m, 1h, 4h, 1d rollup candles present and mathematically correct (high = max of constituent
   1m highs, etc.) — verified via provided test script
3. `oracle_prices` table populated with Reflector readings every 5 min; at least 200 records per
   major asset (XLM, USDC, EURC) over the 24h period (Reflector at 5-min intervals yields ~288
   per asset per day)
4. Simulated new asset deployment on testnet → appears in `assets` table within 1 hour
5. Processing lag: Lambda processes each S3 file within 10 seconds of the S3 PutObject event
   (CloudWatch `Duration` metric confirms)

---

### Milestone 3 — Public API v1 (Weeks 5–6)

**Work:**
- Core API endpoints implemented and deployed behind API Gateway:
  - `GET /assets` (paginated, sortable, filterable)
  - `GET /assets/{asset_identifier}` (single asset)
  - `GET /assets/{asset_identifier}/price` (current price with `sources` breakdown)
  - `GET /assets/{asset_identifier}/ohlcv` (OHLCV with timeframe/granularity params)
- API Gateway: response caching (TTLs: `/assets` 60s, `/ohlcv` 60s, `/price` 15s), usage plans,
  API key issuance, throttling
- Input validation: asset identifier format enforced, param ranges validated, 400 on invalid input

**Acceptance criteria:**
1. All 4 endpoint groups return correct, schema-valid responses for at least 20 major assets
2. Load test (k6 or Locust, script provided): 100 req/s sustained for 5 minutes on
   `GET /assets/{id}/price` → p95 latency <200ms, error rate <0.1%
3. Cache confirmed: consecutive identical requests within TTL window return `X-Cache: Hit` header
4. Invalid inputs return `400` with descriptive error message — not 500
5. API key required: requests without a key return `403`

---

### Milestone 4 — Oracle Integration, VWAP, and Polish (Weeks 7–8)

**Work:**
- Aquarius integration: swap events arrive via the Galexie stream (same as Soroswap/Phoenix);
  this milestone validates Aquarius pool metadata and qualifies it as a named VWAP source
- Full VWAP formula wired into Current Price Updater Lambda (Section 5.4):
  `Weighted Price = Σ(source_price × source_volume_24h) / Σ(source_volume_24h)` across SDEX,
  Soroswap, Aquarius; sources below `min_volume_usd` threshold excluded
- Outlier detection: sources deviating >configurable% from inter-source median excluded from
  VWAP update cycle
- `POST /prices/batch` endpoint (multi-asset current price in one call)
- `GET /oracles/{asset_identifier}` endpoint (Reflector oracle cross-reference data)

**Acceptance criteria:**
1. Aquarius appearing as a named source in `sources` breakdown for assets with Aquarius pools
   (e.g., AQUA/XLM)
2. VWAP calculation verifiable: `price_usd` in `current_prices` matches
   `Σ(source_price × source_volume_24h) / Σ(source_volume_24h)` against the same raw rows —
   verified via provided SQL query
3. `?min_volume_usd=` query param: sources below the threshold absent from the `sources`
   breakdown and excluded from `price_usd`
4. Outlier exclusion: inject a test record where one source's price deviates >X% from the others
   → verify via DB that the source is excluded from that VWAP cycle
5. `POST /prices/batch` returns correct current prices for all requested assets in one response
6. `GET /oracles/{asset}` returns Reflector price records for at least XLM, USDC, EURC; oracle
   data does not appear in the `price_usd` field

---

### Milestone 5 — Documentation, Testing, and Hardening (Week 9)

**Work:**
- OpenAPI 3.0 specification covering all endpoints with request/response schemas, error codes,
  field descriptions
- Self-service onboarding portal (S3 + CloudFront): API key request form, quickstart guide,
  example queries, note on using `volume_24h_usd` and `vwap_24h` for safe collateral valuation
- Integration test suite (automated, runs in CI): covers all 6 endpoint groups with real data
  assertions
- Load test report: k6 or Locust, documented test plan, results at 100/s, 500/s, 1000/s
- Partition manager Lambda: creates next 2 months of partitions on the 1st of each month

**Acceptance criteria:**
1. OpenAPI spec passes `openapi-validator` lint with no errors
2. Swagger UI deployed and accessible at documented URL; all endpoints callable from the UI
3. Onboarding portal accessible at documented URL; self-service API key request flow functional
4. Integration test suite: all tests pass on CI (GitHub Actions link provided). Tests cover:
   correct OHLCV math, pagination cursor correctness, cache TTL behavior, error handling
5. Load test report provided: p50/p95/p99 at each load level; p95 <100ms at 100 req/s confirmed

---

### Milestone 6 — Production Launch (Week 10)

**Work:**
- Security review checklist: IAM least-privilege, no secrets in env vars, input sanitization,
  HTTPS-only, RDS not publicly accessible
- X-Ray tracing enabled end-to-end (API Gateway → Lambda → RDS)
- CloudWatch dashboards: API latency, error rate, ingestion lag (Galexie S3 file freshness),
  DB CPU/connections, current price staleness per asset
- GitHub repository made public with README, architecture docs, local dev setup, deploy
  instructions
- Handoff documentation to Stellar team

**Acceptance criteria:**
1. Security checklist signed off — minimum: no wildcard IAM permissions, RDS has no public
   endpoint, all secrets in Secrets Manager, all inputs validated
2. X-Ray service map shows end-to-end traces; latency breakdown per segment visible
3. GitHub repository public; `cdk deploy` from README works in a fresh AWS account
4. CloudWatch dashboard accessible to Stellar team (read-only IAM role provided); all alarms
   in OK state at launch
5. 7-day post-launch monitoring report: uptime %, error rate, p95 latency, Galexie ingestion lag

---

## 10. Cost Estimate (AWS, monthly)

### Starting (low traffic)
| Service | Estimated Cost |
|---------|---------------|
| RDS PostgreSQL (db.t4g.micro, Single-AZ) | ~$12 |
| ECS Fargate — Galexie (1 vCPU / 2 GB, continuous) | ~$36 |
| Lambda — API handlers | ~$20 |
| Lambda — Ingestion workers (~500K invocations) | ~$10 |
| API Gateway (10M requests + 0.5 GB cache) | ~$50 |
| NAT Gateway (single AZ, ~5 GB data) | ~$35 |
| CloudWatch + X-Ray | ~$20 |
| Secrets Manager + S3 | ~$5 |
| **Total** | **~$188/mo** |

### Scaled up (high traffic)
| Service | Added Cost |
|---------|-----------|
| Upgrade to db.r6g.large + Multi-AZ | +~$350 |
| Add read replica | +~$175 |
| Add RDS Proxy | +~$25 |
| Add Lambda provisioned concurrency | +~$45 |
| **Total at scale** | **~$783/mo** |

Start lean, scale components independently as traffic demands.
