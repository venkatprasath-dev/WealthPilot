# Kafka Event Topology

**Broker:** Apache Kafka 3.7 (KRaft mode — no Zookeeper)  
**Total topics:** 10 + 2 DLQs

---

## Topic Index

| Topic | Partitions | Producer | Consumers |
|-------|-----------|---------|---------|
| `market.prices.live` | 12 | Market Data Service | WebSocket Gateway, Analytics Service, Notification Service |
| `market.prices.eod` | 4 | Market Data Service | TimescaleDB Writer, Analytics Service |
| `portfolio.events` | 4 | Portfolio Service | Analytics Service, Tax Service, Goals Service |
| `alerts.triggered` | 4 | Notification Service | Email Worker, Push Worker |
| `news.raw` | 6 | News Service | Sentiment Worker (Celery) |
| `news.raw.dlq` | 1 | Sentiment Worker | Manual inspection |
| `news.scored` | 4 | Sentiment Worker | Elasticsearch Writer, Notification Service, DB Writer |
| `documents.uploaded` | 2 | Document Service | AI Service (PDF extraction) |
| `reports.scheduled` | 2 | Quartz Scheduler | Report Service |
| `dividends.upcoming` | 2 | Market Data Service | Notification Service |
| `sip.missed` | 2 | Quartz Scheduler | Notification Service |
| `supply.signal.congestion` | 2 | Supply Chain Service | Notification Service |

---

## Topic Details

### market.prices.live
**Purpose:** Real-time price ticks for all subscribed instruments  
**Volume:** ~1 message per subscribed symbol per 15 seconds  
**Partitions:** 12 (highest throughput topic — partitioned by symbol hash)  
**Message:**
```json
{
  "symbol": "AAPL",
  "instrumentId": "uuid",
  "price": 218.45,
  "change": 2.30,
  "changePct": 1.06,
  "volume": 18500000,
  "ts": "2026-08-01T14:30:00Z",
  "source": "YAHOO"
}
```

**Consumer groups:**
- `websocket-gateway` — pushes ticks to subscribed browser sessions
- `analytics-realtime` — maintains in-memory portfolio value (Redis update)
- `alert-evaluator` — evaluates alert conditions per tick

Each consumer group maintains its own offset — same message read independently by all three.

---

### market.prices.eod
**Purpose:** End-of-day OHLCV snapshots for all instruments  
**Volume:** Once per day per instrument (~2,000 messages if 2,000 instruments tracked)  
**Message:**
```json
{
  "instrumentId": "uuid",
  "symbol": "AAPL",
  "date": "2026-08-01",
  "open": 216.10,
  "high": 219.80,
  "low": 215.50,
  "close": 218.45,
  "volume": 85000000,
  "adjClose": 218.45
}
```

**Consumer groups:**
- `timescaledb-writer` — bulk-inserts into `price_history` hypertable
- `analytics-eod` — invalidates analytics cache for affected portfolios

---

### portfolio.events
**Purpose:** All portfolio state changes. Key fan-out topic — three independent consumers.  
**Volume:** User-triggered (low frequency)  
**Message:**
```json
{
  "eventId": "uuid",
  "eventType": "HOLDING_UPDATED",
  "userId": "uuid",
  "portfolioId": "uuid",
  "holdingId": "uuid",
  "transactionType": "SELL",
  "timestamp": "2026-08-01T10:00:00Z"
}
```
`eventType` values: `HOLDING_CREATED`, `HOLDING_UPDATED`, `HOLDING_DELETED`, `TRANSACTION_ADDED`

**Consumer groups:**
- `analytics-portfolio` — invalidates analytics Redis cache (`DEL analytics:{userId}:*`)
- `tax-service` — on SELL events, classifies realized gain as STCG/LTCG
- `goals-service` — updates `goals.current_value` for linked portfolios

---

### alerts.triggered
**Purpose:** Fired alert events for delivery  
**Message:**
```json
{
  "alertId": "uuid",
  "userId": "uuid",
  "instrumentId": "uuid",
  "symbol": "AAPL",
  "triggerPrice": 251.00,
  "condition": {"type": "PRICE_ABOVE", "value": 250},
  "channels": ["email", "push"],
  "triggeredAt": "2026-08-01T14:32:00Z"
}
```

**Consumer groups:**
- `email-delivery` — sends via AWS SES
- `push-delivery` — sends Web Push notification

---

### news.raw
**Purpose:** Raw ingested articles before NLP processing  
**Partitions:** 6 (parallel FinBERT workers)  
**Message:**
```json
{
  "articleId": "uuid",
  "title": "Apple reports record Q3 earnings",
  "source": "FINNHUB",
  "url": "https://...",
  "publishedAt": "2026-08-01T09:00:00Z",
  "body": "...",
  "rawTickers": ["AAPL"]
}
```

**Consumer group:** `sentiment-worker` (Celery) — FinBERT inference per article

**DLQ (`news.raw.dlq`):** Articles failing FinBERT after 3 retries. Parked for manual inspection. Does not block main pipeline.

---

### news.scored
**Purpose:** Articles with sentiment scores, ready for indexing and alerts  
**Message:**
```json
{
  "articleId": "uuid",
  "title": "...",
  "source": "FINNHUB",
  "url": "https://...",
  "publishedAt": "2026-08-01T09:00:00Z",
  "tickers": ["AAPL"],
  "sentimentScore": 0.78,
  "sentimentLabel": "BULLISH",
  "summary": "Apple exceeded Q3 estimates, driven by services revenue growth."
}
```

**Consumer groups:**
- `elasticsearch-indexer` — indexes into `news-articles` Elasticsearch index
- `news-db-writer` — persists to `news_articles` PostgreSQL table
- `news-alert-checker` — if `sentimentScore < -0.7` for a held ticker → publish to `alerts.triggered`

---

### documents.uploaded
**Purpose:** Triggers PDF extraction pipeline  
**Message:**
```json
{
  "documentId": "uuid",
  "userId": "uuid",
  "filePath": "/userId/2026/documentId.pdf",
  "filename": "zerodha-tradebook-jul-2026.pdf",
  "uploadedAt": "2026-08-01T11:00:00Z"
}
```

**Consumer group:** `document-extractor` (AI Service) — PyMuPDF extract + Claude structured extraction

---

### reports.scheduled
**Purpose:** Triggers monthly report generation  
**Message:**
```json
{
  "userId": "uuid",
  "reportType": "MONTHLY",
  "period": "2026-07",
  "requestedAt": "2026-08-01T06:00:00Z"
}
```

**Consumer group:** `report-generator` (Report Service)

---

### dividends.upcoming
**Purpose:** Triggers email alert 3 days before ex-dividend date  
**Message:**
```json
{
  "instrumentId": "uuid",
  "symbol": "TCS",
  "exDate": "2026-08-05",
  "payDate": "2026-08-15",
  "amount": 22.00,
  "currency": "INR"
}
```

**Consumer group:** `dividend-alert` (Notification Service) — queries which users hold `instrumentId` → sends email

---

### sip.missed
**Purpose:** Alerts user when expected SIP installment not recorded  
**Message:**
```json
{
  "userId": "uuid",
  "sipPlanId": "uuid",
  "instrumentId": "uuid",
  "expectedDate": "2026-08-10",
  "amount": 5000.00
}
```

**Consumer group:** `sip-alert` (Notification Service)

---

### supply.signal.congestion
**Purpose:** Port congestion signal for commodity alert integration  
**Message:**
```json
{
  "port": "Rotterdam",
  "commodity": "crude_oil",
  "vesselCount": 47,
  "historicalAvg30d": 32,
  "congestionIndex": 1.47,
  "detectedAt": "2026-08-01T12:00:00Z"
}
```

**Consumer group:** `congestion-alert` (Notification Service) — alerts users holding relevant commodity positions

---

## Consumer Configuration (all topics)

```properties
# Reliability settings (no message loss on crash)
enable.auto.commit=false
max.poll.interval.ms=300000
session.timeout.ms=30000

# Error handling
auto.offset.reset=earliest

# Acknowledgement (all in-sync replicas must ack)
# Producer side: acks=all
```

All consumers use **manual offset commit** — offset committed only after successful processing. Message reprocessed on consumer crash. Idempotent consumers handle this correctly.

---

## DLQ Policy

| DLQ Topic | Retry attempts | Retry backoff | After retries exhausted |
|-----------|---------------|--------------|------------------------|
| `news.raw.dlq` | 3 | Exponential (1s → 2s → 4s) | Park in DLQ, alert to Slack (future) |

Main pipeline continues unblocked when article goes to DLQ.
