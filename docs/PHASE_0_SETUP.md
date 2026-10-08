# Phase 0 — Setup

> Repo · Docker Compose · DB Baseline · Design Patterns

**Status:** 🔴 Not Started  
**Goal:** Everything defined before any application code is written. By end of Phase 0, `docker compose up -d` brings the full infrastructure stack online and all 18 database tables exist.

---

## Steps

| Step | Title | Status |
|------|-------|--------|
| 0.1 | Monorepo Structure + Git Strategy | 🔴 |
| 0.2 | Agile Sprint Board | 🔴 |
| 0.3 | Docker Compose Dev Stack | 🔴 |
| 0.4 | Flyway Migration Baseline | 🔴 |
| 0.5 | Design Patterns Catalog | 🔴 |

---

## Step 0.1 — Monorepo Structure + Git Strategy

**Tech:** `Git` · `Linux` · `System Design`

### What it does
Single Git repository holds the complete system — all microservices, frontend, infrastructure configs, and docs. One `git clone`, one history, cross-service changes traceable in a single PR.

### Directory layout

```
wealthpilot/
├── frontend/wealthpilot-web/
├── services/
│   ├── api-gateway/
│   ├── auth-service/
│   ├── portfolio-service/
│   ├── market-data-service/
│   ├── analytics-service/
│   ├── notification-service/
│   ├── news-sentiment-service/
│   ├── ai-service/
│   ├── supply-chain-service/
│   ├── goals-service/
│   ├── document-service/
│   ├── report-service/
│   └── tax-service/
├── infrastructure/
│   ├── docker-compose.yml
│   ├── kafka/
│   ├── postgres/
│   ├── redis/
│   ├── elasticsearch/
│   ├── minio/
│   └── monitoring/
├── deployment/
│   ├── kubernetes/
│   └── helm/
├── docs/
│   ├── phases/
│   ├── services/
│   ├── infrastructure/
│   └── architecture/
└── README.md
```

### Git branching rules

| Branch type | Pattern | Rule |
|------------|---------|------|
| Production | `main` | Always deployable. CI required before merge. |
| Feature | `feature/<service>/<task>` | Max 3 days before merge or rebase. |
| Hotfix | `hotfix/<description>` | Branched from `main` only. |
| Release | `release/<version>` | Cut at end of each phase. |

**PR rule:** No direct push to `main`. Every PR must pass the Jenkins CI pipeline before merge.  
**Commit style:** `feat(auth): add refresh token rotation` — conventional commits so changelog is auto-generatable.

### Done when
- [ ] Repository exists on GitHub
- [ ] Directory structure matches layout above (empty directories with `.gitkeep`)
- [ ] `main` branch protection enabled (require PR + CI)
- [ ] Branch naming convention documented in `docs/CONTRIBUTING.md`

---

## Step 0.2 — Agile Sprint Board

**Tech:** `Agile/Scrum`

### What it does
GitHub Projects Kanban board maps every build step to a sprint story. Each phase is an Epic. Each step is a User Story with acceptance criteria matching the "Done when" checklist in this document.

### Board structure

```
Columns:
  Backlog → Sprint → In Progress → PR Open → Done
```

### Sprint rules

- **Sprint length:** 1 week (solo developer — short cycles = tight feedback)
- **Sunday:** Move stories from Backlog → Sprint for the coming week
- **Saturday:** Review. A story is Done only when the feature is usable end-to-end in the running app — not when code is merged
- **Never "done" without testing:** Every story requires at least one passing test as acceptance criterion

### Epics mapping

| Epic | Phase |
|------|-------|
| Foundation Setup | Phase 0 |
| Auth + Portfolio MVP | Phase 1 |
| Market Data + Alerts | Phase 2 |
| Analytics + News + Tax | Phase 3 |
| Calendar + Documents + Goals | Phase 4 |
| AI + Reports + Production | Phase 5 |
| Extended Features | Phase 6 |

### Done when
- [ ] GitHub Projects board created with all columns
- [ ] All Phase 0 and Phase 1 steps are created as stories with acceptance criteria
- [ ] Epics created for all 7 phases

---

## Step 0.3 — Docker Compose Dev Stack

**Tech:** `Docker` · `Linux` · `PostgreSQL` · `Redis` · `Apache Kafka` · `Elasticsearch`

### What it does
Single `docker-compose.yml` in `infrastructure/` spins up the complete infrastructure dependency layer. No application services here — only the data stores and messaging infrastructure that services depend on.

### Services in the compose file

| Container | Image | Port | Purpose |
|-----------|-------|------|---------|
| `postgres` | `timescale/timescaledb:latest-pg16` | 5432 | Primary DB + TimescaleDB extension |
| `redis` | `redis:7-alpine` | 6379 | Cache + rate limiting + sessions |
| `kafka` | `confluentinc/cp-kafka:7.7.0` | 9092 | Event streaming (KRaft mode, no Zookeeper) |
| `elasticsearch` | `docker.elastic.co/elasticsearch/elasticsearch:8.13.0` | 9200 | Search + log indexing |
| `minio` | `minio/minio:latest` | 9000 / 9001 | Object storage (S3-compatible) |
| `kibana` | `docker.elastic.co/kibana/kibana:8.13.0` | 5601 | Log dashboard (dev convenience) |

### Key config decisions

**PostgreSQL:** TimescaleDB image pre-installs the extension. Init script runs `CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE` on first start. Single `wealthpilot` database. Named volume `postgres_data` for persistence between restarts.

**Kafka:** KRaft mode (no Zookeeper). Environment: `KAFKA_PROCESS_ROLES=broker,controller`. `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true` for dev convenience (topics auto-created when first producer publishes). Production will use explicit topic creation.

**Elasticsearch:** `discovery.type=single-node` and `xpack.security.enabled=false` for local dev. Security enabled in production.

**MinIO:** Root user/password set via environment. On first start, creates `documents` and `reports` buckets. Console accessible at `:9001`.

**Health checks:** Every container defines a health check. Application services (added later) use `depends_on: condition: service_healthy`. This prevents a service from starting before its DB is ready.

**Shared network:** All containers on `wealthpilot-net` bridge network. Services reference each other by container name.

### Done when
- [ ] `docker compose up -d` starts all 6 containers without error
- [ ] All containers show `healthy` in `docker ps`
- [ ] Can connect to PostgreSQL: `psql -h localhost -U postgres -d wealthpilot`
- [ ] Can connect to Redis: `redis-cli ping` → `PONG`
- [ ] Kafka topic creation works: `kafka-topics.sh --create --topic test`
- [ ] Elasticsearch: `curl localhost:9200` → `{ "name": "...", "cluster_name": "..." }`
- [ ] MinIO console accessible at `http://localhost:9001`

---

## Step 0.4 — Flyway Migration Baseline

**Tech:** `PostgreSQL` · `SQL` · `Design Patterns`

### What it does
All database schema changes go through numbered Flyway SQL migration files. No manual `CREATE TABLE` ever. Schema is version-controlled, reproducible in CI, and safe to run on any environment without human intervention.

### Migration files location
```
services/{service-name}/src/main/resources/db/migration/
  V001__create_users.sql
  V002__create_portfolios.sql
  V003__create_instruments.sql
  ...
```

Each service owns its own migrations. The `flyway_schema_history` table in PostgreSQL tracks what version each environment is at.

### Tables to create (across all migration files)

**Identity (Auth Service)**
```
users                 — id UUID PK, email, name, password_hash, provider, base_currency, timezone
user_preferences      — user_id FK, theme, email_frequency, alert_channels JSONB
```

**Portfolio (Portfolio Service)**
```
portfolios            — id UUID PK, user_id FK, name, description, currency
holdings              — id UUID PK, portfolio_id FK, instrument_id FK, quantity, avg_cost_price
transactions          — id UUID PK, holding_id FK, type ENUM, quantity, price, fees, executed_at
transaction_lots      — id, transaction_id FK, remaining_qty, acquired_at   ← FIFO engine
```

**Instruments (shared)**
```
instruments           — id UUID PK, symbol, exchange, name, type ENUM, currency, metadata JSONB
```

**Market Data (Market Data Service)**
```
price_history         — instrument_id FK, ts TIMESTAMPTZ, open, high, low, close, volume  ← TimescaleDB hypertable
live_quotes           — instrument_id FK, price, change, change_pct, updated_at
dividends             — id UUID PK, instrument_id FK, ex_date, pay_date, amount, type ENUM
```

**Planning**
```
goals                 — id UUID PK, user_id FK, name, target_amount, target_date, linked_portfolio_ids UUID[]
sip_plans             — id UUID PK, user_id FK, instrument_id FK, amount, frequency ENUM, day_of_period
target_allocations    — id, portfolio_id FK, asset_class ENUM, target_pct
```

**Alerts + Events**
```
alerts                — id UUID PK, user_id FK, instrument_id FK, condition JSONB, channels TEXT[], active BOOL
market_events         — id UUID PK, type ENUM, title, event_date, related_tickers TEXT[], source
```

**AI + News**
```
news_articles         — id UUID PK, title, source, url, published_at, tickers TEXT[], sentiment_score FLOAT
documents             — id UUID PK, user_id FK, filename, file_path, mime_type, tags TEXT[], extracted_text
```

**Financial**
```
assets_liabilities    — id UUID PK, user_id FK, name, type ENUM, category ENUM, current_value
report_history        — id UUID PK, user_id FK, report_type, generated_at, sent_at, status
exchange_rates        — id, from_currency, to_currency, rate, fetched_at
```

### TimescaleDB hypertable conversion
After creating `price_history`, a migration step runs:
```sql
SELECT create_hypertable('price_history', 'ts', chunk_time_interval => INTERVAL '1 month');
SELECT add_compression_policy('price_history', INTERVAL '30 days');
```

### Done when
- [ ] `mvn flyway:migrate` runs with zero errors on the local dev database
- [ ] All 18 tables visible in `psql`: `\dt`
- [ ] `price_history` is a TimescaleDB hypertable: `SELECT * FROM timescaledb_information.hypertables`
- [ ] `flyway_schema_history` shows all migrations as `SUCCESS`

---

## Step 0.5 — Design Patterns Catalog

**Tech:** `Design Patterns` · `System Design`

### What it does
Documents the 9 design patterns applied across WealthPilot. This becomes an ADR file (`docs/architecture/DESIGN_PATTERNS.md`) and is the foundation for architecture discussions.

### Patterns in use

| Pattern | Where | Effect |
|---------|-------|--------|
| **Strategy** | `MarketDataProvider` interface | Switching Yahoo → Twelve Data = config change, zero code change |
| **Chain of Responsibility** | Provider cascade (Yahoo → Twelve Data → Alpha Vantage → Redis) | Each handler decides to handle or pass down |
| **Observer** | Kafka topics as event bus | Portfolio events broadcast to independent Analytics, Tax, Goals consumers |
| **Builder** | Complex DTOs (portfolio performance response, monthly report) | Fluent construction of multi-field objects |
| **Repository** | Spring Data JPA repositories | Persistence abstracted from service logic |
| **Factory** | `CostBasisCalculatorFactory` | Instantiates FIFO / LIFO / WeightedAverage based on user preference |
| **Circuit Breaker** | Resilience4j wrapping each external provider | CLOSED → OPEN → HALF-OPEN; prevents cascade failures |
| **Template Method** | `AbstractBrokerCSVParser` | Import algorithm fixed; column mapping overridden per broker |
| **Singleton** | FinBERT model (Python) | Model loaded once at startup; avoids 2–3s reload per request |

→ Full reference: [`docs/architecture/DESIGN_PATTERNS.md`](../architecture/DESIGN_PATTERNS.md)

### Done when
- [ ] `DESIGN_PATTERNS.md` written with each pattern, its location in the codebase, and why it was chosen over alternatives
- [ ] ADR entry created: `docs/adr/ADR-001-design-patterns.md`

---

## Phase 0 Exit Criteria

Before writing any application code:

- [ ] `docker compose up -d` → all 6 containers `healthy`
- [ ] All 18 database tables exist
- [ ] `price_history` is a TimescaleDB hypertable with compression policy
- [ ] GitHub repo exists with correct directory structure
- [ ] Sprint board configured with Phase 0 and Phase 1 stories
- [ ] `DESIGN_PATTERNS.md` completed

→ Next: [Phase 1 — Auth + Portfolio + Dashboard MVP](PHASE_1_MVP.md)
