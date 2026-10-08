# Phase 4 — Calendar + Supply Chain + Documents + Goals

> Weeks 21–28

**Status:** 🔴 Not Started  
**Goal:** Event calendar, commodity supply chain tracking on a live map, document vault with AI extraction, and goal projections via Monte Carlo simulation.

---

## Steps

| Step | Title | Service | Status |
|------|-------|---------|--------|
| 4.1 | Market Event Calendar | Market Data + Notification | 🔴 |
| 4.2 | Supply Chain Service: Vessel + Cargo | Supply Chain | 🔴 |
| 4.3 | Document Service: Vault + AI Extraction | Document + AI | 🔴 |
| 4.4 | Goals Service: Monte Carlo Simulation | Goals | 🔴 |
| 4.5 | Net Worth Tracker | Portfolio | 🔴 |

---

## Step 4.1 — Market Event Calendar

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Quartz` · `REST API` · `Apache Kafka`

### What it does
Aggregates macroeconomic and corporate events into a unified calendar. Earnings, IPOs, central bank meetings, budget dates, ex-dividends, and user custom events in a single view.

### Data model

```
market_events (
  id UUID PK,
  type ENUM(EARNINGS, IPO, DIVIDEND, CB_MEETING, BUDGET, CUSTOM),
  title TEXT,
  event_date DATE,
  related_tickers TEXT[],
  source TEXT,
  description TEXT
)
```

### Data sources

| Source | Fetched by | Schedule |
|--------|-----------|---------|
| Finnhub earnings calendar | Quartz job | Weekly, next 30 days |
| Finnhub IPO calendar | Quartz job | Weekly |
| ECB / Fed / RBI / BoE meeting dates | Static seed, updated annually | N/A |
| Ex-dividend dates | JOIN from `dividends` table | N/A (already exist) |
| Indian Union Budget | Manual, flagged `source=MANUAL` | Annual |
| User custom events | `POST /api/v1/events/custom` | On demand |

Upsert strategy: `(instrument_id, event_date, type)` unique constraint. New data updates records, doesn't duplicate.

### Weekly digest
Quartz job every Sunday 07:00 → queries next 7 days' events for each user (filtered to held tickers for earnings/dividends) → Notification Service sends Monday morning digest email.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/events` | `?from=&to=&type=&tickers=` — filtered events |
| `GET` | `/api/v1/events/upcoming` | Next 7 days for held tickers |
| `POST` | `/api/v1/events/custom` | Add user custom event |
| `DELETE` | `/api/v1/events/custom/{id}` | Remove custom event |

### Done when
- [ ] Earnings calendar shows correct dates for held stocks (verified against Finnhub)
- [ ] ECB and RBI meeting dates visible in calendar
- [ ] Weekly digest email received every Monday with next 7 days' events
- [ ] Custom event added → appears in calendar immediately
- [ ] `GET /events?type=EARNINGS&tickers=INFY,TCS` returns only matching events

---

## Step 4.2 — Supply Chain Service: Vessel + Cargo Tracking

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Redis` · `REST API` · `Apache Kafka` · `Design Patterns`

### What it does
Tracks oil tankers, bulk carriers, and cargo aircraft as leading indicators for commodity prices. Live positions on a Leaflet map. Port congestion index as a supply pressure signal.

### Data model

```
vessel_tracking (
  id UUID PK,
  vessel_name TEXT, imo VARCHAR, mmsi VARCHAR,
  cargo_type ENUM(CRUDE_TANKER, BULK_GRAIN, BULK_METALS, CONTAINER),
  lat DECIMAL(9,6), lon DECIMAL(9,6),
  speed DECIMAL, destination TEXT, eta TIMESTAMPTZ,
  last_updated TIMESTAMPTZ
)

flight_tracking (
  id UUID PK, flight_number TEXT, airline TEXT,
  origin TEXT, destination TEXT,
  cargo_type TEXT, status TEXT,
  departure_time TIMESTAMPTZ, arrival_time TIMESTAMPTZ
)
```

### Data ingestion
Quartz job every 5 minutes → MarineTraffic API (or AIS data aggregator) → vessel positions fetched for configured cargo types → cache in Redis (`vessel:{imo}` TTL 5min) + snapshot to `vessel_tracking` table.

AviationStack job every 15 minutes → cargo flight data → `flight_tracking` table.

### Port congestion index
```sql
SELECT COUNT(*) as vessel_count, destination
FROM vessel_tracking
WHERE last_updated > NOW() - INTERVAL '6 hours'
  AND ST_Distance(
    ST_Point(lon, lat),
    ST_Point(port_lon, port_lat)
  ) < 50000  -- 50km radius
GROUP BY destination
```
Haversine distance used (DSA component). Stored in Redis `congestion:{port}`. If count > 30-day average for that port → publish `supply.signal.congestion` Kafka event.

### AI assistant integration
`get_supply_chain_context` tool calls this service's REST API → returns `{commodity, vessel_count, port_congestion_index}`. Lets Claude answer "What is the current crude oil supply situation?"

### Frontend (Leaflet)
Vessel positions as map markers. Clicking a marker → vessel name, cargo type, origin, destination, ETA, speed. Layer toggles: crude tankers / grain carriers / cargo aircraft.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/supply-chain/vessels` | Active vessel positions |
| `GET` | `/api/v1/supply-chain/vessels/{imo}` | Vessel detail |
| `GET` | `/api/v1/supply-chain/flights` | Cargo flight tracking |
| `GET` | `/api/v1/supply-chain/commodity/{type}` | Commodity flow (vessel count/day) |
| `GET` | `/api/v1/supply-chain/analytics` | Port congestion index per port |

### Done when
- [ ] Vessel map renders with real positions in Leaflet
- [ ] Clicking a vessel marker shows correct detail (name, destination, ETA)
- [ ] Port congestion index computed for Rotterdam, Singapore, Houston
- [ ] `get_supply_chain_context` tool returns correct vessel count for crude oil

---

## Step 4.3 — Document Service: Vault + AI Extraction

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Elasticsearch` · `Apache Kafka` · `MinIO` · `LLM Orchestration` · `Prompt Engineering` · `REST API`

### What it does
Secure file storage with AI-powered PDF extraction, auto-tagging by ticker, and full-text search.

### Upload flow

```
User uploads PDF
        ↓
Document Service: store in MinIO at /{userId}/{year}/{documentId}.pdf
        ↓
INSERT into documents table
        ↓
Publish to Kafka: documents.uploaded {documentId, userId, filePath}
```

### Extraction pipeline (Kafka consumer in AI Service)

```
Consume documents.uploaded
        ↓
PyMuPDF (fitz): extract raw text per page
        ↓
Claude API: structured extraction prompt → JSON {transactions: [...]}
        ↓
Update documents.extracted_text
        ↓
Update documents.tags with extracted ticker symbols
        ↓
Index into Elasticsearch: {filename, extracted_text, tags, uploaded_at}
```

### Per-user storage quota
Sum `file_size` for all documents by `user_id`. If > 1GB → reject upload with 413 + message suggesting deletion.

### Full-text search
`GET /api/v1/documents/search?q=tax+2025` → Elasticsearch `multi_match` on `filename` + `extracted_text`. Returns results ranked by relevance.

### Document-to-holding link
`POST /api/v1/documents/{id}/link` → associates document with `holding_id` or `transaction_id`. Stored in `document_links` junction table for traceability.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/documents/upload` | Upload PDF → MinIO |
| `GET` | `/api/v1/documents` | List documents with metadata |
| `GET` | `/api/v1/documents/{id}` | Document metadata + tags |
| `GET` | `/api/v1/documents/{id}/download` | Pre-signed MinIO URL |
| `DELETE` | `/api/v1/documents/{id}` | Delete document + MinIO object |
| `GET` | `/api/v1/documents/search` | Full-text search |
| `POST` | `/api/v1/documents/{id}/link` | Link to holding/transaction |

### Done when
- [ ] Upload Zerodha contract note PDF → appears in document list
- [ ] Tags field shows extracted ticker symbols (e.g., `["INFY", "TCS"]`)
- [ ] Full-text search `?q=Infosys` returns the uploaded document
- [ ] Pre-signed download URL works correctly
- [ ] Per-user quota enforced: upload > 1GB total → 413 rejected
- [ ] `document_links` record created when document linked to holding

---

## Step 4.4 — Goals Service: Monte Carlo Simulation

**Tech:** `Python FastAPI` · `PostgreSQL` · `Redis` · `DSA` · `System Design` · `NumPy` · `SciPy`

### What it does
Maps portfolios to financial goals and projects probability of success via 10,000 Monte Carlo paths.

### Data model

```
goals (
  id UUID PK, user_id FK,
  name TEXT, target_amount NUMERIC, target_date DATE, priority INT,
  linked_portfolio_ids UUID[],   -- PostgreSQL array
  current_value NUMERIC,
  status ENUM(ON_TRACK, AT_RISK, ACHIEVED)
)
```

### Monte Carlo algorithm (DSA / numerical methods)

```
Input: current_value, monthly_contribution, target_amount, target_date, risk_profile

1. Determine T = years to target date
2. Select μ (mean) and σ (std dev) from risk_profile:
   - Aggressive: μ=12%, σ=18%
   - Balanced:   μ=9%, σ=12%
   - Conservative: μ=7%, σ=5%

3. Run 10,000 simulations:
   For each simulation:
     portfolio_value = current_value
     For each month in T×12 months:
       monthly_return = draw from log-normal(μ/12, σ/sqrt(12))
       portfolio_value = portfolio_value × (1 + monthly_return) + monthly_contribution
     real_value = portfolio_value / (1.06)^T   ← inflation adjustment

4. Collect final_values[10000]
5. P10 = 10th percentile, P50 = 50th, P90 = 90th
6. Probability = count(final_value >= target) / 10000

7. Required contribution: binary search on monthly_contribution
   until probability >= 0.75
```

### Kafka-triggered current value update
Consumes `portfolio.events`. When holdings update → sum linked portfolio values → update `goals.current_value`. Recompute status:
- P50 projection ≥ target → ON_TRACK
- P50 projection < 80% of target → AT_RISK
- `current_value >= target_amount` → ACHIEVED

### Results caching
Redis: `goal:{goalId}:projection` → cached for 24 hours. Invalidated when user changes parameters.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/goals` | Create goal |
| `GET` | `/api/v1/goals` | List goals with current status |
| `GET` | `/api/v1/goals/{id}` | Goal detail |
| `PUT` | `/api/v1/goals/{id}` | Update goal / change parameters |
| `DELETE` | `/api/v1/goals/{id}` | Delete goal |
| `GET` | `/api/v1/goals/{id}/projection` | P10/P50/P90 curves + probability |
| `POST` | `/api/v1/goals/simulate` | Custom parameter simulation (no save) |

### Done when
- [ ] Create "Retirement at 50" goal with ₹5Cr target → projection runs in < 5 seconds
- [ ] P10/P50/P90 chart renders with 3 distinct curves
- [ ] Probability % non-zero and sensible (e.g., 68% for a realistic scenario)
- [ ] Status changes from ON_TRACK to AT_RISK when target amount increased significantly
- [ ] 10,000 simulations produce consistent P50 (within ±5% on repeated runs)

---

## Step 4.5 — Net Worth Tracker

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `REST API`

### What it does
Adds non-investment assets and liabilities to the portfolio view for a complete financial picture.

### Data model

```
assets_liabilities (
  id UUID PK, user_id FK,
  name TEXT,
  type ENUM(ASSET, LIABILITY),
  category ENUM(REAL_ESTATE, BANK, GOLD, LOAN, EMI, PPF, ESOP, OTHER),
  current_value NUMERIC, currency CHAR(3),
  notes TEXT, updated_at TIMESTAMPTZ
)
```

### Net worth formula
```
net_worth = sum(ASSET.current_value) + portfolio_current_value - sum(LIABILITY.current_value)
```
All values converted to base currency using `exchange_rates`.

### Net worth timeline
`GET /api/v1/analytics/net-worth/history` joins:
- `price_history × historical holdings` to reconstruct portfolio value at past dates
- `assets_liabilities` snapshot history (each manual update timestamped)

Returns monthly time-series for the "net worth over time" chart.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/networth` | Current net worth breakdown |
| `POST` | `/api/v1/networth/assets` | Add asset (real estate, bank account, gold) |
| `POST` | `/api/v1/networth/liabilities` | Add liability (loan, EMI) |
| `PUT` | `/api/v1/networth/{id}` | Update value |
| `DELETE` | `/api/v1/networth/{id}` | Remove entry |

### Done when
- [ ] Add house (REAL_ESTATE ASSET) → appears in net worth total
- [ ] Add home loan (LIABILITY) → deducted from net worth
- [ ] Net worth timeline chart renders with at least 6 months of data

---

## Phase 4 Exit Criteria

- [ ] Event calendar shows earnings and dividend dates for held stocks
- [ ] Weekly digest email received on Monday
- [ ] Vessel map renders with real positions
- [ ] Port congestion index computed
- [ ] PDF document uploaded → auto-tagged by ticker symbols
- [ ] Full-text search finds uploaded document
- [ ] Goal with Monte Carlo shows P10/P50/P90 curves
- [ ] Probability % shown on goal card
- [ ] Net worth total includes assets, liabilities, and portfolio value

→ Next: [Phase 5 — AI Assistant + Reports + Production](PHASE_5_AI_PRODUCTION.md)
