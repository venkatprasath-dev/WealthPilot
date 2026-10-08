# Phase 2 — Market Data + WebSocket + Dividends + Alerts

> Weeks 7–12

**Status:** 🔴 Not Started  
**Goal:** Real prices on candlestick charts, WebSocket live feed, dividend tracking, multi-currency P&L, and working price alerts.

---

## Steps

| Step | Title | Service | Status |
|------|-------|---------|--------|
| 2.1 | TimescaleDB Hypertable + Price History API | Market Data | 🔴 |
| 2.2 | WebSocket Live Price Gateway | Market Data + WebSocket GW | 🔴 |
| 2.3 | Candlestick Charts (TradingView) | Frontend | 🔴 |
| 2.4 | Dividend Tracker | Market Data + Portfolio | 🔴 |
| 2.5 | Multi-Currency P&L + FX Impact | Analytics | 🔴 |
| 2.6 | Watchlist + Price Alert System | Notification | 🔴 |
| 2.7 | SIP / DCA Tracker + XIRR | Portfolio | 🔴 |

---

## Step 2.1 — TimescaleDB Hypertable + Price History API

**Tech:** `TimescaleDB` · `SQL` · `Query Optimization` · `JPA/Hibernate` · `PostgreSQL`

### Hypertable setup
`price_history` converted to TimescaleDB hypertable partitioned monthly. Compression activates after 30 days. Continuous aggregate `price_history_weekly` materializes weekly OHLCV from daily rows automatically.

### Query optimization
`EXPLAIN ANALYZE` run and documented for 5 critical queries:
1. 1-year daily close for 50 instruments (dashboard chart data)
2. Last EOD price for all user holdings (batch quote)
3. Portfolio value at arbitrary historical date
4. 5-year weekly aggregate for a single instrument
5. Latest price with fallback to last-known

Composite index on `(instrument_id, ts DESC)`. Partial index on `ts > NOW() - INTERVAL '7 days'` for recent queries.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/market/history/{symbol}?period=1Y&interval=1D` | Daily close from TimescaleDB |
| `GET` | `/api/v1/market/history/{symbol}?period=5Y&interval=1W` | Weekly from continuous aggregate |

### Done when
- [ ] TimescaleDB compression visible: `SELECT * FROM chunk_compression_stats('price_history')`
- [ ] 1-year daily query returns in < 100ms for 50 instruments (EXPLAIN ANALYZE verified)
- [ ] All 5 query plans documented in `docs/query-plans/`
- [ ] Continuous aggregate `price_history_weekly` refreshes automatically

---

## Step 2.2 — WebSocket Live Price Gateway

**Tech:** `Spring WebFlux` · `Apache Kafka` · `Redis` · `Event-Driven Architecture` · `Design Patterns`

### What it does
Pushes live price ticks to browsers via WebSocket. Separate Spring WebFlux application from the API Gateway.

### Subscription registry
`ConcurrentHashMap<String, Set<WebSocketSession>>` maps symbol → sessions. Reverse index maps session → symbols (for cleanup on disconnect). O(1) broadcast: Kafka tick → look up subscribers → push to all.

### Data flow
```
Market Data Service → polls provider every 15s per subscribed symbol
        ↓
Kafka: market.prices.live
        ↓
WebSocket Gateway: Kafka consumer → symbolSubscriberMap.get(symbol) → push to all sessions
        ↓ (same tick, same path)
Redis: SET quote:{symbol} EX 15   ← REST callers also benefit from live feed
```

### Active subscription tracking
When user subscribes via WebSocket → Gateway adds symbol to Redis set `active.subscriptions`. Market Data polling thread reads this set every 30s. When subscriber count hits 0 → Market Data stops polling that symbol.

### Done when
- [ ] Connect WebSocket → subscribe to AAPL → receive price tick within 15 seconds
- [ ] Disconnect → symbol eventually removed from active subscriptions
- [ ] 50 concurrent WebSocket connections: no thread pool exhaustion (WebFlux non-blocking)
- [ ] REST `GET /market/quote/{symbol}` also serves updated price after WebSocket push

---

## Step 2.3 — Candlestick Charts (TradingView)

**Tech:** `React 18` · `TypeScript` · `WebSocket` · `TimescaleDB`

### What it does
Interactive OHLC candlestick charts with real historical data and live tick updates.

### Data flow
1. User clicks a holding → fetch `GET /market/history/{symbol}?period=1Y&interval=1D`
2. Map `{ts, open, high, low, close}` → TradingView `CandlestickData[]`
3. Chart renders full history
4. Frontend subscribes via WebSocket → new ticks update last candle with `chart.update(tick)`

### Period handling
- < 1 day: intraday from provider (1m / 5m / 15m / 1H)
- 1D to 1Y: daily from `price_history` TimescaleDB
- > 1Y: weekly from `price_history_weekly` continuous aggregate

### Benchmark overlay
Toggle: overlay NIFTY 50 or S&P 500 on performance chart, both series normalized to 100 at first investment date.

### Done when
- [ ] Candlestick chart renders for any held symbol with 1Y daily data
- [ ] Period selector switches between 1D / 1W / 1M / 1Y / 5Y
- [ ] Live tick updates last candle without full re-render
- [ ] Benchmark overlay shows NIFTY 50 vs portfolio on same chart

---

## Step 2.4 — Dividend Tracker

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Apache Kafka` · `Quartz` · `REST API`

### What it does
Full dividend lifecycle: ex-dates, received history, yield, DRIP simulation, email alerts.

### Data model additions
```
dividends (
  id UUID PK, instrument_id FK,
  ex_date DATE, record_date DATE, pay_date DATE,
  amount NUMERIC, currency CHAR(3),
  type ENUM(REGULAR, SPECIAL, INTERIM),
  source ENUM(FINNHUB, MANUAL)
)
```

### Auto-fetch (Quartz)
Nightly at 02:00 IST: Finnhub dividend calendar API → upsert by `(instrument_id, ex_date)` unique constraint. Manually-edited records not overwritten.

### DRIP simulation
For each DIVIDEND transaction in history: calculate shares purchasable at `price_history` close on pay_date → accumulate → compare portfolio value under DRIP vs actual.

### Alert trigger
Market Data Service publishes to `dividends.upcoming` 3 days before ex-date. Notification Service: query which users hold that instrument → send email.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/analytics/dividends` | Dividend income history + yield per holding |
| `POST` | `/api/v1/holdings/{id}/dividends` | Manual dividend record |
| `GET` | `/api/v1/portfolios/{id}/drip-simulation` | DRIP vs actual comparison |
| `GET` | `/api/v1/analytics/dividends/calendar` | Upcoming ex-dates |

### Done when
- [ ] Dividend calendar shows correct upcoming ex-dates for held stocks
- [ ] Nightly Quartz job populates new dividend records
- [ ] DRIP simulation returns comparison value (verified against manual calculation for 1 holding)
- [ ] Email alert received 3 days before ex-date for a test holding

---

## Step 2.5 — Multi-Currency P&L + FX Impact

**Tech:** `Spring Boot 3` · `PostgreSQL` · `Redis` · `SQL` · `Query Optimization`

### What it does
Separates FX gain/loss from asset gain/loss for foreign-currency holdings.

### Formula
```
asset_pnl = qty × (current_price_native - avg_cost_price)
fx_pnl    = qty × current_price_native × (current_fx_rate - purchase_fx_rate)
total_pnl = asset_pnl × current_fx_rate + fx_pnl
```

Fetches exchange rates from `exchange_rates` table (Redis cache first, 1-hour TTL).

### API enrichment
`GET /portfolios/{id}/holdings` response includes per-holding: `asset_pnl`, `fx_pnl`, `total_pnl`, `asset_pnl_pct`, `fx_pnl_pct`, `current_fx_rate`, `purchase_fx_rate`.

### Done when
- [ ] US stock holding shows separate asset P&L and FX P&L
- [ ] Total P&L = asset P&L + FX P&L (verified manually)
- [ ] Base currency change → all values recompute correctly

---

## Step 2.6 — Watchlist + Price Alert System

**Tech:** `Spring Boot 3` · `Apache Kafka` · `Redis` · `PostgreSQL` · `Event-Driven Architecture`

### What it does
User-defined watchlists with PRICE_ABOVE / PRICE_BELOW / PCT_CHANGE / 52W_HIGH / 52W_LOW / MA_CROSS / VOLUME_SPIKE alert conditions. Delivery within 60 seconds of trigger.

### Alert condition as JSONB
```json
{"type": "PRICE_ABOVE", "value": 250}
{"type": "PCT_CHANGE", "value": -5, "period": "1D"}
{"type": "MA_CROSS", "period": 200, "direction": "BELOW"}
{"type": "VOLUME_SPIKE", "multiplier": 3, "avg_days": 30}
```
JSONB means new alert types require no schema migration.

### Evaluation engine
Notification Service consumes `market.prices.live`. Per tick:
1. Load active alerts for `instrument_id` (cached in Redis by instrument)
2. Pattern-match on `condition.type`
3. Compute condition-specific value (MA from Redis, 52W from TimescaleDB cached)
4. If condition matches: insert `alert_history`, set `last_triggered_at`, publish to `alerts.triggered`

Deduplication: after alert fires, cooldown stored in Redis (`alert:{id}:cooldown` → TTL 1 hour). Prevents repeated triggers on the same condition.

### Delivery
Kafka consumer on `alerts.triggered` → email via AWS SES + browser push (Web Push / VAPID). Target: < 60 seconds from trigger to delivery.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/alerts` | Create alert |
| `GET` | `/api/v1/alerts` | List user alerts |
| `PUT` | `/api/v1/alerts/{id}` | Update / toggle active |
| `DELETE` | `/api/v1/alerts/{id}` | Delete alert |
| `GET` | `/api/v1/alerts/notifications` | Notification history |

### Done when
- [ ] PRICE_ABOVE alert created → mock price tick crosses threshold → email received within 60 seconds
- [ ] Alert does not re-fire during 1-hour cooldown period
- [ ] 52W_HIGH alert evaluates correctly against TimescaleDB query
- [ ] Alert history table populated with trigger events

---

## Step 2.7 — SIP / DCA Tracker + XIRR

**Tech:** `Spring Boot 3` · `PostgreSQL` · `DSA` · `SQL` · `Quartz`

### What it does
Tracks systematic investment plans. XIRR measures true annualized return accounting for irregular timing of each installment.

### XIRR — Newton-Raphson (DSA component)

```
Find r such that: Σ (CF_i / (1+r)^(days_i/365)) = 0

Algorithm:
1. Initial guess: r = 0.1
2. f(r) = sum of discounted cash flows
3. f'(r) = numerical derivative (delta = 1e-6)
4. r_new = r - f(r) / f'(r)
5. Repeat until |r_new - r| < 1e-7 or max 200 iterations
```

Cash flows: each SIP installment = negative (money out). Current value of accumulated units = positive (money in today). XIRR result is the annualized return. Verified to match Excel XIRR output for identical inputs.

### Missed SIP alert
Quartz job daily at 09:00: check if any active SIP plan has a `day_of_period` matching today. If no SIP transaction recorded in last 3 days for that plan → publish `sip.missed` event → Notification Service emails user.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/sip-plans` | Create SIP plan |
| `GET` | `/api/v1/sip-plans` | List plans |
| `GET` | `/api/v1/sip-plans/{id}/xirr` | XIRR for this SIP |
| `PUT` | `/api/v1/sip-plans/{id}/pause` | Pause SIP |

### Done when
- [ ] XIRR result matches Excel for same cash flow inputs (test verified)
- [ ] SIP installment history shows correct units and NAV per date
- [ ] Missed SIP alert triggers if no installment recorded on due date

---

## Phase 2 Exit Criteria

- [ ] Candlestick charts render with real OHLCV data
- [ ] Live prices update via WebSocket without page reload
- [ ] Dividend calendar shows upcoming ex-dates for all held stocks
- [ ] Multi-currency holding shows asset P&L vs FX P&L separately
- [ ] Price alert fires within 60 seconds of threshold crossing
- [ ] XIRR matches Excel for a known SIP cash flow sequence

→ Next: [Phase 3 — Analytics + News Intelligence + Tax](PHASE_3_ANALYTICS.md)
