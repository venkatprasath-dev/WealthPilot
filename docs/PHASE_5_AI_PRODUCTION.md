# Phase 5 — AI Assistant + Reports + Production Hardening

> Weeks 29–36

**Status:** 🔴 Not Started  
**Goal:** AI assistant operational, automated monthly reports, full observability stack, Kubernetes production deployment, all performance targets met.

---

## Steps

| Step | Title | Service | Status |
|------|-------|---------|--------|
| 5.1 | Monthly Report Service | Report | 🔴 |
| 5.2 | AI Chat Assistant (LangChain ReAct Agent) | AI | 🔴 |
| 5.3 | Observability: Structured Logging + Distributed Tracing | All | 🔴 |
| 5.4 | Broker Import Expansion + Expense Ratio Analyzer | Portfolio | 🔴 |
| 5.5 | Performance Tuning + Security Audit | All | 🔴 |
| 5.6 | Kubernetes Production Deployment | Infra | 🔴 |
| 5.7 | E2E Test Suite (Playwright — Top 10 Flows) | All | 🔴 |

---

## Step 5.1 — Monthly Report Service

**Tech:** `Spring Boot 3` · `Quartz` · `JPA/Hibernate` · `PostgreSQL` · `MinIO` · `AWS SES` · `Apache Kafka` · `REST API`

### What it does
Generates a professional HTML email report on the 1st of each month covering the previous month's portfolio performance. Stored in MinIO. Re-downloadable as PDF.

### Trigger chain

```
Quartz cron: 0 0 6 1 * ?  (06:00 on 1st of every month)
        ↓
Publish to Kafka: reports.scheduled {userId, reportPeriod}
        ↓
Report Service Kafka consumer:
  1. Fetch from Analytics Service: month's opening/closing value, top 3 gainers/losers, realized P&L, dividends
  2. Fetch from Event Service: upcoming 30-day events
  3. Fetch from Analytics Service: YTD return vs NIFTY 50 / S&P 500
  4. Assemble MonthlyReportDTO
  5. Render Thymeleaf template → HTML
  6. Store HTML in MinIO
  7. Send via AWS SES (Spring Mail + JavaMailSender)
  8. Insert into report_history: {userId, type=MONTHLY, generated_at, sent_at, status=SENT}
```

### Thymeleaf template structure
- Header: portfolio name, total value, month-over-month gain/loss
- Top 3 gainers + losers table with % returns and sparklines (inline SVG)
- Asset allocation drift (target vs actual)
- Dividend income received in the month
- Upcoming events (next 30 days)
- YTD performance vs benchmark (inline chart)
- Footer: download PDF link + unsubscribe link

### PDF download
`GET /api/v1/reports/{id}/download` → re-renders Thymeleaf to PDF using iText → served as download. MinIO stores the rendered PDF. Pre-signed URL valid for 24 hours.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/reports/generate` | Manual trigger (admin or user) |
| `GET` | `/api/v1/reports` | List report history |
| `GET` | `/api/v1/reports/{id}/status` | Generation status |
| `GET` | `/api/v1/reports/{id}/download` | Download PDF |

### Done when
- [ ] Monthly report email received on 1st of month with correct data
- [ ] Top 3 gainers/losers table shows real holding names
- [ ] YTD benchmark comparison visible in email
- [ ] Manual trigger works for any past month (`?period=2025-07`)
- [ ] PDF download link in email produces a valid PDF

---

## Step 5.2 — AI Chat Assistant (LangChain ReAct Agent)

**Tech:** `Python FastAPI` · `LangChain` · `ReAct Agents` · `Claude API` · `LLM Orchestration` · `Prompt Engineering` · `GenAI Pipelines` · `Redis` · `REST API` · `SSE`

### What it does
Conversational AI assistant that answers natural-language questions about the user's portfolio. Grounded in live data via tool calls. Streaming responses via SSE.

### System prompt construction
Dynamic system prompt built from Jinja2 template. Injected into every Claude API call:

```
System: You are WealthPilot, a personal investment advisor.
        Today is {date}. User's base currency is {base_currency}.
        
        Portfolio snapshot:
        Total value: {total_value}
        Top 10 holdings:
        {holdings_table}   ← fetched fresh from Portfolio Service
        
        Use tools to fetch precise data before answering.
        Never guess numbers — always call a tool.
        Be concise: 3–5 sentences unless a table is better.
```

### ReAct loop behavior
```
User message → Claude:
  THOUGHT: what does the user want?
  ACTION: which tool to call, with what args?
  OBSERVATION: tool returns data
  THOUGHT: is this enough to answer?
  (repeat up to 5 times)
  FINAL_ANSWER: response to user
```
`max_iterations=5` hard cap. If no answer in 5 calls → graceful "couldn't find a complete answer" message.

### Tool catalog (all 13 tools)

| Tool | Downstream Service | Triggered when user asks... |
|------|-------------------|---------------------------|
| `get_portfolio_summary` | Portfolio Service | "What's my portfolio worth?" |
| `get_pnl` | Analytics Service | "How much have I made?" |
| `get_top_performers` | Analytics Service | "What's my best/worst stock?" |
| `get_sector_exposure` | Analytics Service | "Am I overexposed to tech?" |
| `get_dividend_forecast` | Analytics Service | "How much dividend income this year?" |
| `get_live_quote` | Market Data Service | "What's AAPL trading at?" |
| `run_tax_harvest` | Tax Service | "Can I harvest any losses?" |
| `run_goal_projection` | Goals Service | "When can I retire?" |
| `get_news_sentiment` | News Service | "What's the market sentiment on HDFC?" |
| `get_events_upcoming` | Market Data Service | "What earnings are coming up?" |
| `get_risk_metrics` | Analytics Service | "What's my Sharpe ratio?" |
| `get_allocation` | Portfolio Service | "What's my asset allocation?" |
| `get_supply_chain_context` | Supply Chain Service | "What's the crude oil supply situation?" |

Tool descriptions are precise and mutually exclusive — each description includes what NOT to use it for to prevent wrong tool selection.

### Conversation history (Redis)
```
Key: chat:{userId}:history
Value: [{role, content, ts}] — last 50 messages
```
Injected as message history into every Claude API call. `POST /api/v1/ai/chat/new` clears the key (new conversation).

### SSE streaming
Claude API called with `stream=True`. FastAPI `StreamingResponse` with `Content-Type: text/event-stream`. Each token from Claude forwarded immediately. React `EventSource` renders tokens progressively.

### React chat UI
Fixed-position slide-in panel (right side). Input at bottom. User/AI message bubbles. Markdown rendering for tables, code blocks, lists in AI response. SSE connection via `EventSource`.

### API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/ai/chat` | Single-turn chat (non-streaming) |
| `POST` | `/api/v1/ai/chat/stream` | Streaming chat (SSE) |
| `POST` | `/api/v1/ai/chat/new` | Clear conversation history |
| `GET` | `/api/v1/ai/chat/history` | Retrieve conversation history |

### Done when
- [ ] "What's my best performing stock?" → returns a real holding name from test portfolio
- [ ] Response streams progressively (tokens appear as they arrive, not all at once)
- [ ] "What's the crude oil supply situation?" → `get_supply_chain_context` tool is called (visible in logs)
- [ ] Asking 5 questions in a row → conversation history maintained across turns
- [ ] "New conversation" clears history → next question has no memory of previous
- [ ] Token budget: month-to-date spend tracked, cap message returned if limit hit

---

## Step 5.3 — Observability: Structured Logging + Distributed Tracing

**Tech:** `Structured Logging` · `Distributed Tracing` · `OpenTelemetry` · `Jaeger` · `Prometheus` · `Grafana` · `ELK Stack` · `Docker` · `Linux`

### Structured logging — Java services

Logback with `logstash-logback-encoder`. Every log statement emits JSON:
```json
{
  "timestamp": "2026-08-01T10:30:00Z",
  "level": "INFO",
  "service": "portfolio-service",
  "traceId": "abc123def456",
  "spanId": "789ghi",
  "userId": "uuid",
  "event": "holding_created",
  "holdingId": "uuid",
  "durationMs": 23
}
```
`traceId` and `spanId` injected automatically by OpenTelemetry agent — no manual instrumentation needed.

### Structured logging — Python services

`structlog` library configured to emit identical JSON field schema. Same field names as Java services for consistent Kibana queries across all 13 services.

### OpenTelemetry — Java

OTel Java agent attached via `-javaagent:/otel-agent.jar` in each service's Dockerfile. Auto-instruments: Spring MVC controllers, JPA queries, Kafka producers/consumers, Redis commands. Zero code changes in application.

### Jaeger tracing

Every request from React frontend generates a Correlation ID at the API Gateway (`UUID`). This becomes the root `traceId`. Every downstream service call creates a child span. Jaeger trace for `GET /portfolios/{id}/performance`:

```
API Gateway (5ms)
  └── Portfolio Service (15ms)
        └── Market Data Service (45ms)
              └── Redis cache hit (2ms)
        └── Analytics Service (120ms)
              └── PostgreSQL query (85ms)
              └── TimescaleDB query (30ms)
              └── Redis cache write (3ms)
```

Jaeger UI: search by `traceId` or `userId` to find any request.

### ELK Stack

Filebeat on each node reads Docker container logs → Logstash parses JSON → Elasticsearch indices `logs-{service}-{date}`. Kibana dashboards:
- Error rate per service (last 24h)
- P95 latency per endpoint
- Kafka consumer lag per topic
- Redis cache hit rate %

### Prometheus + Grafana

Spring Boot Actuator `/actuator/prometheus` scraped by Prometheus. Grafana dashboards:
- JVM heap + GC pause time
- HTTP request rate + error rate per endpoint
- HPA scaling events (Market Data + Analytics)
- Kafka producer throughput
- Redis connected clients + memory usage
- TimescaleDB connection pool usage (HikariCP)

### Done when
- [ ] Jaeger trace visible for a full `GET /portfolios/{id}/performance` request
- [ ] Trace shows all 4 downstream service calls with individual latencies
- [ ] Kibana query for `service: "portfolio-service" AND level: "ERROR"` returns results
- [ ] Grafana dashboard shows request rate per endpoint for last 1 hour
- [ ] HPA scaling event visible in Grafana when market data load spikes

---

## Step 5.4 — Broker Import Expansion + Expense Ratio Analyzer

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `Design Patterns` · `REST API`

### New broker parsers
`GrowwCsvParser`, `VanguardCsvParser`, `IBKRCsvParser`, `SchwabCsvParser` — each extends `AbstractBrokerCSVParser`, overrides `mapColumns(Row) → TransactionDTO` only.

**IBKR multi-currency:** `Currency` column read per transaction, attached as `currency` field. FX trades flagged as `type=FOREX`.

**Format auto-detection:** `BrokerFormatDetector` reads first 5 header rows, matches against known header patterns per broker. No manual format selection for known brokers.

**Unknown column mapping:** Unresolved columns shown in preview screen with dropdown for manual mapping. Saved to `broker_mappings (broker_name, original_column, internal_field)`. Future imports reuse saved mapping.

### Expense ratio analyzer

`GET /api/v1/holdings/expense-ratios` — all MF holdings with `expense_ratio` from `instruments.metadata JSONB`.

Computes per holding:
- Annual ER cost: `current_value × expense_ratio`
- 10-year ER cost: NPV at 0% discount (compounding not assumed)
- Cheaper alternative: lookup from `fund_alternatives (regular_isin, direct_isin, er_regular, er_direct)` table — seeded with India direct/regular plan pairs

Returns: annual cost, 10-year projection, suggested switch, estimated annual savings, return impact (direct plan return estimate).

### Done when
- [ ] Import Groww CSV → all rows parsed correctly
- [ ] IBKR multi-currency statement → USD trades appear with correct currency
- [ ] Same file imported twice → zero duplicates (idempotent)
- [ ] Unknown broker column → mapped manually → saved → next import uses saved mapping
- [ ] Expense ratio analyzer shows total annual ER cost for all MF holdings

---

## Step 5.5 — Performance Tuning + Security Audit

**Tech:** `Query Optimization` · `TimescaleDB` · `PostgreSQL` · `SQL` · `Docker` · `Kubernetes` · `Linux` · `Spring Security`

### Performance targets (all must pass)

| Target | Metric | Tool |
|--------|--------|------|
| Dashboard load (cached) | P95 < 2s at 50 concurrent users | k6 |
| CRUD API | P95 < 200ms | k6 |
| Analytics API | P95 < 1s | k6 + Jaeger |
| WebSocket price tick latency | < 500ms from Kafka publish | Jaeger |
| TimescaleDB compression active | 30-day chunks compressed | SQL query |

### k6 load test scenarios

**Dashboard load (50 concurrent):**
```
50 virtual users, each:
  1. Login → get token
  2. GET /portfolios/consolidated
  3. GET /market/quotes/batch?symbols=...
  4. GET /portfolios/{id}/performance?period=1Y
  Ramp up: 10 users over 30s → hold 50 for 2min → ramp down
Target: P95 response time < 2000ms for steps 2–4
```

### Query optimization process

1. Export all queries where `durationMs > 100` from structured logs
2. Run `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` on each
3. Fix identified issues:
   - Missing index → add index
   - N+1 query → convert to single JOIN
   - Non-indexed WHERE clause → add index on that column
4. Re-run EXPLAIN ANALYZE post-fix → verify improvement
5. Document all query plans in `docs/query-plans/`

### OWASP Top 10 checklist

| ID | Issue | Mitigation in WealthPilot |
|----|-------|--------------------------|
| A01 | Broken Access Control | Every endpoint validates `X-User-Id` matches resource owner. Integration test: cross-user access → 403. |
| A02 | Cryptographic Failures | HTTPS at Nginx. JWT signed RS256. Passwords BCrypt. |
| A03 | Injection | JPA parameterized queries. No string concat in SQL. ES structured DSL. |
| A04 | Insecure Design | Auth service issues tokens. Gateway validates. No service bypasses gateway. |
| A05 | Security Misconfiguration | Vault for all secrets. No K8s YAML secrets. ES security enabled in prod. |
| A06 | Vulnerable Components | Dependabot alerts. Maven/pip dependency scanning in CI. |
| A07 | Auth Failures | JWT blacklist on logout. Refresh rotation. Rate limit on `/auth/**`. |
| A09 | Logging Failures | All requests logged with `traceId`. Failed auth attempts logged and alerted. |
| A10 | SSRF | MarketDataService is the only external HTTP caller. No user-supplied URLs fetched. |

### HikariCP tuning
`maximumPoolSize` = `floor(PostgreSQL max_connections / number_of_services)`. Monitor `hikari_connection_pending_threads` in Prometheus. Alert if > 0 for > 30 seconds.

### Done when
- [ ] k6 dashboard test: P95 < 2s at 50 concurrent users ✓
- [ ] k6 CRUD test: P95 < 200ms ✓
- [ ] TimescaleDB: `SELECT * FROM chunk_compression_stats('price_history')` shows compressed chunks
- [ ] OWASP checklist: cross-user access test returns 403 (automated in integration tests)
- [ ] All `durationMs > 100` queries from logs have been EXPLAIN ANALYZED and fixed

---

## Step 5.6 — Kubernetes Production Deployment

**Tech:** `Kubernetes` · `Docker` · `CI/CD` · `Jenkins` · `Linux` · `HashiCorp Vault`

### K8s manifest per service

```yaml
# Deployment
replicas: 2 (most services)
         1 (Goals, Supply Chain - low traffic)

# HPA
Market Data: 2→10 pods on CPU > 70%
Analytics:   2→4  pods on CPU > 70%

# Probes
liveness:  GET /actuator/health — every 10s, 3 failures → restart
readiness: GET /actuator/health/readiness — every 5s, fail → no traffic

# Resources
requests: cpu=100m, memory=256Mi
limits:   cpu=500m, memory=512Mi
```

### HashiCorp Vault integration
Vault agent runs as sidecar in each pod. On pod startup, fetches secrets for that service → writes to pod filesystem as files. Application reads secrets from filesystem. Secrets never appear in `kubectl describe pod` output.

Secrets managed in Vault:
- `secret/wealthpilot/db` — PostgreSQL password
- `secret/wealthpilot/redis` — Redis password
- `secret/wealthpilot/kafka` — Kafka SASL credentials
- `secret/wealthpilot/jwt` — JWT signing key (RSA private key)
- `secret/wealthpilot/claude` — Claude API key
- `secret/wealthpilot/ses` — AWS SES credentials
- `secret/wealthpilot/market/*` — Provider API keys

### Blue-green deployment (Jenkins Stage 9)
1. New deployment `-green` created alongside live `-blue`
2. Jenkins waits for `-green` pods to be Ready
3. Service selector updated from `blue` to `green`
4. Old `-blue` deployment kept 30 minutes for fast rollback
5. Auto-rollback: monitoring script checks new pods — if `restart_count > 3` within 5 minutes, selector reverts to `-blue`

### Nginx Ingress
SSL termination via cert-manager (Let's Encrypt). WebSocket annotation: `nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"` to hold long-lived connections.

### Done when
- [ ] All 13 services deployed in K8s (not Docker Compose)
- [ ] All pods show `1/1 Running` in `kubectl get pods`
- [ ] HPA active for Market Data and Analytics services
- [ ] Vault agent sidecar running in each pod — no secrets in env variables
- [ ] Blue-green deploy succeeds: new pod comes up, traffic switches, old pod terminated
- [ ] Auto-rollback: kill new pods repeatedly → selector reverts to blue automatically

---

## Step 5.7 — E2E Test Suite (Playwright — Top 10 Flows)

**Tech:** `Playwright` · `Integration Testing` · `CI/CD`

### Setup
Headless Chromium. Test database seeded with fixed test user + 10 known holdings before each run. Seed rolled back after run. Run against staging environment. Part of Jenkins Stage 7.

### Top 10 flows

| # | Flow | Assertion |
|---|------|-----------|
| 1 | Register → login → empty dashboard | Dashboard loads, portfolio value = ₹0 |
| 2 | Add stock holding → verify holdings table | Holding appears with correct quantity + avg cost |
| 3 | Record BUY → record SELL → verify P&L | Realized P&L visible and correct |
| 4 | Import Zerodha CSV (20 rows) → re-import → no duplicates | Holdings count correct; re-import changes nothing |
| 5 | Set price alert → mock price cross → verify email queued | Alert history shows triggered alert |
| 6 | AI chat: "What's my best stock?" → verify real holding name in response | AI response contains a known test holding |
| 7 | Analytics: view risk metrics page → all 5 metrics visible | Sharpe / Sortino / Beta / VaR / Max Drawdown all non-zero |
| 8 | Upload PDF document → verify auto-tagged by ticker | Tags field contains expected ticker symbols |
| 9 | Create goal → view Monte Carlo chart | P10 / P50 / P90 curves all render |
| 10 | Monthly report manual trigger → verify Mailhog receipt | Email received in test Mailhog instance |

### Done when
- [ ] All 10 Playwright flows green on staging
- [ ] Playwright suite runs in Jenkins Stage 7 and blocks Stage 9 on failure
- [ ] Test seed + teardown leaves no residual test data in staging DB

---

## Phase 5 Exit Criteria

- [ ] Monthly report email received with correct data on 1st
- [ ] AI chat streaming response visible for any portfolio question
- [ ] Jaeger trace covers full request path for any analytics query
- [ ] k6: dashboard P95 < 2s at 50 concurrent users
- [ ] OWASP: cross-user access returns 403 (automated)
- [ ] All 10 Playwright E2E flows green
- [ ] All 13 services running in Kubernetes
- [ ] Blue-green deploy succeeds with zero downtime
- [ ] Vault: no secrets in env variables or K8s YAML manifests

→ Next: [Phase 6 — Extended Features](PHASE_6_EXTENDED.md)
