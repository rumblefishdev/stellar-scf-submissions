# Stellar Block Explorer — Technical Design

A production-grade, Soroban-first block explorer for the Stellar network. The system prioritizes **human-readable transaction display** and first-class Soroban smart contract support. The frontend communicates exclusively with a custom NestJS REST API, which sources chain data from Stellar Horizon — either via Horizon's REST API or by querying Horizon's PostgreSQL database directly.

---

## 1. Frontend

### 1.1 Goals

- **Human-readable format** — Show exactly what occurred in each transaction. Users should understand payments, DEX operations, and Soroban contract calls without decoding XDR or raw operation codes.
- **Classic + Soroban** — Support both classic Stellar operations (payments, offers, path payments, etc.) and Soroban operations (invoke host function, contract events, token swaps).

### 1.2 Architecture

The frontend is a React application served via CloudFront CDN. It consumes the backend REST API with polling-based updates for new transactions and events.

```
┌────────┐     ┌──────────────────────────────────────────────────┐
│  User  │────>│  Global Search Bar                               │
│        │     │  (contracts, transactions, tokens, accounts, …)  │
│        │     └──────────────────────────────────────────────────┘
│        │
│        │     ┌─────────────────────────────────────────────────────────────┐
│        │────>│  React Router                                               │
└────────┘     │                                                             │
               │  /                          ── GET /network/stats ──────┐   │
               │  /transactions              ── GET /transactions ───────┤   │
               │  /transactions/:hash        ── GET /transactions/:hash ─┤   │
               │  /ledgers                   ── GET /ledgers ────────────┤   │
               │  /ledgers/:seq              ── GET /ledgers/:seq ───────┤   │
               │  /tokens                    ── GET /tokens ─────────────┤   │
               │  /tokens/:id                ── GET /tokens/:id ─────────┤   │
               │  /contracts/:id             ── GET /contracts/:id ──────┤   │
               │  /nfts                      ── GET /nfts ───────────────┤   │
               │  /nfts/:id                  ── GET /nfts/:id ───────────┤   │
               │  /liquidity-pools           ── GET /liquidity-pools ────┤   │
               │  /liquidity-pools/:id       ── GET /liquidity-pools/:id ┤   │
               │  /search?q=                 ── GET /search ─────────────┘   │
               │                                         │                   │
               └─────────────────────────────────────────┼───────────────────┘
                                                         │
                                                         ▼
                                                 ┌──────────────┐
                                                 │   REST API   │
                                                 └──────────────┘
```

### 1.3 Routes and Pages

| Route                    | Page            | Primary API Endpoint(s)                                                     |
| ------------------------ | --------------- | --------------------------------------------------------------------------- |
| `/`                      | Home            | `GET /network/stats`, `GET /transactions?limit=10`, `GET /ledgers?limit=10` |
| `/transactions`          | Transactions    | `GET /transactions`                                                         |
| `/transactions/:hash`    | Transaction     | `GET /transactions/:hash`                                                   |
| `/ledgers`               | Ledgers         | `GET /ledgers`                                                              |
| `/ledgers/:sequence`     | Ledger          | `GET /ledgers/:sequence`                                                    |
| `/tokens`                | Tokens          | `GET /tokens`                                                               |
| `/tokens/:id`            | Token           | `GET /tokens/:id`, `GET /tokens/:id/transactions`                           |
| `/contracts/:contractId` | Contract        | `GET /contracts/:contract_id`, `GET /contracts/:contract_id/interface`      |
| `/nfts`                  | NFTs            | `GET /nfts`                                                                 |
| `/nfts/:id`              | NFT             | `GET /nfts/:id`                                                             |
| `/liquidity-pools`       | Liquidity Pools | `GET /liquidity-pools`                                                      |
| `/liquidity-pools/:id`   | Liquidity Pool  | `GET /liquidity-pools/:id`                                                  |
| `/search?q=`             | Search Results  | `GET /search`                                                               |

#### Home (`/`)

Entry point and chain overview. Provides at-a-glance state of the Stellar network and quick access to exploration.

- Global search bar — accepts transaction hashes, contract IDs, token codes, account IDs, ledger sequences
- Latest transactions table — hash (truncated), source account, operation type, status badge, timestamp
- Latest ledgers table — sequence, closed_at, transaction count
- Chain overview — current ledger sequence, transactions per second, total accounts, total contracts

#### Transactions (`/transactions`)

Paginated, filterable table of all indexed transactions. Default sort: most recent first.

- Transaction table — hash, ledger sequence, source account, operation type, status badge (success/failed), fee, timestamp
- Filters — source account, contract ID, operation type
- Cursor-based pagination controls

#### Transaction (`/transactions/:hash`)

Both modes display the same base transaction details:

- Transaction hash (full, copyable), status badge (success/failed), ledger sequence (link), timestamp
- Fee charged (XLM + stroops), source account (link), memo (type + content)
- Signatures — signer, weight, signature hex

Two display modes toggle how **operations** are presented:

- **Normal mode** — graph/tree representation of the transaction's operation flow. Visually shows the relationships between source account, operations, and affected accounts/contracts. Each node in the tree displays a human-readable summary (e.g. "Sent 1,250 USDC to GD2M…K8J1", "Swapped 100 USDC for 95.2 XLM on Soroswap"). Soroban invocations render as a nested call tree showing the contract-to-contract hierarchy. Designed for general users exploring transactions.

- **Advanced mode** — targeted at developers and experienced users. Shows per-operation raw parameters, full argument values, operation IDs, and return values. Includes events emitted (type, topics, raw data), diagnostic events, and collapsible raw XDR sections (`envelope_xdr`, `result_xdr`, `result_meta_xdr`). All values are shown in their original format without simplification.

#### Ledgers (`/ledgers`)

Paginated table of all ledgers. Default sort: most recent first.

- Ledger table — sequence, hash (truncated), closed_at, protocol version, transaction count
- Cursor-based pagination controls

#### Ledger (`/ledgers/:sequence`)

- Ledger summary — sequence, hash, closed_at, protocol version, transaction count, base fee
- Transactions in ledger — paginated table of all transactions in this ledger
- Previous / next ledger navigation

#### Tokens (`/tokens`)

List of all known tokens (classic Stellar assets and Soroban token contracts).

- Token table — asset code, issuer / contract ID, type (classic / SAC / Soroban), total supply, holder count
- Filters — type (classic, SAC, Soroban), asset code search
- Cursor-based pagination controls

#### Token (`/tokens/:id`)

Single token detail view.

- Token summary — asset code, issuer or contract ID (copyable), type badge, total supply, holder count, deployed at ledger (if Soroban)
- Metadata — name, description, icon (if available), domain/home page
- Latest transactions — paginated table of recent transactions involving this token

#### Contract (`/contracts/:contractId`)

Contract details and interface.

- Contract summary — contract ID (full, copyable), deployer account (link), deployed at ledger (link), WASM hash, SAC badge if applicable
- Contract interface — list of public functions with parameter names and types, allowing users to understand the contract's API without reading source code
- Invocations tab — recent invocations table (function name, caller account, status, ledger, timestamp)
- Events tab — recent events table (event type, topics, data, ledger)
- Stats — total invocations count, unique callers

#### NFTs (`/nfts`)

List of NFTs on the Stellar network (Soroban-based NFT contracts).

- NFT table — name/identifier, collection name, contract ID, owner, preview image (if available)
- Filters — collection, contract ID
- Cursor-based pagination controls

#### NFT (`/nfts/:id`)

Single NFT overview.

- NFT summary — name, identifier/token ID, collection name, contract ID (link), owner account (link)
- Media preview — image, video, or other media associated with the NFT
- Metadata — full attribute list (traits, properties)
- Transfer history — table of ownership changes

#### Liquidity Pools (`/liquidity-pools`)

Paginated table of all liquidity pools.

- Pool table — pool ID (truncated), asset pair (e.g. XLM/USDC), total shares, reserves per asset, fee percentage
- Filters — asset pair, minimum TVL
- Cursor-based pagination controls

#### Liquidity Pool (`/liquidity-pools/:id`)

- Pool summary — pool ID (full, copyable), asset pair, fee percentage, total shares, reserves per asset
- Charts — TVL over time, volume over time, fee revenue
- Pool participants — table of liquidity providers and their share
- Recent transactions — deposits, withdrawals, and trades involving this pool

#### Search Results (`/search?q=`)

Generic search across all entity types. For exact matches (transaction hash, contract ID, account ID), redirects directly to the detail page. Otherwise displays grouped results.

- Search input — pre-filled with current query, allows refinement
- Results grouped by type — transactions, contracts, tokens, accounts, NFTs, liquidity pools (with type headers and counts)
- Each result row — identifier (linked), type badge, brief context
- Empty state — "No results found" with suggestions

### 1.4 Shared UI Elements

Present across all pages:

- **Header** — logo, global search bar (searches contracts, transactions, tokens, accounts, pools, NFTs), network indicator (mainnet/testnet)
- **Navigation** — links to home, transactions, ledgers, tokens, contracts, NFTs, liquidity pools
- **Linked identifiers** — all hashes, account IDs, contract IDs, token IDs, pool IDs, and ledger sequences are clickable links to their respective detail pages
- **Copy buttons** — on all full-length identifiers
- **Relative timestamps** — "2 min ago" with full timestamp on hover
- **Polling indicator** — shows when data was last refreshed

### 1.5 Performance and Error Handling

- **Pagination** — all list views use cursor-based pagination sourced from the backend API, which in turn maps to Horizon's cursor model
- **Loading states** — skeleton loaders for all data-dependent sections; spinner for search
- **Error states** — clear error messages for network failures, 404s (unknown hash/account), and rate limit responses; retry affordances where appropriate
- **Caching** — the frontend relies on backend-level caching (CloudFront, API Gateway) rather than local state caching to ensure data freshness

---

## 2. Backend

### 2.1 Architecture

The backend is a NestJS application running on AWS Lambda behind API Gateway. It is a REST API. The backend does not perform chain indexing; it sources chain data from Stellar Horizon — either via Horizon's REST API or by querying Horizon's PostgreSQL database directly.

```
┌──────────┐    HTTPS    ┌─────────────┐              ┌──────────────────────┐
│  Client  │────────────>│ API Gateway │─────────────>│  Lambda (NestJS)     │
└──────────┘             └─────────────┘              │                      │
                                                      │  NestJS Modules:     │
                                                      │  ├─ Network ─────────┤
                                                      │  ├─ Transactions ────┤
                                                      │  ├─ Ledgers ─────────┤
                                                      │  ├─ Tokens ──────────┤
                                                      │  ├─ Contracts ───────┤
                                                      │  ├─ NFTs ────────────┤
                                                      │  ├─ Liquidity Pools ─┤
                                                      │  └─ Search ──────────┤
                                                      └──────────┬───────────┘
                                                                 │
                                            ┌────────────────────┴────────────────────┐
                                            │                                         │
                                            ▼                                         ▼
                                ┌──────────────────────┐              ┌──────────────────────┐
                                │  Horizon REST API    │              │  Horizon PostgreSQL  │
                                │                      │              │  (direct queries)    │
                                └──────────────────────┘              └──────────────────────┘
```

### 2.2 API Responsibilities and Boundaries

The backend serves as a **facade** over Horizon, adding:

- **Data normalization** — transforms Horizon's response structures into a consistent, frontend-friendly format (e.g. flattening nested fields, adding human-readable operation summaries, attaching event interpretations)
- **Soroban enrichment** — decorates contract invocations with metadata, function names, and structured interpretations that Horizon does not natively provide
- **Search** — unified search across transaction hashes, account IDs, contract IDs, and contract metadata using PostgreSQL full-text indexes
- **XDR decoding** — parses `envelope_xdr`, `result_xdr`, and `result_meta_xdr` using `stellar-sdk` and returns structured operation and result data

The backend does **not** own or duplicate chain history. Horizon (via its REST API or PostgreSQL database) is the authoritative source for all chain data. The frontend never communicates with Horizon directly — all requests go through the NestJS API.

### 2.3 Endpoints

**Base URL:** `https://api.soroban-explorer.com/v1`

#### Network

**`GET /network/stats`** — Chain overview: current ledger sequence, TPS, total accounts, total contracts.

#### Transactions

**`GET /transactions`** — Paginated list. Query params: `limit`, `cursor`, `filter[source_account]`, `filter[contract_id]`, `filter[operation_type]`.

**`GET /transactions/:hash`** — Full detail for a single transaction (supports both normal and advanced representations):

```json
{
  "hash": "7b2a8c...",
  "ledger_sequence": 12345678,
  "source_account": "GABC...XYZ",
  "successful": true,
  "fee_charged": 100,
  "operations": [
    {
      "type": "invoke_host_function",
      "contract_id": "CCAB...DEF",
      "function_name": "swap",
      "human_readable": "Swapped 100 USDC for 95.2 XLM on Soroswap"
    }
  ],
  "operation_tree": [...],
  "events": [...],
  "envelope_xdr": "...",
  "result_xdr": "..."
}
```

#### Ledgers

**`GET /ledgers`** — Paginated list of ledgers.

**`GET /ledgers/:sequence`** — Ledger detail including transaction count and linked transactions.

#### Tokens

**`GET /tokens`** — Paginated list of tokens (classic assets + Soroban token contracts). Query params: `limit`, `cursor`, `filter[type]` (classic/sac/soroban), `filter[code]`.

**`GET /tokens/:id`** — Token detail: asset code, issuer/contract, type, supply, holder count, metadata.

**`GET /tokens/:id/transactions`** — Paginated transactions involving this token.

#### Contracts

**`GET /contracts/:contract_id`** — Contract metadata, deployer, WASM hash, stats.

**`GET /contracts/:contract_id/interface`** — Public function signatures (names, parameter types, return types).

**`GET /contracts/:contract_id/invocations`** — Paginated list of contract invocations.

**`GET /contracts/:contract_id/events`** — Paginated list of contract events.

#### NFTs

**`GET /nfts`** — Paginated list of NFTs. Query params: `limit`, `cursor`, `filter[collection]`, `filter[contract_id]`.

**`GET /nfts/:id`** — NFT detail: name, token ID, collection, contract, owner, metadata, media URL.

**`GET /nfts/:id/transfers`** — Transfer history for a single NFT.

#### Liquidity Pools

**`GET /liquidity-pools`** — Paginated list of pools. Query params: `limit`, `cursor`, `filter[assets]`, `filter[min_tvl]`.

**`GET /liquidity-pools/:id`** — Pool detail: asset pair, fee, reserves, total shares, TVL.

**`GET /liquidity-pools/:id/transactions`** — Deposits, withdrawals, and trades for this pool.

**`GET /liquidity-pools/:id/chart`** — Time-series data for TVL, volume, and fee revenue. Query params: `interval` (1h/1d/1w), `from`, `to`.

#### Search

**`GET /search?q=&type=transaction,contract,token,account,nft,pool`** — Generic search across all entity types. Uses prefix/exact matching on hashes, account IDs, contract IDs, and asset codes. Full-text search on metadata via `tsvector`/`tsquery` and GIN indexes.

### 2.4 Caching Strategy

Caching operates at two levels:

- **CloudFront / API Gateway caching** — responses for immutable data (historical transactions, closed ledgers) are cached with long TTLs at the CDN edge. Mutable data (account balances, recent transactions) uses short TTLs (5–15 seconds) aligned with Stellar's ledger close interval.
- **Backend in-memory caching** — frequently accessed reference data (contract metadata, network stats) is cached in the Lambda execution environment with TTLs of 30–60 seconds to reduce Horizon API calls.

### 2.5 Fault Tolerance

- **Horizon unavailability** — the backend returns cached data where available and degrades gracefully (e.g. serving stale ledger data with a freshness indicator) when Horizon is unreachable. Health check endpoints report Horizon connectivity status.
- **Lambda cold starts** — mitigated via ARM/Graviton2 runtime and provisioned concurrency at higher traffic tiers.
- **Database connection pooling** — RDS Proxy manages connection pools to prevent exhaustion under burst traffic.

---

## 3. Infrastructure

### 3.1 System Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           SYSTEM ARCHITECTURE                             │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  STELLAR NETWORK         INDEXING (HORIZON)           DATA STORAGE        │
│  ┌────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐ │
│  │ Stellar Core       │  │ Stellar Horizon      │  │ Horizon PostgreSQL │ │
│  │ (Captive Core +    │─>│ (ingestion + API)    │─>│ (ledgers, tx, ops, │ │
│  │  history archives) │  │                      │  │  accounts, state)  │ │
│  └────────────────────┘  └────────────┬─────────┘  └──────────┬─────────┘ │
│                                       │                       │ read-only │
│  BACKEND                              │                       │           │
│  ┌──────────────────────┐             │                       │           │
│  │ API Gateway          │<─── React   │                       │           │
│  │   │                  │             │                       │           │
│  │   └─> NestJS REST API│<────────────┴───────────────────────┘           │
│  │                      │   (Horizon REST API + Horizon PostgreSQL)       │
│  └──────────────────────┘                                                 │
└───────────────────────────────────────────────────────────────────────────┘

Connections:
  Stellar Core + history archives --> Horizon (ingestion)
  Horizon --> Horizon PostgreSQL (write)
  NestJS --> Horizon REST API (standard queries)
  NestJS --> Horizon PostgreSQL read-only (direct queries)
  React --> API Gateway --> NestJS REST API
```

### 3.2 Deployment Model

All infrastructure runs on AWS. At launch the system is deployed in a **single Availability Zone** (`us-east-1a`) within a VPC, expanding to multi-AZ when SLA requirements demand it.

```
┌─ VPC — us-east-1a ──────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─ Public Subnet ───────────────────────────────────────────────────────┐  │
│  │  CloudFront CDN                 API Gateway                           │  │
│  └────────┬───────────────────────────┬──────────────────────────────────┘  │
│           │                           │                                     │
│  ┌─ Private Subnet ──────────────────────────────────────────────────────┐  │
│  │        │                           ▼                                  │  │
│  │        │                  Lambda (NestJS API)                         │  │
│  │        │                           │                                  │  │
│  │        │              ┌────────────┴────────────┐                     │  │
│  │        │              │ Horizon API              │                    │  │
│  │        │              │ Horizon PostgreSQL (RO)  │                    │  │
│  │        │              └─────────────────────────┘                     │  │
│  └────────┼──────────────────────────────────────────────────────────────┘  │
└───────────┼─────────────────────────────────────────────────────────────────┘
            │
  Route 53 ──> CloudFront CDN
  Lambda ····> Secrets Manager (credentials)
  Lambda ····> CloudWatch / X-Ray (monitoring)
```

### 3.3 Environments

| Environment     | Purpose                   | Horizon Target                             | Database                       |
| --------------- | ------------------------- | ------------------------------------------ | ------------------------------ |
| **Development** | Local and CI development  | Stellar testnet (public Horizon or local)  | Local PostgreSQL or none       |
| **Staging**     | Pre-production validation | Self-hosted Horizon on testnet             | Horizon PostgreSQL (read-only) |
| **Production**  | Live service              | Self-hosted Horizon on mainnet (or hosted) | Horizon PostgreSQL (read-only) |

Infrastructure is provisioned via IaC (Terraform or CDK) with environment-specific parameter overrides.

### 3.4 Tech Stack

**Data Layer**

All chain data originates from **Stellar Horizon**, which ingests from Stellar Core (via Captive Core) and history archives, then stores structured records in its own PostgreSQL database. The NestJS backend accesses this data through two paths:

- **Horizon REST API** — used for standard entity queries (ledgers, transactions, operations, accounts). Provides cursor-based pagination, SSE streaming, and well-structured JSON responses.
- **Horizon PostgreSQL (read-only)** — used for custom queries the REST API does not expose efficiently: full-text search, aggregations, Soroban-specific joins, and batch lookups. The backend never writes to this database.

**Application Layer**

| Component | Technology          | Purpose                                           |
| --------- | ------------------- | ------------------------------------------------- |
| Framework | NestJS / TypeScript | Modular REST API with excellent AWS integration   |
| Compute   | AWS Lambda          | Serverless; auto-scaling; Horizon runs separately |
| Gateway   | AWS API Gateway     | Rate limiting, API keys, request routing          |

**Infrastructure Layer**

| Component  | Technology        | Purpose                                 |
| ---------- | ----------------- | --------------------------------------- |
| CDN        | CloudFront        | Static asset delivery, response caching |
| DNS        | Route 53          | Domain management, health checks        |
| Monitoring | CloudWatch, X-Ray | Logging, distributed tracing, alarms    |
| Secrets    | Secrets Manager   | Database credentials, API keys          |

### 3.5 Scalability

| Component      | Mechanism                                    | Trigger              |
| -------------- | -------------------------------------------- | -------------------- |
| **API**        | Lambda auto-scale (up to 50 concurrent)      | On-demand            |
| **PostgreSQL** | RDS Proxy for connection pooling             | Default              |
|                | Materialized views for aggregated statistics | Default              |
| **CDN**        | CloudFront scales automatically              | N/A                  |
| **Multi-AZ**   | Expand VPC + enable RDS Multi-AZ             | SLA > 99.9% required |

### 3.6 Storage

- **Horizon PostgreSQL** — all chain data (ledgers, transactions, operations, accounts, effects). Owned and managed by Horizon; the NestJS API reads this database directly (read-only) for custom queries alongside using Horizon's REST API.
- **S3** — static frontend assets served via CloudFront.

### 3.7 Monitoring and Observability

- **CloudWatch Logs** — structured JSON logging from Lambda and API Gateway
- **X-Ray** — distributed tracing across API Gateway → Lambda → Horizon API calls, with correlation IDs propagated to error responses
- **CloudWatch Alarms** — Lambda error rate, API Gateway 5xx rate, Horizon ingestion lag (if self-hosted), RDS CPU/connection utilization

---

## 4. Indexing

### 4.1 How Stellar Horizon Works

Stellar Horizon is the canonical API server for the Stellar network. It connects to the Stellar network via **Captive Core** (an embedded instance of Stellar Core) and reads from **history archives** — append-only ledger snapshots published by network validators. Horizon ingests this raw network data, processes it, and stores structured records (ledgers, transactions, operations, effects, accounts, offers, trust lines) in its own PostgreSQL database. It then exposes this data through a REST API and Server-Sent Events (SSE) for streaming.

Horizon is both the **indexer** and the **API server**. The block explorer does not run a separate indexing pipeline.

### 4.2 Indexing Flow

```
  Stellar network (Core + history archives)
                    │
                    ▼
            ┌───────────────┐
            │   Horizon     │  ingestion (Captive Core + archives)
            │   ingestion   │  → build state from checkpoint
            │               │  → resume to tip, then live
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   Horizon     │  REST API + PostgreSQL
            │   PostgreSQL  │  (history_ledgers, history_transactions,
            └───────┬───────┘   history_operations, accounts, …)
                    │
                    ▼
            NestJS backend  →  Horizon API (primary)
                            →  Horizon PostgreSQL read-only (direct)
```

**Step-by-step process:**

1. **Start Horizon with ingestion enabled** (testnet or mainnet). Configure `HISTORY_ARCHIVE_URLS` and Captive Core for the target network. Apply database migrations, then start the Horizon process.
2. **Build state** — Horizon builds cumulative account state from the **latest checkpoint** available in the history archives.
3. **Catch up** — from that checkpoint + 1, Horizon ingests ledger by ledger up to the **network tip** via Captive Core. Initial catch-up can take hours depending on how far behind the archives are.
4. **Live ingestion** — new ledgers are ingested as they close (~every 5 seconds). After catch-up, the block explorer sees near-real-time data with only a few seconds of delay.

### 4.3 Data Indexed by Horizon

**Core chain data**

- **Ledgers** — `GET /ledgers`, `GET /ledgers/:sequence`
  - sequence, hash, closed_at, protocol_version, transaction_count
- **Transactions** — `GET /transactions`, `GET /transactions/:hash`
  - hash, source_account, fee_charged, successful, envelope_xdr, result_xdr
- **Operations** — `GET /operations`, `GET /transactions/:hash/operations`
  - type, details (JSONB), transaction reference
- **Effects** — `GET /effects`, `GET /operations/:id/effects`
  - type, account, details

**Account and asset state**

- **Accounts** — `GET /accounts/:id`
  - account_id, sequence, balances, signers, thresholds
- **Trust lines** — embedded in `GET /accounts/:id`
  - asset_type, asset_code, asset_issuer, balance, limit
- **Offers** — `GET /offers`, `GET /accounts/:id/offers`
  - offer_id, seller, selling, buying, amount, price
- **Liquidity pools** — `GET /liquidity_pools`, `GET /liquidity_pools/:id`
  - pool_id, fee, reserves, total_shares, total_trustlines

**Market data**

- **Trades** — `GET /trades`
  - base_asset, counter_asset, price, base_amount, time
- **Trade aggregations** — `GET /trade_aggregations`
  - base_asset, counter_asset, open, high, low, close
- **Order book** — `GET /order_book`
  - bids, asks per asset pair

### 4.4 Pagination and Cursors

Horizon uses **cursor-based pagination** with opaque paging tokens. The block explorer backend maps these directly:

- Each Horizon list response includes `_links.next` and `_links.prev` with cursor parameters
- The block explorer API exposes `cursor` and `limit` query parameters that are passed through to Horizon
- When querying Horizon's PostgreSQL directly, cursors are implemented using sequential IDs or timestamps to maintain a consistent pagination model

### 4.5 Historical Backfill

By default, Horizon starts from the latest checkpoint in the archives, meaning early ledger history is not available. If the block explorer must show **full history from ledger 1**:

1. **Backfill in chunks** — run `horizon db reingest range <from> <to>` repeatedly (e.g. `1 64000`, `64001 128000`, … up to the max available in the archives). The `to` sequence must not exceed the latest checkpoint published in the history archives.
2. **Isolate backfill** — run these commands without Horizon's normal ingestion process writing to the same database (stop Horizon or use a dedicated job).
3. **Resume live** — when backfill is complete up to the latest archive checkpoint, start Horizon with ingestion enabled so it resumes from that point to the tip and then stays live.

### 4.6 Near-Real-Time vs Historical Indexing

| Mode         | Mechanism                                      | Latency          | Use Case                   |
| ------------ | ---------------------------------------------- | ---------------- | -------------------------- |
| **Live**     | Captive Core streams new ledgers as they close | ~5 seconds       | Current network activity   |
| **Catch-up** | Horizon ingests from last checkpoint to tip    | Minutes to hours | Initial deployment         |
| **Backfill** | `db reingest range` from history archives      | Hours to days    | Full history from ledger 1 |

History archives can only be ingested up to their **latest published checkpoint**. Live data comes directly from Captive Core, so "latest blocks" are available with minimal delay once Horizon is caught up.

### 4.7 Idempotency and Re-sync

Horizon's ingestion is **idempotent** — re-ingesting a ledger range that was already processed produces the same result. This means:

- Failed or interrupted backfill runs can be safely retried
- If Horizon loses its database, it can be rebuilt from the archives and Captive Core
- The block explorer does not need to implement its own reorg safety or rollback logic; Stellar's consensus model (SCP) does not produce chain reorganizations in the way proof-of-work chains do. Once a ledger closes, it is final.

### 4.8 Block Explorer Consumption

- **Horizon REST API:** NestJS calls Horizon's REST API for standard queries (ledgers, transactions, operations, accounts).
- **Horizon PostgreSQL (direct):** For custom analytics, Soroban-specific views, or queries that the Horizon API does not expose well, NestJS queries Horizon's PostgreSQL directly (`history_ledgers`, `history_transactions`, etc.). Access is read-only — the block explorer never writes to Horizon's database.
- **XDR decoding:** XDR fields returned by either path are decoded in the backend using `stellar-sdk`.

The NestJS API chooses the appropriate data source per endpoint. Chain data remains in Horizon as the single source of truth.

---

## 5. XDR Parsing

### 5.1 What is Stellar XDR

**XDR (External Data Representation)** is the binary serialization format used across the Stellar network. All on-chain data — transaction envelopes, operation results, ledger metadata, account state — is encoded in XDR before being written to the ledger. Horizon stores several XDR fields alongside decoded data:

- **`envelope_xdr`** — the full signed transaction envelope
  - Source account, sequence number, operations, memo, time bounds, signatures
- **`result_xdr`** — the outcome of the transaction
  - Success/failure status, per-operation result codes and details
- **`result_meta_xdr`** — ledger state changes caused by the transaction
  - Balance deltas, trust line changes, contract state mutations, Soroban events
- **`fee_meta_xdr`** — fee-related ledger changes
  - Fee charging state deltas applied before transaction execution

### 5.2 Parsing Strategy

XDR parsing is performed in the **backend** (NestJS) using `stellar-sdk`, which provides full XDR codec support for all Stellar types. The frontend receives pre-decoded structured data and does not need to parse XDR directly (though raw XDR is available in collapsible sections for advanced users).

**Transaction result XDR parsing:**

1. Decode `result_xdr` using `xdr.TransactionResult.fromXDR(base64String, 'base64')`
2. Extract the top-level result code (`txSuccess`, `txFailed`, `txBadSeq`, etc.)
3. For each operation in the result, extract the operation-specific result (e.g. `PaymentResult`, `ManageSellOfferResult`, `InvokeHostFunctionResult`)
4. Map result codes to human-readable status messages

**Operation result XDR parsing:**

1. Each operation result contains a discriminated union based on operation type
2. For classic operations (payment, offer, path payment), extract structured fields (amount, asset, destination, offer ID, etc.)
3. For Soroban operations (`InvokeHostFunctionResult`), extract the return value, contract events, and diagnostic events from `result_meta_xdr`
4. Soroban events (CAP-67) are decoded from the meta XDR to extract event type, topics, and data — these are the basis for human-readable interpretations (e.g. identifying token transfers, swaps, and state changes)

### 5.3 Data Extracted and Stored

- **`envelope_xdr`** — operation types, parameters, memo, signatures
  - Decoded and served via API; raw XDR preserved for advanced view
- **`result_xdr`** — per-operation success/failure, result details
  - Mapped to human-readable status; included in transaction response
- **`result_meta_xdr`** — Soroban events, state changes, return values
  - Decoded and served via API; event data queryable from Horizon PostgreSQL
- **`fee_meta_xdr`** — fee-related state changes
  - Decoded on demand; not persisted separately

### 5.4 Soroban-Specific XDR Handling

Soroban contract interactions produce rich metadata in `result_meta_xdr` that is not available through standard Horizon API fields:

- **Contract events** — emitted via `env.events().publish()` in Soroban contracts. Each event has a type (contract/system/diagnostic), topics (up to 4 XDR values), and a data payload. These are decoded and stored for the events tab and for generating human-readable interpretations.
- **Return values** — the return value of `invokeHostFunction` is an XDR `ScVal` that must be decoded to a typed value (integer, string, address, bytes, map, etc.).
- **Invocation tree** — complex transactions may contain nested contract-to-contract calls. The `result_meta_xdr` encodes the full invocation tree, which is decoded to show the call hierarchy in the explorer.

### 5.5 Error Handling for XDR

- **Malformed XDR** — if `fromXDR()` throws, the backend logs the error with the transaction hash for investigation and returns the raw base64 XDR to the frontend with a flag indicating decode failure. The transaction is still displayed with available non-XDR fields.
- **Unknown operation types** — new protocol versions may introduce operation types not yet supported by the SDK version in use. These are rendered as "Unknown operation" with raw XDR shown, and an alert is raised to update the SDK.
- **Schema evolution** — Stellar protocol upgrades may change XDR schemas. The block explorer pins `stellar-sdk` to a version compatible with the target network's protocol version and updates promptly after protocol upgrades.

---

## 6. Estimates

### 6.1 Effort Breakdown by Project Part

#### A. Design — 35–40 days (already estimated)

#### B. AWS Architecture Setup + Stellar Horizon Infrastructure

| Task                                                    | Days   |
| ------------------------------------------------------- | ------ |
| VPC, subnets, security groups, IAM roles (IaC)          | 4      |
| Lambda + API Gateway setup (NestJS deployment pipeline) | 4      |
| CloudFront CDN + Route 53 + TLS                         | 1      |
| Secrets Manager, CloudWatch, X-Ray                      | 2      |
| Horizon deployment (testnet) — Docker/EC2, Captive Core | 6      |
| Horizon deployment (mainnet) + monitoring               | 5      |
| CI/CD pipeline (GitHub Actions → AWS)                   | 4      |
| Staging + production environment parity                 | 4      |
| **Subtotal**                                            | **30** |

#### C. Data Indexing (Testnet + Mainnet)

| Task                                                               | Days   |
| ------------------------------------------------------------------ | ------ |
| Horizon testnet ingestion + verification                           | 5      |
| Horizon mainnet ingestion + catch-up                               | 5      |
| Historical backfill (`db reingest range`) — scripting + monitoring | 4      |
| Validate data completeness (ledgers, transactions, operations)     | 3      |
| Horizon PostgreSQL read-only access setup + query testing          | 2      |
| Ingestion lag monitoring + alerting                                | 2      |
| **Subtotal**                                                       | **20** |

#### D. Core API Endpoints (NestJS)

| Task                                                             | Days   |
| ---------------------------------------------------------------- | ------ |
| NestJS project scaffolding, module structure, TypeORM setup      | 3      |
| Network stats endpoint                                           | 1      |
| Transactions endpoints (list + detail + operation tree)          | 9      |
| Ledgers endpoints (list + detail)                                | 3      |
| Tokens endpoints (list + detail + transactions)                  | 5      |
| Contracts endpoints (detail + interface + invocations + events)  | 9      |
| NFTs endpoints (list + detail + transfers)                       | 5      |
| Liquidity Pools endpoints (list + detail + transactions + chart) | 6      |
| Search endpoint (full-text + prefix matching)                    | 4      |
| XDR decoding service (envelope, result, result_meta, fee_meta)   | 8      |
| Human-readable operation summaries + Soroban interpretation      | 7      |
| Cursor-based pagination (Horizon passthrough + direct SQL)       | 3      |
| Rate limiting, API key auth, error handling, health checks       | 3      |
| Caching layer (in-memory + CloudFront TTL configuration)         | 3      |
| **Subtotal**                                                     | **72** |

#### E. Frontend Components + API Integration

| Task                                                                 | Days   |
| -------------------------------------------------------------------- | ------ |
| React project scaffolding, routing, design system setup              | 3      |
| Shared components (header, nav, search bar, copy button, timestamps) | 3      |
| Home page (chain overview, latest transactions + ledgers)            | 2      |
| Transactions page (paginated table, filters)                         | 2      |
| Transaction detail page — normal mode (graph/tree view)              | 5      |
| Transaction detail page — advanced mode (raw data, XDR)              | 4      |
| Ledgers page (paginated table)                                       | 1      |
| Ledger detail page                                                   | 2      |
| Tokens page (list, filters)                                          | 2      |
| Token detail page (summary + transactions)                           | 2      |
| Contract detail page (summary + interface + invocations + events)    | 7      |
| NFTs page (list, filters)                                            | 2      |
| NFT detail page (media preview, metadata, transfers)                 | 5      |
| Liquidity Pools page (list, filters)                                 | 2      |
| Liquidity Pool detail page (summary + charts)                        | 5      |
| Search results page                                                  | 6      |
| Error states, loading skeletons, empty states                        | 3      |
| Polling, freshness indicators, responsive layout                     | 2      |
| **Subtotal**                                                         | **60** |

#### F. Testing

| Task                                           | Days   |
| ---------------------------------------------- | ------ |
| Unit tests — API endpoints (NestJS)            | 8      |
| Unit tests — XDR parsing + operation summaries | 5      |
| Load testing (1M baseline scenario)            | 4      |
| Bug fixing + stabilization buffer              | 15     |
| **Subtotal**                                   | **32** |

### 6.2 Summary

| Project Part                         | Days        |
| ------------------------------------ | ----------- |
| A. Design                            | 35–40       |
| B. AWS Architecture + Horizon Infra  | 30          |
| C. Data Indexing (Testnet + Mainnet) | 20          |
| D. Core API Endpoints                | 72          |
| E. Frontend Components + Integration | 60          |
| F. Testing                           | 32          |
| **Total (incl. design)**             | **249–254** |

### 6.3 Cost Estimation

#### Traffic Assumptions

Two traffic scenarios are modeled:

- **Low traffic (launch):** **1M API requests/month** (~0.4 req/sec average, ~2 req/sec peak). Reflects a new Stellar block explorer entering an ecosystem with established competitors (StellarExpert). Infrastructure costs dominate at this volume.
- **High traffic (growth):** **10M API requests/month** (~3.8 req/sec average, ~15 req/sec peak). Represents meaningful traction and sustained organic adoption.

The architecture scales between these levels by adding Lambda provisioned concurrency and an RDS read replica — no re-architecture required.

#### Monthly Costs — Low Traffic (1M requests)

At 1M requests/month, costs are dominated by fixed infrastructure (NAT, optional RDS for derived data). Horizon may be self-hosted (separate cost) or use a hosted provider.

| Service        | Configuration                                        | Monthly Cost                  |
| -------------- | ---------------------------------------------------- | ----------------------------- |
| API Gateway    | 1M requests                                          | $4                            |
| Lambda         | 800K invocations, 1024MB ARM                         | $5                            |
| RDS PostgreSQL | Optional, db.r6g.large Single-AZ (derived data only) | $190 (or $0 if no derived DB) |
| RDS Storage    | 500GB gp3 (if RDS used)                              | $115                          |
| CloudFront     | 10GB transfer                                        | $5                            |
| Data Transfer  | 10GB outbound                                        | $5                            |
| CloudWatch     | Logs, metrics                                        | $50                           |
| NAT Gateway    | 1x, 100GB                                            | $40                           |
| Other          | Secrets, Route53                                     | $50                           |
| Horizon        | Self-hosted or hosted (not in this table)             | —                             |

**Total (with optional RDS):** ~$465/month. Lower if no block-explorer RDS; Horizon cost is separate (self-hosted or third-party).

#### Scaling Path to High Traffic (10M requests)

| Change                               | Trigger           | Added Cost |
| ------------------------------------ | ----------------- | ---------- |
| Add Lambda provisioned (5 instances) | >2 req/sec avg    | +$75       |
| Add RDS read replica (db.r6g.large)  | Primary CPU >60%  | +$190      |
| Enable RDS Multi-AZ                  | SLA >99.9% needed | +$190      |
| Expand VPC to Multi-AZ               | With Multi-AZ RDS | +$35 (NAT) |
| API Gateway + Lambda growth          | Proportional      | +$30       |
| CloudFront / data transfer growth    | Proportional      | +$20       |

**Estimated cost at 10M requests/month:** ~$1,100 – $1,200/month.

### 6.4 Three-Phase Delivery Plan

#### Phase 1 — Infrastructure & Core Backend (Month 1)

- **AWS infrastructure setup** — VPC topology (public/private subnets, NAT, security groups), IAM least-privilege policies, IaC (Terraform/CDK), secrets management, CloudWatch dashboards + X-Ray tracing
- **Horizon setup** — Run Horizon (self-hosted or hosted); configure ingestion (history archives, Captive Core), testnet deployment, optional backfill with `db reingest range` in chunks
- **Database design** — Horizon's schema as source of truth; optional block-explorer DB schema for derived data (search, event interpretations), versioned migrations, testnet seed data
- **API specification** — OpenAPI 3.0 for all endpoints, cursor-based pagination contracts, rate limiting tiers
- **NestJS scaffolding** — Project structure, TypeORM setup, core modules (Transactions, Ledgers, Network stats), Horizon REST API integration, XDR decoding service (core)
- **CI/CD pipeline** — GitHub Actions → AWS deployment pipeline

#### Phase 2 — Feature Complete (Month 2)

- **Horizon mainnet deployment** — Mainnet ingestion, historical backfill, ingestion lag monitoring + alerting
- **Remaining API endpoints** — Tokens, Contracts, NFTs, Liquidity Pools, Search (full-text + prefix matching), human-readable operation summaries + Soroban interpretation
- **Caching + rate limiting + auth** — In-memory cache, CloudFront TTL, API key authentication, per-key/per-IP rate limiting
- **Frontend** — React SPA with all pages (Home, Transaction List/Detail, Ledger Detail, Account Detail, Contract Detail, Tokens, NFTs, Liquidity Pools, Search), shared components (header, search bar, linked identifiers, copy buttons, polling indicator)
- **CloudFront CDN deployment** with S3 origin

#### Phase 3 — Testing, Polish & Launch (Month 3)

- **Testing** — Unit tests (API + XDR parsing), E2E integration tests (Horizon → API → frontend), load testing (1M baseline, 10M stress test)
- **Security audit** — OWASP Top 10, CORS, API key leakage
- **Performance tuning** — Query plans, index optimization, Lambda cold start evaluation, cost optimization (right-size RDS, Reserved Instances)
- **Staging → production** — Environment setup, DNS/TLS (API Gateway + CloudFront), monitoring go-live (PagerDuty/Slack alerting, runbooks)
- **Bug fixing + stabilization**
- **Documentation** — API reference, explorer usage guide, stakeholder handoff

**Total timeline: 3 months.** Design (35–40 days) runs before or in parallel with Phase 1.

### 6.5 Risk Areas and Assumptions

- **AWS knowledge concentration**

  - Impact: infrastructure work depends on a narrow skill set; unavailability of the AWS-experienced developer blocks deployments
  - Mitigation: document all IaC decisions early; cross-train a second team member on AWS during Phase 1

- **Frontend blockchain learning curve**

  - Impact: transaction detail page (graph/tree view) and contract page require understanding Stellar data structures
  - Mitigation: prepare mock API responses with realistic Stellar data early; build frontend against mocks while learning the domain

- **Horizon API gaps for Soroban-specific data**

  - Impact: some endpoints (contract interface, NFT metadata, event interpretations) may require direct PostgreSQL queries or XDR parsing that Horizon's REST API does not cover
  - Mitigation: audit Horizon API coverage in Phase 1; plan for read-only DB fallback from the start

- **Historical backfill time on mainnet**

  - Impact: `db reingest range` for full mainnet history can take days; delays mainnet launch if full history is required
  - Mitigation: start backfill in Phase 1 as a background task; consider launching with recent history only and backfilling incrementally

- **Transaction graph/tree view complexity**

  - Impact: the normal-mode visualization (nested operation tree, Soroban call hierarchy) is the most complex frontend component
  - Mitigation: allocate 10 days (largest single frontend task); provide sample tree structures from the blockchain team; consider using an existing tree/graph library (e.g. React Flow)

- **NFT and Liquidity Pool data availability**

  - Impact: Stellar's NFT ecosystem is nascent; liquidity pool chart data requires aggregation not natively provided by Horizon
  - Mitigation: build these pages last (Phase 2); design API to gracefully return empty states; chart data aggregation may require a lightweight background worker

- **Conservative estimate assumption**
  - All estimates include buffer for context switching, code review, and ramp-up time
  - No parallel sprint overlap assumed within a single developer — each task is sequential per person
  - Design phase (35–40 days) is assumed to run before or overlap with Phase 1; not included in the 17-week execution timeline
