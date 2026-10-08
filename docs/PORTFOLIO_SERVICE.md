# Portfolio Service

**Language:** Java 21  
**Framework:** Spring Boot 3  
**Phase:** 1  
**Status:** 🔴 Not Started

---

## Responsibility
Core investment data store. Manages portfolios, holdings, and transactions. FIFO/LIFO/Weighted-Average cost basis engine. Broker CSV import. Publishes all state changes to Kafka.

## Stack
| Component | Technology |
|-----------|-----------|
| Framework | Spring Boot 3 + Spring MVC |
| ORM | Spring Data JPA + Hibernate 6 |
| Primary DB | PostgreSQL 16 |
| Messaging | Apache Kafka (producer) |
| Cache | Redis (exchange rates for FX normalization) |
| CSV parsing | OpenCSV |
| Service discovery | Eureka client |

## Package Structure
```
com.wealthpilot.portfolio/
├── PortfolioServiceApplication.java
├── config/
│   ├── KafkaProducerConfig.java
│   └── SecurityConfig.java       -- trust X-User-Id header, no JWT re-validation
├── controller/
│   ├── PortfolioController.java
│   ├── HoldingController.java
│   ├── TransactionController.java
│   └── ImportController.java
├── service/
│   ├── PortfolioService.java
│   ├── HoldingService.java
│   ├── TransactionService.java
│   ├── CostBasisService.java      -- FIFO / LIFO / Weighted Average
│   └── ImportService.java         -- CSV parsing orchestration
├── repository/
│   ├── PortfolioRepository.java
│   ├── HoldingRepository.java
│   ├── TransactionRepository.java
│   └── TransactionLotRepository.java
├── entity/
│   ├── Portfolio.java
│   ├── Holding.java
│   ├── Transaction.java
│   └── TransactionLot.java        -- FIFO engine backing table
├── dto/
│   ├── PortfolioDTO.java
│   ├── HoldingDTO.java
│   ├── TransactionDTO.java
│   └── ImportPreviewDTO.java
├── kafka/
│   └── PortfolioEventProducer.java
├── integration/
│   ├── AbstractBrokerCsvParser.java     -- Template Method base class
│   ├── ZerodhaCsvParser.java
│   ├── GrowwCsvParser.java
│   └── IBKRCsvParser.java
├── factory/
│   └── CostBasisCalculatorFactory.java  -- Factory Pattern
└── exception/
    ├── PortfolioNotFoundException.java
    ├── HoldingNotFoundException.java
    └── InsufficientLotException.java
```

## Database Tables
```sql
portfolios (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL,
  name        VARCHAR(255) NOT NULL,
  description TEXT,
  currency    CHAR(3) DEFAULT 'INR',
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

holdings (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id     UUID REFERENCES portfolios(id),
  instrument_id    UUID NOT NULL,
  quantity         NUMERIC(20,8) NOT NULL,
  avg_cost_price   NUMERIC(20,8) NOT NULL,  -- recomputed on each BUY
  cost_currency    CHAR(3),
  buy_date         DATE,
  broker           TEXT,
  notes            TEXT
);

transactions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  holding_id   UUID REFERENCES holdings(id),
  type         VARCHAR(20) NOT NULL,  -- BUY|SELL|DIVIDEND|SPLIT|BONUS|SIP
  quantity     NUMERIC(20,8),
  price        NUMERIC(20,8),
  fees         NUMERIC(10,4) DEFAULT 0,
  currency     CHAR(3),
  executed_at  TIMESTAMPTZ NOT NULL,
  notes        TEXT
);

transaction_lots (                          -- FIFO engine
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id  UUID REFERENCES transactions(id),  -- BUY transactions only
  holding_id      UUID REFERENCES holdings(id),
  remaining_qty   NUMERIC(20,8) NOT NULL,
  acquired_at     TIMESTAMPTZ NOT NULL,
  cost_price      NUMERIC(20,8) NOT NULL
);
CREATE INDEX idx_lots_holding_acquired ON transaction_lots(holding_id, acquired_at ASC);
```

## FIFO Algorithm Detail

On each SELL transaction:
```
1. SELECT id, remaining_qty, cost_price, acquired_at
   FROM transaction_lots
   WHERE holding_id = ?
     AND remaining_qty > 0
   ORDER BY acquired_at ASC
   FOR UPDATE                          ← row-level lock prevents concurrent sell race

2. Iterate lots oldest-first:
   - If lot.remaining_qty >= remaining_sell_qty:
       consume remaining_sell_qty, update lot
       realized_gain += (sell_price - lot.cost_price) × remaining_sell_qty
       break
   - Else:
       consume full lot, set remaining_qty = 0
       realized_gain += (sell_price - lot.cost_price) × lot.remaining_qty
       remaining_sell_qty -= lot.remaining_qty

3. If remaining_sell_qty > 0 after all lots consumed: throw InsufficientLotException
```

## Kafka Event Schema

Topic: `portfolio.events`  
Published: **after** DB commit (not before)

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

Consumers: Analytics Service, Tax Service, Goals Service (independent consumer groups).

## CSV Import — Template Method Pattern

```
AbstractBrokerCsvParser:
  parse(file) → ImportPreviewDTO
    1. readFile()           -- common: OpenCSV reader
    2. detectFormat()       -- optional override
    3. mapColumns(row)      -- ABSTRACT: each broker overrides this
    4. validateRow(dto)     -- common: null checks, date parse
    5. deduplicateRow(dto)  -- common: hash check against existing transactions
    6. return preview rows

Concrete parsers: ZerodhaCsvParser, GrowwCsvParser, IBKRCsvParser
Each overrides only mapColumns(Row) → TransactionDTO
```

Deduplication hash: `MD5(symbol + executed_at.toString() + type + quantity + price)`

## API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/portfolios` | List user portfolios |
| `POST` | `/api/v1/portfolios` | Create portfolio |
| `GET` | `/api/v1/portfolios/{id}` | Get portfolio |
| `PUT` | `/api/v1/portfolios/{id}` | Update portfolio |
| `DELETE` | `/api/v1/portfolios/{id}` | Delete portfolio |
| `GET` | `/api/v1/portfolios/consolidated` | Merged view across all portfolios |
| `GET` | `/api/v1/portfolios/{id}/holdings` | List holdings |
| `POST` | `/api/v1/portfolios/{id}/holdings` | Add holding |
| `PUT` | `/api/v1/holdings/{id}` | Edit holding |
| `DELETE` | `/api/v1/holdings/{id}` | Delete holding |
| `GET` | `/api/v1/holdings/{id}/transactions` | Transaction history |
| `POST` | `/api/v1/holdings/{id}/transactions` | Record BUY / SELL / DIVIDEND / SPLIT |
| `POST` | `/api/v1/portfolios/{id}/import` | CSV import → preview (no commit) |
| `POST` | `/api/v1/portfolios/{id}/import/confirm` | Commit previewed rows |
| `GET` | `/api/v1/portfolios/{id}/allocation` | Asset allocation breakdown |
| `GET` | `/api/v1/portfolios/{id}/rebalance` | Rebalancing suggestions |
| `PUT` | `/api/v1/portfolios/{id}/target-allocation` | Set target allocation |

## Build Checklist
- [ ] CRUD for portfolios, holdings, transactions all working
- [ ] FIFO: BUY 100 shares → SELL 60 → transaction_lots.remaining_qty = 40
- [ ] FIFO: two lots → partial sell from oldest lot first (verified manually)
- [ ] FIFO: sell > available quantity → `InsufficientLotException` → 400 response
- [ ] `portfolio.events` published after each write (verify in Kafka CLI)
- [ ] CSV import: 20-row Zerodha file → all imported
- [ ] CSV import: same file twice → no duplicates
- [ ] CSV import: preview endpoint does not persist data
- [ ] `consolidated` endpoint returns merged view in base currency
- [ ] Testcontainers integration test covers full SELL → lot deduction → Kafka event cycle
- [ ] SonarQube ≥ 80% coverage
