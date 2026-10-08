# Phase 1 — Auth + Portfolio CRUD + Dashboard MVP

> Weeks 1–6

**Status:** 🔴 Not Started  
**Goal:** Working app where you can log in, add your real holdings, and see portfolio value with live prices and P&L. By end of Week 3, you are tracking your actual portfolio in your own app.

---

## Steps

| Step | Title | Service | Status |
|------|-------|---------|--------|
| 1.1 | Auth Service | Auth | 🔴 |
| 1.2 | API Gateway | Gateway | 🔴 |
| 1.3 | Portfolio Service | Portfolio | 🔴 |
| 1.4 | Instrument Registry | Portfolio | 🔴 |
| 1.5 | Market Data Service (quotes + cache) | Market Data | 🔴 |
| 1.6 | Dashboard v1 | Frontend | 🔴 |
| 1.7 | Phase 1 Test Suite | All | 🔴 |
| 1.8 | Jenkins CI/CD — Phase 1 Services | Infra | 🔴 |

→ Full service specs: [`docs/services/`](../services/)

---

## Step 1.1 — Auth Service

**Tech:** `Spring Boot 3` · `Spring Security 6` · `JWT` · `OAuth2` · `BCrypt` · `JPA/Hibernate` · `PostgreSQL` · `Redis` · `Spring MVC` · `REST API`

### What it does
Identity layer for the entire platform. Registration, login, token issuance, token refresh, Google OAuth, logout. Every other service trusts the tokens this service produces.

### Flows

**Registration:** User submits email + password → BCrypt hash (cost 12) → insert `users` row → issue JWT access token (15-min TTL) + refresh token (7-day TTL) → store refresh token in Redis.

**Login:** Lookup user by email → verify BCrypt → issue token pair.

**Google OAuth:** Spring Security OAuth2 Client → redirect to Google → exchange code for Google ID token → extract email → find or create user → issue WealthPilot JWT. From this point identical to email user.

**Token refresh:** Client sends refresh token → validate against Redis entry → issue new access token → rotate refresh token (delete old, insert new) → if Redis has no entry, return 401.

**Logout:** Delete refresh token from Redis → add access token JTI to Redis blacklist with TTL = remaining access token lifetime → API Gateway checks this blacklist on every request.

### Data model

```
users (
  id UUID PK,
  email VARCHAR(255) UNIQUE,
  name VARCHAR(255),
  password_hash VARCHAR(255),   -- null for OAuth users
  provider ENUM(LOCAL, GOOGLE),
  base_currency CHAR(3) DEFAULT 'INR',
  timezone VARCHAR(50),
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
)

user_preferences (
  user_id UUID FK → users.id,
  theme VARCHAR(20) DEFAULT 'dark',
  email_frequency ENUM(MONTHLY, WEEKLY, QUARTERLY),
  alert_channels JSONB,          -- {"email": true, "push": false}
  updated_at TIMESTAMPTZ
)
```

### Redis key design

| Key | Value | TTL |
|-----|-------|-----|
| `refresh:{userId}` | refresh token string | 7 days |
| `blacklist:jti:{jti}` | `"1"` | Remaining access token lifetime |
| `rate:{userId}` | request counter | 1 minute (rolling) |

### API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/v1/auth/register` | No | Create account → token pair |
| `POST` | `/api/v1/auth/login` | No | Login → token pair |
| `POST` | `/api/v1/auth/refresh` | Refresh token | Rotate tokens |
| `POST` | `/api/v1/auth/logout` | Bearer | Invalidate session |
| `POST` | `/api/v1/auth/oauth2/google` | No | Google OAuth login |
| `GET` | `/api/v1/auth/me` | Bearer | Get user profile |
| `PUT` | `/api/v1/auth/me` | Bearer | Update profile/preferences |

### Done when
- [ ] POST `/register` creates user, returns token pair
- [ ] POST `/login` returns tokens, BCrypt verified
- [ ] GET `/me` with valid token returns correct user
- [ ] Google OAuth redirects and returns WealthPilot tokens
- [ ] Refresh rotates both tokens; old refresh is invalidated
- [ ] Logout → subsequent request with same token returns 401
- [ ] JUnit 5 + Testcontainers (real PG + Redis): all auth flows tested
- [ ] Mockito: `KafkaProducer`, `EmailSender` mocked in unit tests where applicable

---

## Step 1.2 — API Gateway

**Tech:** `Spring Cloud Gateway` · `Spring WebFlux` · `Spring Security` · `JWT` · `Redis` · `Eureka` · `Resilience4j`

### What it does
Single ingress point for all client HTTP traffic. JWT validation, routing, rate limiting, CORS, circuit breaking — all at the edge before any downstream call is made.

### Key behaviors

**JWT validation (GlobalFilter):** Extracts `Authorization: Bearer <token>` → validates signature + expiry + JTI blacklist (Redis lookup) → injects `X-User-Id` header into forwarded request. Invalid tokens return 401 immediately. Downstream services trust `X-User-Id` without re-validating.

**Rate limiting:** Redis-backed token-bucket. 60 requests/minute per authenticated `X-User-Id`. Exceeding returns 429. State stored as `rate:{userId}` in Redis with rolling TTL.

**Circuit breaker:** Resilience4j on each route. Downstream failure → 503 with JSON error body. Prevents connection hangs.

**CORS:** Configured for React frontend origin with credentials. Preflight handled at gateway, not forwarded.

**Service discovery:** Routes resolved via Eureka — `lb://auth-service` rewrites to a real host:port at request time.

### Route table

| Prefix | Downstream |
|--------|-----------|
| `/api/v1/auth/**` | `auth-service` |
| `/api/v1/portfolios/**` | `portfolio-service` |
| `/api/v1/holdings/**` | `portfolio-service` |
| `/api/v1/market/**` | `market-data-service` |
| `/api/v1/analytics/**` | `analytics-service` |
| `/api/v1/alerts/**` | `notification-service` |
| `/api/v1/news/**` | `news-sentiment-service` |
| `/api/v1/goals/**` | `goals-service` |
| `/api/v1/documents/**` | `document-service` |
| `/api/v1/reports/**` | `report-service` |
| `/api/v1/tax/**` | `tax-service` |
| `/api/v1/ai/**` | `ai-service` |
| `/api/v1/supply-chain/**` | `supply-chain-service` |

### Done when
- [ ] Request without token → 401, never reaches downstream service
- [ ] Valid token → correct service receives request with `X-User-Id` header
- [ ] 61 rapid requests from same user → 62nd returns 429
- [ ] Downstream service killed → gateway returns 503, not connection timeout
- [ ] Jaeger trace shows gateway as root span for every request

---

## Step 1.3 — Portfolio Service

**Tech:** `Spring Boot 3` · `Spring MVC` · `JPA/Hibernate` · `PostgreSQL` · `Apache Kafka` · `REST API` · `Design Patterns` · `DSA`

### What it does
Stateful core of the application. Manages portfolios, holdings, and transactions. FIFO cost basis engine. Broker CSV import. Publishes events to Kafka.

### Data model

```
portfolios (id UUID PK, user_id FK, name, description, currency, created_at)

holdings (
  id UUID PK, portfolio_id FK, instrument_id FK,
  quantity NUMERIC(20,8),
  avg_cost_price NUMERIC(20,8),   -- denormalized, recomputed on each BUY
  cost_currency CHAR(3),
  buy_date DATE, broker TEXT, notes TEXT
)

transactions (
  id UUID PK, holding_id FK,
  type ENUM(BUY, SELL, DIVIDEND, SPLIT, BONUS, SIP),
  quantity NUMERIC, price NUMERIC, fees NUMERIC,
  currency CHAR(3), executed_at TIMESTAMPTZ, notes TEXT
)

transaction_lots (                  -- FIFO engine
  id UUID PK, transaction_id FK,   -- FK to BUY transactions only
  remaining_qty NUMERIC,
  acquired_at TIMESTAMPTZ
)
```

### FIFO matching — the algorithm

On each SELL:
1. `SELECT ... FROM transaction_lots WHERE holding_id = ? AND remaining_qty > 0 ORDER BY acquired_at ASC FOR UPDATE`
2. Iterate lots in order, consuming `remaining_qty` until sell quantity exhausted
3. Realized gain per lot: `(sell_price - lot_cost) × qty_consumed`
4. `FOR UPDATE` row lock prevents concurrent sell race condition

This is a queue drain (FIFO). First-in, first-out — implemented in SQL.

### Kafka event

After every successful DB commit:
```json
{
  "eventType": "HOLDING_UPDATED",
  "userId": "uuid",
  "portfolioId": "uuid",
  "holdingId": "uuid",
  "timestamp": "2026-08-01T10:00:00Z"
}
```
Topic: `portfolio.events`. Published **after** commit. If Kafka is down, transaction still saves. Idempotent consumers handle reprocessing.

### CSV import — Template Method pattern

`AbstractBrokerCSVParser` defines: read → detect format → map columns → validate → preview → commit.  
Concrete subclasses: `ZerodhaCsvParser`, `GrowwCsvParser`, `IBKRCsvParser` — each overrides column mapping only.  
Deduplication: hash = `MD5(symbol + executed_at + type + quantity + price)`. If hash exists → skip row. Import is idempotent.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/portfolios` | List user portfolios |
| `POST` | `/api/v1/portfolios` | Create portfolio |
| `GET` | `/api/v1/portfolios/{id}` | Get portfolio |
| `PUT` | `/api/v1/portfolios/{id}` | Update portfolio |
| `DELETE` | `/api/v1/portfolios/{id}` | Delete portfolio |
| `GET` | `/api/v1/portfolios/{id}/holdings` | List holdings |
| `POST` | `/api/v1/portfolios/{id}/holdings` | Add holding |
| `PUT` | `/api/v1/holdings/{id}` | Edit holding |
| `DELETE` | `/api/v1/holdings/{id}` | Delete holding |
| `GET` | `/api/v1/holdings/{id}/transactions` | Transaction history |
| `POST` | `/api/v1/holdings/{id}/transactions` | Record BUY / SELL |
| `POST` | `/api/v1/portfolios/{id}/import` | CSV import → preview |
| `POST` | `/api/v1/portfolios/{id}/import/confirm` | Commit previewed rows |
| `GET` | `/api/v1/portfolios/consolidated` | Merged view all portfolios |

### Done when
- [ ] Add stock holding → appears in GET response with correct values
- [ ] Record BUY → transaction_lots row created
- [ ] Record SELL → FIFO lots consumed, remaining_qty decremented
- [ ] FIFO realized gain matches manual calculation
- [ ] Import 20-row Zerodha CSV → all rows imported
- [ ] Re-import same CSV → zero duplicates
- [ ] `portfolio.events` message visible in Kafka CLI after each write
- [ ] Testcontainers integration test: full SELL cycle verified end-to-end

---

## Step 1.4 — Instrument Registry

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Elasticsearch` · `REST API`

### What it does
Shared catalog of all financial instruments. Every service references instruments by ID. Auto-registers unknown symbols via Market Data Service.

### Data model

```
instruments (
  id UUID PK,
  symbol VARCHAR, exchange VARCHAR, name VARCHAR,
  type ENUM(STOCK, MF, BOND, COMMODITY, FOREX, ETF, CRYPTO, REAL_ESTATE, GOLD),
  currency CHAR(3), country CHAR(2), sector TEXT, industry TEXT, isin VARCHAR,
  metadata JSONB   -- type-specific: {expense_ratio}, {coupon_rate}, {nav}, etc.
)
```

`metadata JSONB` stores type-specific attributes without schema sprawl. Adding a new instrument type = new JSON shape, no migration.

### Auto-registration
Unknown symbol added via Portfolio Service → Market Data Service fetch → auto-insert into `instruments`. If not found → 422 "unknown symbol."

### Symbol autocomplete
Elasticsearch index on `instruments`. `GET /api/v1/instruments/search?q=HDFC` → sub-10ms fuzzy match on name + symbol.

### Done when
- [ ] Add unknown symbol in holdings → auto-registers in instruments table
- [ ] Search `?q=TCS` → returns relevant results within 10ms

---

## Step 1.5 — Market Data Service (Phase 1 scope)

**Tech:** `Spring Boot 3` · `Spring WebFlux` · `Redis` · `TimescaleDB` · `Quartz` · `Resilience4j` · `REST API` · `Design Patterns`

### What it does (Phase 1 subset)
Quote fetching with provider cascade and Redis caching. EOD price ingestion via Quartz. WebSocket and full TimescaleDB work is Phase 2.

### Provider cascade

```
Yahoo Finance  → cache miss → call provider
  ↓ (Resilience4j circuit OPEN)
Twelve Data   → call provider
  ↓ (circuit OPEN)
Alpha Vantage → call provider
  ↓ (all fail)
Redis stale   → last-known price, response includes "stale": true
```

### Redis quote schema

| Key | Value | TTL |
|-----|-------|-----|
| `quote:{symbol}` | JSON: `{symbol, price, change, change_pct, volume, updated_at, source}` | 15 seconds (live hours) / 24 hours (after close) |

### Circuit breaker config (per provider)

- Failure rate threshold: 50%
- Slow call threshold: 2 seconds
- Sliding window: 10 calls
- Wait in OPEN state: 30 seconds

### Quartz EOD job
Runs at 18:00 IST / 21:00 EST daily. Fetches OHLCV for all known instruments. Batch-inserts into `price_history` TimescaleDB hypertable. Job log records success/failure per instrument.

### API (Phase 1)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/market/quote/{symbol}` | Live quote (Redis first, then provider) |
| `GET` | `/api/v1/market/quotes/batch` | `?symbols=AAPL,INFY` — batch Redis lookup |
| `GET` | `/api/v1/market/search` | Instrument autocomplete (Elasticsearch) |

### Done when
- [ ] `GET /market/quote/AAPL` returns price
- [ ] Second call returns from Redis (response time visibly < 5ms)
- [ ] Mock Yahoo Finance down → falls to Twelve Data
- [ ] Mock all providers down → returns stale Redis value with `"stale": true`
- [ ] Quartz job fires at scheduled time (visible in logs)
- [ ] `price_history` has rows after EOD job

---

## Step 1.6 — Dashboard v1

**Tech:** `React 18` · `TypeScript` · `TanStack Query` · `Zustand` · `Recharts` · `TanStack Table`

### What it does
First working UI. Login → dashboard showing total portfolio value, asset allocation donut, performance line chart, holdings table with live prices.

### Data flows

On dashboard mount, 3 parallel TanStack Query calls:
1. `GET /portfolios/consolidated` — all holdings + current values
2. `GET /portfolios/{id}/performance?period=1Y` — value time-series for chart
3. `GET /market/quotes/batch?symbols=...` — live prices for all held symbols

TanStack Query stale-while-revalidate: data refetches every 60 seconds in background without blocking the UI.

### Components

| Component | Data source | Tech |
|-----------|------------|------|
| Net worth card | `consolidated` | Recharts number display |
| Asset allocation donut | `consolidated.allocation` | Recharts `PieChart` |
| Performance line chart | `performance` | Recharts `LineChart` |
| Holdings table | `consolidated.holdings` + batch quotes | TanStack Table (sortable, filterable) |
| Period selector | N/A — triggers query param change | React state |

### Zustand global store
Holds: `activePortfolioId`, `baseCurrency`, `user` profile. All other data is server state (TanStack Query). No Redux, no prop-drilling.

### Auth flow in frontend
On app load: check `localStorage` for access token → check expiry → if expired, call `/auth/refresh` → if that fails, redirect to `/login`. Login page: email/password form + "Sign in with Google" button.

### Done when
- [ ] Login with email works end-to-end
- [ ] Google OAuth button redirects and logs in
- [ ] Real holdings visible in holdings table
- [ ] Total portfolio value matches manual calculation
- [ ] Allocation donut slices add to 100%
- [ ] Performance line chart renders with real data
- [ ] Table sortable by P&L column
- [ ] Auto-refresh (60s) updates prices without page reload

---

## Step 1.7 — Phase 1 Test Suite

**Tech:** `JUnit 5` · `Mockito` · `Testcontainers` · `Integration Testing` · `Playwright`

### Unit tests (JUnit 5 + Mockito)

**What Mockito mocks in each service:**

| Service | Mocked | Why |
|---------|--------|-----|
| Portfolio | `KafkaProducer`, `MarketDataClient` | Verify event payload without Kafka; inject known prices |
| Auth | `EmailSender`, `RedisTemplate` | Isolate token logic from infra |
| Market Data | `YahooFinanceProvider`, `TwelveDataProvider` | Test cascade without external calls |

**Key Mockito usage:**
- `ArgumentCaptor<KafkaMessage>` — assert exact payload fields on Kafka publish
- `verify(producer, times(1)).send(...)` — confirm event fires exactly once per transaction
- `when(provider.fetchQuote("AAPL")).thenThrow(...)` — simulate provider failure for cascade test

**Key unit test classes:**
- `CostBasisServiceTest` — 10+ FIFO scenarios: partial lot, multi-lot, over-sell throws, split adjustment
- `AuthServiceTest` — register/login/refresh/logout token lifecycle, blacklist insertion
- `MarketDataCacheTest` — cache hit bypasses provider, cache miss triggers call, stale fallback

### Integration tests (Testcontainers)

Starts real PostgreSQL + Redis + Kafka per test class (`@Container` + static lifecycle — one container instance per class, not per method).

**Key integration test classes:**
- `PortfolioControllerIntegrationTest` — POST holding → POST sell → verify lot table → verify Kafka message consumed by embedded consumer
- `AuthControllerIntegrationTest` — register/login/refresh/logout cycle against real Redis
- `MarketDataProviderIntegrationTest` — WireMock stubs for each provider, cascade fallback verified

### E2E tests (Playwright — top 3 flows for Phase 1)

1. Register → login → add holding → see it on dashboard
2. Record BUY → record SELL → verify realized P&L shown
3. Google OAuth login

Playwright tests run against `localhost:3000` with seeded test DB. Part of Jenkins Stage 7.

### Coverage gate
SonarQube blocks pipeline if line coverage < 80% on any service. Enforced in Jenkins Stage 5.

### Done when
- [ ] `CostBasisServiceTest` passes all 10 FIFO scenarios
- [ ] `AuthControllerIntegrationTest` covers all 5 auth flows against real containers
- [ ] All 3 Playwright flows green in a running local environment
- [ ] SonarQube shows ≥ 80% line coverage on Auth, Portfolio, Market Data services

---

## Step 1.8 — Jenkins CI/CD — Phase 1 Services

**Tech:** `Jenkins` · `CI/CD` · `SonarQube` · `Docker` · `Git`

### Pipeline stages (per service)

| Stage | Action | Failure behavior |
|-------|--------|-----------------|
| 1. Checkout | `git checkout` from feature branch | N/A |
| 2. Build | `mvn clean package -DskipTests` | Stop — compilation error |
| 3. Unit Tests | `mvn test` — JUnit 5 + Mockito | Stop — failing tests block PR |
| 4. Integration Tests | `mvn verify -Pintegration` — Testcontainers | Stop — real infra failures caught |
| 5. Static Analysis | SonarQube + SpotBugs — quality gate ≥ 80% coverage, no CRITICAL | Stop — no exceptions |
| 6. Docker Build + Push | `docker build` + `docker push` with Git SHA tag | Stop — broken image not pushed |
| 7. Deploy to Staging | `kubectl set image` + `kubectl rollout status` (120s timeout) | Stop — rollout failure |
| 8. Integration Tests on Staging | Newman (Postman) against staging URL | Stop — blocks production |
| 9. Promote to Production | Manual approval → blue-green deploy | Automated rollback on pod failure |

### Jenkinsfile location
`infrastructure/jenkins/Jenkinsfile.{service-name}` — one per service.  
Shared library: `infrastructure/jenkins/shared-lib/` — common stage definitions.

### Docker image tagging
`wealthpilot/{service}:{git-sha}` — every image uniquely traceable to a commit. No `latest` tag in production.

### Blue-green deployment
New pods come up alongside old. Service selector switches when new pods are Ready. Old pods kept 30 minutes for fast rollback. Auto-rollback if new pods have `restart_count > 3` within 5 minutes.

### Done when
- [ ] Jenkins pipeline exists and runs for auth-service, portfolio-service, market-data-service, api-gateway
- [ ] Pipeline fails correctly on: compilation error, test failure, coverage gate miss
- [ ] Docker image pushed with Git SHA tag after successful pipeline
- [ ] Staging deployment completes; Newman tests pass
- [ ] Manual approval gate visible in Jenkins UI

---

## Phase 1 Exit Criteria

- [ ] Log in with Google on your own app
- [ ] Add your real stock holdings manually
- [ ] Import a Zerodha CSV statement
- [ ] Dashboard shows correct total portfolio value, P&L, and allocation
- [ ] Jenkins CI green for all Phase 1 services
- [ ] SonarQube ≥ 80% on all Phase 1 services

→ Next: [Phase 2 — Market Data + WebSocket + Dividends + Alerts](PHASE_2_MARKET.md)
