# Prices API — Technical Design Document

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

## 1. Architecture Overview

```
                         ┌────────────────────────────────┐
                         │       AWS API Gateway          │
                         │  (REST API, rate limiting,     │
                         │   API keys, throttling,        │
                         │   built-in response caching)   │
                         └────────────┬───────────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │      AWS Lambda          │
                         │  (API handler functions) │
                         │   Node.js / TypeScript   │
                         └────────────┬─────────────┘
                                      │
                                      │
                         ┌────────────▼──────────────┐
                         │     Rds PostgreSQL        │
                         │  (db.t4g.micro, Single-AZ)│
                         └───────────────────────────┘

        ───── Data Ingestion (separate Lambda functions) ─────

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Horizon     │  │  Soroswap    │  │  Reflector   │  │  Soroban RPC │
  │  Trade Agg.  │  │  API         │  │  Oracle      │  │  (SEP-41)    │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │                 │
         └────────┬────────┴────────┬────────┘                 │
                  │                 │                          │
         ┌────────▼─────────────────▼──────────────────────────▼───┐
         │        CloudWatch EventBridge Schedule (cron)           │
         │            triggers Lambda functions                    │
         │           "Price Ingestion Workers"                     │
         └─────────────────────────┬───────────────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │  RDS PostgreSQL │
                          │                 │
                          └─────────────────┘
```

## 2. AWS Services Breakdown

| Service | Role | Details |
|---------|------|---------|
| **API Gateway** | Public API entry point + caching | REST API with usage plans, API key auth, rate limiting (e.g. 100 req/s per key), request validation. Built-in response cache (0.5 GB) with per-endpoint TTLs |
| **Lambda (API)** | API request handlers | Individual functions per route group (assets, prices, ohlcv, oracles). Node.js 20 runtime, 256–512 MB memory, 15s timeout |
| **Lambda (Workers)** | Data ingestion | Separate functions per data source. 512 MB–1 GB memory, up to 5 min timeout for heavy ingestion |
| **RDS PostgreSQL** | Primary data store | `db.t4g.micro` (2 vCPU, 1 GB RAM) Single-AZ to start. Native range partitioning for time-series. Scale up as needed (see Section 6) |
| **EventBridge Scheduler** | Scheduled ingestion | Cron/rate triggers for data ingestion at various frequencies (see Section 5) |
| **CloudWatch Logs/Metrics** | Observability | API latency, error rates, ingestion success/failure, Lambda duration/concurrency metrics |
| **S3** | Backup & bulk export | Daily DB snapshots, optional bulk CSV/JSON export of historical data |
| **Secrets Manager** | Credentials | DB passwords, API keys for Soroswap, oracle contract keys |

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

-- Create monthly partitions (auto-created by cleanup-worker Lambda ahead of time)
CREATE TABLE price_ohlcv_2026_01 PARTITION OF price_ohlcv
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE price_ohlcv_2026_02 PARTITION OF price_ohlcv
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... new partitions created monthly by the partition-manager Lambda

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
    sources         JSONB,                 -- {"sdex": 1.02, "soroswap": 1.01, "reflector": 1.015}
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

### 3.5 Retention Policy (cleanup-worker Lambda)
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

On subsequent requests, the server decodes the cursor and uses a **keyset condition** to seek
directly to the right position via index:
```sql
SELECT * FROM current_prices JOIN assets ON assets.id = current_prices.asset_id
WHERE (volume_24h, id) < (1523400.50, 42)  -- decoded from cursor
ORDER BY volume_24h DESC, id DESC
LIMIT 51;
```

`has_more` is determined by fetching `limit + 1` rows — if all `limit + 1` come back, `has_more: true`
and only `limit` items are returned. Otherwise `has_more: false` and no cursor.

Why cursor over offset/page number:
- **Performance** — keyset seek via index is O(1), while `OFFSET 10000` scans and discards 10000 rows
- **Consistency** — if new assets are inserted between pages, offset pagination causes duplicates or skipped rows; cursor is stable

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
        "reflector": "1.0000"
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
Current real-time price (latest snapshot).

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
    "sdex": {"price": "1.0001", "volume_24h": "800000"},
    "soroswap": {"price": "1.0002", "volume_24h": "500000"},
    "reflector": {"price": "1.0000", "timestamp": "2026-02-10T11:55:00Z"}
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
Oracle-specific prices for an asset.

**Response:**
```json
{
  "asset": "USDC:GA5ZSE...XYZ",
  "oracles": [
    {"name": "reflector", "price_usd": "1.0000", "updated_at": "2026-02-10T11:55:00Z"},
    {"name": "redstone", "price_usd": "1.0001", "updated_at": "2026-02-10T11:58:00Z"}
  ]
}
```

## 5. Data Ingestion Pipeline

### 5.1 Ingestion Workers (triggered by CloudWatch Events)

| Worker | Frequency | Source | Data |
|--------|-----------|--------|------|
| **SDEX Trade Aggregator** | Every 1 min | Horizon `GET /trade_aggregations` | OHLCV for all tracked pairs vs XLM/USDC |
| **Soroswap Pool Fetcher** | Every 1 min | Soroswap API `/quote` + pool reserves | Spot prices, pool volumes |
| **Aquarius Pool Fetcher** | Every 5 min | Aquarius AMM API | Pool reserves, LP token prices |
| **Oracle Fetcher** | Every 5 min | Reflector Oracle (Soroban RPC `simulateTransaction`) | Oracle reported prices |
| **Asset Discovery** | Every 1 hour | Horizon `GET /assets` + Soroban RPC contract scan | New assets/tokens |
| **OHLCV Rollup** | Every 15 min | Internal DB | Roll up 1m candles → 15m, 1h, 4h, 1d |
| **Current Price Updater** | Every 1 min | Internal DB (after ingestion) | Weighted avg → `current_prices` table |
| **Data Retention Cleanup** | Daily at 02:00 UTC | Internal DB | Delete expired fine-grained data |

### 5.2 EventBridge Scheduler Rules
```
sdex-ingest:        rate(1 minute)   → Lambda "sdex-worker"
soroswap-ingest:    rate(1 minute)   → Lambda "soroswap-worker"
aquarius-ingest:    rate(5 minutes)  → Lambda "aquarius-worker"
oracle-ingest:      rate(5 minutes)  → Lambda "oracle-worker"
asset-discovery:    rate(1 hour)     → Lambda "discovery-worker"
ohlcv-rollup:       rate(15 minutes) → Lambda "rollup-worker"
price-update:       rate(1 minute)   → Lambda "price-updater"
retention-cleanup:  cron(0 2 * * *)  → Lambda "cleanup-worker"
```

### 5.3 VWAP Calculation Logic
```
Weighted Price = Σ(source_price × source_volume_24h) / Σ(source_volume_24h)

Where sources = [SDEX, Soroswap, Aquarius, ...]
Only include sources where volume_24h > configurable_min_threshold_usd (e.g. $100)
```

Volume threshold is configurable per-request via `?min_volume_usd=` query param or defaults to system setting.

### 5.4 Data Source Integration Details

#### Horizon (SDEX)
- **Endpoint:** `GET https://horizon.stellar.org/trade_aggregations`
- **Params:** `base_asset_type`, `base_asset_code`, `base_asset_issuer`, `counter_asset_type`, `counter_asset_code`, `counter_asset_issuer`, `resolution` (ms: 60000, 300000, 900000, 3600000, 86400000, 604800000), `start_time`, `end_time`
- **Returns:** `timestamp`, `trade_count`, `base_volume`, `counter_volume`, `avg`, `high`, `low`, `open`, `close`
- **Rate limit:** Public Horizon ~100 req/min unauthenticated. We will request an API key from SDF for higher rate limits (this is a Stellar-funded RFP, so cooperation is expected). Ingestion will also be staggered across the minute to avoid burst spikes.

#### Soroswap
- **Base URL:** `https://api.soroswap.finance`
- **Auth:** API key (free registration)
- **Pool data:** `getPools` → returns `token0`, `token1`, `reserve0`, `reserve1`
- **Price quotes:** `/quote` endpoint
- **Covers:** Soroswap pools + aggregated routes across Phoenix, Aqua, SDEX

#### Reflector Oracle
- **Type:** On-chain Soroban contract
- **Access:** Call via Soroban RPC `simulateTransaction` (read-only, no fees)
- **Models:** ReflectorPulse (free, 5-min interval) and ReflectorBeam (paid, faster)
- **Use ReflectorPulse** for our ingestion — 5-min price updates are sufficient for oracle cross-reference
- **Supported assets:** Major Stellar assets (XLM, USDC, EURC, BTC, ETH, etc.)

#### Asset Discovery
- **Classic assets:** Horizon `GET /assets` → paginated list with `asset_code`, `asset_issuer`, `home_domain`
- **SAC derivation:** Each classic asset has a deterministic C-address via `stellar contract id asset`
- **Soroban tokens:** Monitor new contract deployments implementing SEP-41 interface via Soroban RPC event streaming

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
- Sufficient for development and early production with low query volume
- Supports ~80 max connections (enough for our ~10 concurrent Lambda workers + API handlers)
- No RDS Proxy needed at this concurrency level — Lambdas connect directly

**Storage:** 20 GB gp3 initially, auto-scaling enabled up to 200 GB

**Scaling path when traffic grows:**
| Trigger | Action |
|---------|--------|
| CPU credits running out regularly | Upgrade to `db.t4g.small` (2 GB RAM, ~$25/mo) or `db.t4g.medium` (4 GB, ~$50/mo) |
| Sustained >50% CPU usage | Move to `db.r6g.large` (2 vCPU dedicated, 16 GB RAM, ~$175/mo) |
| Connection count approaching limit | Add **RDS Proxy** (~$25/mo) to pool connections across Lambda invocations |
| Need high availability / zero downtime | Enable **Multi-AZ** (doubles RDS cost) |
| Read queries becoming bottleneck | Add **read replica** for API reads, primary for writes only |

All scaling steps are non-destructive — instance class changes and Multi-AZ can be enabled with a few minutes of downtime during a maintenance window.

## 7. Security Considerations

- **API keys** via API Gateway usage plans — required for all requests
- **Rate limiting** per API key to prevent abuse
- **Input validation:** Asset identifiers validated against known patterns (G-address: 56 chars starting with G, C-address: 56 chars starting with C)
- **Price manipulation protection:**
  - Outlier detection: Reject price data points deviating >X% from median across sources
  - Volume-weighted averaging reduces impact of low-liquidity manipulation
  - Oracle cross-reference as sanity check
- **HTTPS only** (enforced at API Gateway)
- **No PII stored** — only blockchain-public data

## 8. Tech Stack Summary

| Component | Technology |
|-----------|-----------|
| Language | TypeScript (Node.js 20+) |
| Runtime | AWS Lambda (Node.js 20.x runtime) |
| API Framework | NestJS exposed with `@vendia/serverless-express` |
| Database | PostgreSQL 16 (native range partitioning, no extensions) |
| DB Connection | Direct Lambda→RDS (add RDS Proxy later if concurrency grows) |
| ORM | TypeORM (lightweight, tree-shakeable — better for Lambda bundle size than Prisma) |
| Stellar SDK | `@stellar/stellar-sdk` (Horizon client + Soroban RPC) |
| Infrastructure | AWS CDK (TypeScript — same language as app code) |
| Bundler | esbuild (fast bundling, tree-shaking for minimal Lambda packages) |
| CI/CD | GitHub Actions → `cdk deploy` (Lambda + API Gateway + EventBridge) |
| Monitoring | CloudWatch Logs + Metrics + Alarms + X-Ray tracing |
| API Docs | OpenAPI 3.0 spec, auto-generated Swagger UI (hosted on S3) |

## 9. Milestone Plan (~10 weeks)

| Week | Milestone | Deliverables |
|------|-----------|-------------|
| 1–2 | **Infrastructure + DB** | AWS CDK stack (Lambda, API Gateway, RDS, EventBridge), RDS provisioned, partitioned schema deployed, CI/CD pipeline |
| 3–4 | **Data Ingestion v1** | SDEX worker Lambda (Horizon trade_aggregations), Soroswap worker Lambda, asset discovery, OHLCV rollup logic |
| 5–6 | **API v1** | Core API Lambdas (`/assets`, `/price`, `/ohlcv`), API Gateway integration with response caching, usage plans + API keys |
| 7–8 | **Oracle + Polish** | Reflector/oracle integration, VWAP logic, batch endpoint, outlier detection, Aquarius integration |
| 9 | **Documentation + Testing** | OpenAPI spec, load testing (<100ms p95 target), self-service onboarding portal, integration tests |
| 10 | **Production Launch** | Final security review, monitoring/alerting, X-Ray tracing, public launch, open-source repository |

## 10. Cost Estimate (AWS, monthly)

### Starting (low traffic)
| Service | Estimated Cost |
|---------|---------------|
| RDS PostgreSQL (db.t4g.micro, Single-AZ) | ~$12 |
| Lambda — API handlers | ~$20 |
| Lambda — Ingestion workers (scheduled, ~500K invocations) | ~$10 |
| API Gateway (10M requests + 0.5 GB cache) | ~$50 |
| NAT Gateway (single AZ, ~5 GB data) | ~$35 |
| CloudWatch + X-Ray | ~$20 |
| Secrets Manager + S3 | ~$5 |
| **Total** | **~$152/mo** |

### Scaled up (high traffic)
| Service | Added Cost |
|---------|-----------|
| Upgrade to db.r6g.large + Multi-AZ | +~$350 |
| Add read replica | +~$175 |
| Add RDS Proxy | +~$25 |
| Add Lambda provisioned concurrency | +~$45 |
| **Total at scale** | **~$747/mo** |

Start lean, scale components independently as traffic demands.
