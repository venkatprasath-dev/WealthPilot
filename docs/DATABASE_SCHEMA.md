# Database Schema

**Primary:** PostgreSQL 16  
**Time-series:** TimescaleDB 2.x (extension on same PostgreSQL instance)  
**Migration tool:** Flyway  
**Total tables:** 18

---

## Table Index

| # | Table | Owner Service | Type |
|---|-------|--------------|------|
| 1 | `users` | Auth | Core entity |
| 2 | `user_preferences` | Auth | Core entity |
| 3 | `portfolios` | Portfolio | Core entity |
| 4 | `holdings` | Portfolio | Core entity |
| 5 | `transactions` | Portfolio | Core entity |
| 6 | `transaction_lots` | Portfolio | FIFO engine |
| 7 | `instruments` | Shared | Reference data |
| 8 | `price_history` | Market Data | **TimescaleDB hypertable** |
| 9 | `live_quotes` | Market Data | Cache mirror |
| 10 | `dividends` | Market Data | Market data |
| 11 | `alerts` | Notification | User config |
| 12 | `market_events` | Market Data | Calendar |
| 13 | `goals` | Goals | Planning |
| 14 | `sip_plans` | Portfolio | Planning |
| 15 | `news_articles` | News | AI/News |
| 16 | `documents` | Document | AI/Docs |
| 17 | `assets_liabilities` | Portfolio | Net worth |
| 18 | `report_history` | Report | Reporting |

---

## Schemas

### 1. users
```sql
CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           VARCHAR(255) UNIQUE NOT NULL,
  name            VARCHAR(255),
  password_hash   VARCHAR(255),
  provider        VARCHAR(20) NOT NULL DEFAULT 'LOCAL',
  base_currency   CHAR(3) NOT NULL DEFAULT 'INR',
  timezone        VARCHAR(50) DEFAULT 'Asia/Kolkata',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
```

### 2. user_preferences
```sql
CREATE TABLE user_preferences (
  user_id           UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  theme             VARCHAR(20) DEFAULT 'dark',
  email_frequency   VARCHAR(20) DEFAULT 'MONTHLY',
  alert_channels    JSONB DEFAULT '{"email": true, "push": false}',
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
```

### 3. portfolios
```sql
CREATE TABLE portfolios (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        VARCHAR(255) NOT NULL,
  description TEXT,
  currency    CHAR(3) NOT NULL DEFAULT 'INR',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_portfolios_user ON portfolios(user_id);
```

### 4. holdings
```sql
CREATE TABLE holdings (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id     UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  instrument_id    UUID NOT NULL,
  quantity         NUMERIC(20,8) NOT NULL,
  avg_cost_price   NUMERIC(20,8) NOT NULL,
  cost_currency    CHAR(3),
  buy_date         DATE,
  broker           TEXT,
  notes            TEXT,
  UNIQUE(portfolio_id, instrument_id)
);
CREATE INDEX idx_holdings_portfolio ON holdings(portfolio_id);
```

### 5. transactions
```sql
CREATE TABLE transactions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  holding_id   UUID NOT NULL REFERENCES holdings(id),
  type         VARCHAR(20) NOT NULL,
  quantity     NUMERIC(20,8),
  price        NUMERIC(20,8),
  fees         NUMERIC(10,4) DEFAULT 0,
  currency     CHAR(3),
  executed_at  TIMESTAMPTZ NOT NULL,
  notes        TEXT,
  import_hash  VARCHAR(64)  -- dedup hash for CSV imports
);
CREATE INDEX idx_transactions_holding ON transactions(holding_id);
CREATE INDEX idx_transactions_executed ON transactions(executed_at DESC);
CREATE UNIQUE INDEX idx_transactions_hash ON transactions(import_hash) WHERE import_hash IS NOT NULL;
```

### 6. transaction_lots (FIFO engine)
```sql
CREATE TABLE transaction_lots (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id  UUID NOT NULL REFERENCES transactions(id),
  holding_id      UUID NOT NULL REFERENCES holdings(id),
  remaining_qty   NUMERIC(20,8) NOT NULL,
  cost_price      NUMERIC(20,8) NOT NULL,
  acquired_at     TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_lots_holding_acquired ON transaction_lots(holding_id, acquired_at ASC);
```

### 7. instruments (shared reference table)
```sql
CREATE TABLE instruments (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  symbol    VARCHAR(50) NOT NULL,
  exchange  VARCHAR(50),
  name      VARCHAR(500) NOT NULL,
  type      VARCHAR(30) NOT NULL,  -- STOCK|MF|BOND|COMMODITY|FOREX|ETF|CRYPTO|REAL_ESTATE|GOLD
  currency  CHAR(3),
  country   CHAR(2),
  sector    TEXT,
  industry  TEXT,
  isin      VARCHAR(20),
  metadata  JSONB DEFAULT '{}',   -- type-specific: expense_ratio, coupon_rate, nav, lot_size, etc.
  created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_instruments_symbol_exchange ON instruments(symbol, exchange);
CREATE INDEX idx_instruments_type ON instruments(type);
-- metadata examples:
-- MF:      {"expense_ratio": 0.0115, "nav": 45.23, "amc": "Axis", "fund_manager": "Jinesh Gopani"}
-- BOND:    {"coupon_rate": 6.5, "face_value": 1000, "maturity_date": "2030-03-15", "credit_rating": "AAA"}
-- COMMODITY: {"lot_size": 1, "unit": "gram"}
```

### 8. price_history (TimescaleDB hypertable)
```sql
CREATE TABLE price_history (
  instrument_id  UUID NOT NULL,
  ts             TIMESTAMPTZ NOT NULL,
  open           NUMERIC(20,8),
  high           NUMERIC(20,8),
  low            NUMERIC(20,8),
  close          NUMERIC(20,8),
  volume         BIGINT,
  adj_close      NUMERIC(20,8),
  PRIMARY KEY (instrument_id, ts)
);

-- Convert to hypertable (monthly partitions)
SELECT create_hypertable('price_history', 'ts', chunk_time_interval => INTERVAL '1 month');

-- Compression: 10-20× reduction after 30 days
ALTER TABLE price_history SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'instrument_id',
  timescaledb.compress_orderby = 'ts DESC'
);
SELECT add_compression_policy('price_history', INTERVAL '30 days');

-- Continuous aggregate: weekly candles (materialized automatically)
CREATE MATERIALIZED VIEW price_history_weekly
WITH (timescaledb.continuous) AS
SELECT
  instrument_id,
  time_bucket('1 week', ts) AS week,
  FIRST(open, ts)  AS open,
  MAX(high)        AS high,
  MIN(low)         AS low,
  LAST(close, ts)  AS close,
  SUM(volume)      AS volume
FROM price_history
GROUP BY instrument_id, week;
```

### 9. live_quotes
```sql
CREATE TABLE live_quotes (
  instrument_id  UUID PRIMARY KEY,
  price          NUMERIC(20,8),
  change         NUMERIC(20,8),
  change_pct     NUMERIC(10,4),
  volume         BIGINT,
  bid            NUMERIC(20,8),
  ask            NUMERIC(20,8),
  updated_at     TIMESTAMPTZ
);
-- Primary source: Redis. Flushed to this table every 15 min for audit trail.
```

### 10. dividends
```sql
CREATE TABLE dividends (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  instrument_id UUID NOT NULL,
  ex_date       DATE NOT NULL,
  record_date   DATE,
  pay_date      DATE,
  amount        NUMERIC(10,4) NOT NULL,
  currency      CHAR(3),
  type          VARCHAR(20) DEFAULT 'REGULAR',  -- REGULAR|SPECIAL|INTERIM
  source        VARCHAR(20) DEFAULT 'FINNHUB',  -- FINNHUB|MANUAL
  UNIQUE(instrument_id, ex_date)
);
```

### 11. alerts
```sql
CREATE TABLE alerts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL,
  instrument_id   UUID NOT NULL,
  condition       JSONB NOT NULL,   -- {"type": "PRICE_ABOVE", "value": 250}
  channels        TEXT[] DEFAULT ARRAY['email'],
  active          BOOLEAN DEFAULT TRUE,
  last_triggered_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_alerts_instrument_active ON alerts(instrument_id) WHERE active = TRUE;
```

### 12. market_events
```sql
CREATE TABLE market_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type            VARCHAR(30) NOT NULL,  -- EARNINGS|IPO|DIVIDEND|CB_MEETING|BUDGET|CUSTOM
  title           TEXT NOT NULL,
  event_date      DATE NOT NULL,
  related_tickers TEXT[],
  source          TEXT,
  description     TEXT,
  user_id         UUID,  -- NULL for system events, set for CUSTOM type
  UNIQUE(type, event_date, COALESCE(related_tickers[1], ''))
);
CREATE INDEX idx_events_date ON market_events(event_date ASC);
```

### 13. goals
```sql
CREATE TABLE goals (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               UUID NOT NULL,
  name                  TEXT NOT NULL,
  target_amount         NUMERIC(20,2) NOT NULL,
  target_date           DATE NOT NULL,
  priority              INT DEFAULT 1,
  linked_portfolio_ids  UUID[],
  current_value         NUMERIC(20,2) DEFAULT 0,
  status                VARCHAR(20) DEFAULT 'ON_TRACK',  -- ON_TRACK|AT_RISK|ACHIEVED
  monthly_contribution  NUMERIC(20,2),
  created_at            TIMESTAMPTZ DEFAULT NOW()
);
```

### 14. sip_plans
```sql
CREATE TABLE sip_plans (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL,
  instrument_id   UUID NOT NULL,
  amount          NUMERIC(20,2) NOT NULL,
  frequency       VARCHAR(20) NOT NULL,  -- WEEKLY|MONTHLY|QUARTERLY
  day_of_period   INT NOT NULL,          -- day of month (1-28) or day of week (1-7)
  start_date      DATE NOT NULL,
  status          VARCHAR(20) DEFAULT 'ACTIVE',  -- ACTIVE|PAUSED|COMPLETED
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 15. news_articles
```sql
CREATE TABLE news_articles (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title            TEXT NOT NULL,
  source           VARCHAR(100),
  url              TEXT UNIQUE,
  published_at     TIMESTAMPTZ,
  tickers          TEXT[],
  summary          TEXT,
  sentiment_score  FLOAT,   -- -1.0 to +1.0
  sentiment_label  VARCHAR(20),  -- BULLISH|NEUTRAL|BEARISH
  sim_hash         BIGINT   -- locality-sensitive hash for dedup
);
CREATE INDEX idx_news_published ON news_articles(published_at DESC);
CREATE INDEX idx_news_tickers ON news_articles USING GIN(tickers);
```

### 16. documents
```sql
CREATE TABLE documents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL,
  filename        TEXT NOT NULL,
  file_path       TEXT NOT NULL,   -- MinIO path: /{userId}/{year}/{docId}.pdf
  mime_type       VARCHAR(100),
  tags            TEXT[] DEFAULT ARRAY[]::TEXT[],
  extracted_text  TEXT,
  file_size       BIGINT,
  uploaded_at     TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_documents_user ON documents(user_id);
CREATE INDEX idx_documents_tags ON documents USING GIN(tags);

CREATE TABLE document_links (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id     UUID REFERENCES documents(id),
  holding_id      UUID,
  transaction_id  UUID,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 17. assets_liabilities
```sql
CREATE TABLE assets_liabilities (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL,
  name           TEXT NOT NULL,
  type           VARCHAR(20) NOT NULL,  -- ASSET|LIABILITY
  category       VARCHAR(30) NOT NULL,  -- REAL_ESTATE|BANK|GOLD|LOAN|EMI|PPF|ESOP|OTHER
  current_value  NUMERIC(20,2) NOT NULL,
  currency       CHAR(3) NOT NULL,
  notes          TEXT,
  updated_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 18. report_history
```sql
CREATE TABLE report_history (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL,
  report_type  VARCHAR(50) NOT NULL,  -- MONTHLY|WEEKLY|QUARTERLY
  period       VARCHAR(20),           -- '2025-07' for monthly
  generated_at TIMESTAMPTZ,
  sent_at      TIMESTAMPTZ,
  status       VARCHAR(20) DEFAULT 'PENDING',  -- PENDING|GENERATED|SENT|FAILED
  file_path    TEXT   -- MinIO path to stored PDF
);
```

---

## Supporting Tables (not in main 18)

```sql
-- Exchange rates (fetched hourly from ECB/RBI)
CREATE TABLE exchange_rates (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_currency CHAR(3) NOT NULL,
  to_currency   CHAR(3) NOT NULL,
  rate          NUMERIC(20,8) NOT NULL,
  fetched_at    TIMESTAMPTZ NOT NULL,
  UNIQUE(from_currency, to_currency, fetched_at)
);

-- Target portfolio allocation
CREATE TABLE target_allocations (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id UUID NOT NULL REFERENCES portfolios(id),
  asset_class  VARCHAR(30) NOT NULL,
  target_pct   NUMERIC(5,2) NOT NULL,
  UNIQUE(portfolio_id, asset_class)
);

-- Broker column mappings (saved from manual mapping in CSV import)
CREATE TABLE broker_mappings (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  broker_name      VARCHAR(100) NOT NULL,
  original_column  VARCHAR(255) NOT NULL,
  internal_field   VARCHAR(100) NOT NULL,
  UNIQUE(broker_name, original_column)
);

-- Alert history (every time an alert fires)
CREATE TABLE alert_history (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  alert_id    UUID NOT NULL REFERENCES alerts(id),
  trigger_price NUMERIC(20,8),
  triggered_at  TIMESTAMPTZ DEFAULT NOW(),
  delivered   BOOLEAN DEFAULT FALSE
);

-- Reference rates (risk-free rate for Sharpe calculation)
CREATE TABLE reference_rates (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rate_type    VARCHAR(50),  -- 'INDIA_91D_TBILL', 'US_FED_FUNDS'
  rate         NUMERIC(8,4),
  effective_date DATE,
  UNIQUE(rate_type, effective_date)
);
```

---

## Index Strategy

**High-priority indexes (query-critical):**

```sql
-- Holdings: all queries filter by portfolio
CREATE INDEX idx_holdings_portfolio ON holdings(portfolio_id);

-- Transactions: history queries filter by holding + time
CREATE INDEX idx_transactions_holding_time ON transactions(holding_id, executed_at DESC);

-- Lots: FIFO queries are holding + time ordered
CREATE INDEX idx_lots_holding_acquired ON transaction_lots(holding_id, acquired_at ASC);

-- Price history: primary access pattern
CREATE INDEX ON price_history(instrument_id, ts DESC);  -- created by TimescaleDB automatically on hypertable

-- News: filter by ticker
CREATE INDEX idx_news_tickers ON news_articles USING GIN(tickers);

-- Alerts: evaluation engine filters by instrument + active
CREATE INDEX idx_alerts_instrument_active ON alerts(instrument_id) WHERE active = TRUE;
```

---

## Flyway Migration Order

```
V001__create_users_and_preferences.sql
V002__create_instruments.sql
V003__create_portfolios_and_holdings.sql
V004__create_transactions_and_lots.sql
V005__create_price_history_hypertable.sql      ← TimescaleDB conversion here
V006__create_live_quotes_and_dividends.sql
V007__create_alerts_and_events.sql
V008__create_goals_and_sip_plans.sql
V009__create_news_and_documents.sql
V010__create_assets_and_reports.sql
V011__create_supporting_tables.sql
V012__create_indexes.sql
V013__seed_reference_rates.sql                 ← Initial rate data
```
