# WealthPilot

> Personal Bloomberg Terminal — Portfolio Intelligence Platform

Java 21 · Spring Boot 3 · React 18 · PostgreSQL · TimescaleDB · Kafka · Redis · Elasticsearch · Python FastAPI · LangChain · Claude API · FinBERT · Docker · Kubernetes · Jenkins

---

## Project Status

| Phase | Focus | Status | Progress |
|-------|-------|--------|----------|
| **Phase 0** | Setup — Repo, Docker, DB Baseline | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 1** | Auth + Portfolio CRUD + Dashboard MVP | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 2** | Market Data + WebSocket + Dividends + Alerts | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 3** | Analytics + News Sentiment + Tax | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 4** | Calendar + Supply Chain + Documents + Goals | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 5** | AI Assistant + Reports + Hardening + K8s | 🔴 Not Started | `░░░░░░░░░░` 0% |
| **Phase 6** | Extended Features + Polish | 🔴 Not Started | `░░░░░░░░░░` 0% |

**Legend:** 🟢 Done · 🟡 In Progress · 🔴 Not Started · ⏸ Blocked

---

## Service Build Status

| # | Service | Language | Phase | Status | Notes |
|---|---------|----------|-------|--------|-------|
| – | Docker Compose Dev Stack | Infra | 0 | 🔴 | PG + Redis + Kafka + ES + MinIO |
| – | Flyway DB Baseline | Infra | 0 | 🔴 | All 18 tables |
| – | Eureka Server | Java | 0 | 🔴 | Service discovery |
| 1 | API Gateway | Java | 1 | 🔴 | JWT validation + routing |
| 2 | Auth Service | Java | 1 | 🔴 | JWT + Google OAuth2 |
| 3 | Portfolio Service | Java | 1 | 🔴 | Holdings + transactions + FIFO |
| 4 | Market Data Service | Java | 1–2 | 🔴 | Quotes + WebSocket + TimescaleDB |
| 5 | Analytics Service | Java | 3 | 🔴 | P&L + risk metrics |
| 6 | Notification Service | Java | 2 | 🔴 | Alerts + email + push |
| 7 | News & Sentiment Service | Python | 3 | 🔴 | FinBERT + Kafka pipeline |
| 8 | AI Service | Python | 5 | 🔴 | LangChain ReAct + SSE |
| 9 | Supply Chain Service | Java | 4 | 🔴 | Vessel + cargo tracking |
| 10 | Goals & Planning Service | Python | 4 | 🔴 | Monte Carlo simulation |
| 11 | Document Service | Java | 4 | 🔴 | MinIO + PDF + Elasticsearch |
| 12 | Report Service | Java | 5 | 🔴 | Thymeleaf + Quartz + SES |
| 13 | Tax Service | Java | 3 | 🔴 | STCG/LTCG + tax harvest |
| – | WebSocket Gateway | Java | 2 | 🔴 | Live price streaming |
| – | React Frontend | TypeScript | 1–5 | 🔴 | SPA dashboard |

---

## Milestone Checklist

### Phase 0 — Setup

- [ ] Monorepo created on GitHub with correct directory layout
- [ ] Git branching strategy documented (`main` protected, CI required)
- [ ] GitHub Projects sprint board configured
- [ ] `docker-compose.yml` brings up PG + Redis + Kafka + ES + MinIO
- [ ] All 5 infra containers pass health checks
- [ ] Flyway migrations: all 18 tables created
- [ ] Design Patterns ADR documented

### Phase 1 — Auth + Portfolio + Dashboard MVP

- [ ] Auth Service: register → JWT issued
- [ ] Auth Service: login → access + refresh tokens
- [ ] Auth Service: Google OAuth flow working
- [ ] Auth Service: token refresh rotates pair
- [ ] Auth Service: logout blacklists token in Redis
- [ ] API Gateway: JWT validated at edge (invalid → 401, never forwarded)
- [ ] API Gateway: rate limiting fires at 60 req/min
- [ ] API Gateway: CORS allows frontend origin
- [ ] Portfolio Service: CRUD for portfolios, holdings, transactions
- [ ] Portfolio Service: FIFO cost basis verified manually
- [ ] Portfolio Service: Zerodha CSV import (20 rows, no duplicates on re-import)
- [ ] Portfolio Service: Kafka `portfolio.events` published after each write
- [ ] Market Data Service: `/market/quote/AAPL` returns price
- [ ] Market Data Service: second call hits Redis (faster)
- [ ] Market Data Service: Quartz EOD job runs at schedule
- [ ] Market Data Service: `price_history` has rows after EOD job
- [ ] Dashboard: login → see holdings table with correct values
- [ ] Dashboard: asset allocation donut renders
- [ ] Dashboard: performance line chart renders
- [ ] Jenkins CI: all Phase 1 services pass 9-stage pipeline
- [ ] SonarQube: ≥ 80% coverage on all Phase 1 services

### Phase 2 — Market Data + WebSocket + Dividends + Alerts

- [ ] TimescaleDB hypertable with compression policy active
- [ ] `EXPLAIN ANALYZE` run on top 5 queries, results documented
- [ ] Candlestick charts render with real data (1D / 1W / 1M / 1Y)
- [ ] WebSocket live prices updating dashboard without page refresh
- [ ] Dividend calendar shows ex-dates for held stocks
- [ ] DRIP simulation returns comparison values
- [ ] Multi-currency: asset P&L vs FX P&L shown separately
- [ ] Price alert created → fires within 60 seconds of threshold cross
- [ ] SIP XIRR matches Excel for same input

### Phase 3 — Analytics + News + Tax

- [ ] Sharpe / Sortino / Beta / VaR / Max Drawdown visible in UI
- [ ] Correlation matrix heatmap renders
- [ ] TWR and XIRR match manual calculation
- [ ] News feed shows articles for held tickers with sentiment badge
- [ ] FinBERT running: articles scored BULLISH / NEUTRAL / BEARISH
- [ ] Claude entity extraction: tickers linked from article text
- [ ] Sector sentiment heatmap renders
- [ ] STCG/LTCG classification correct for a sold position
- [ ] Tax harvest candidates ranked by savings amount
- [ ] Rebalancing trade suggestions generated on allocation drift > 5%

### Phase 4 — Calendar + Supply Chain + Documents + Goals

- [ ] Event calendar shows earnings/dividends for next 30 days
- [ ] Weekly digest email sent every Monday
- [ ] Vessel map renders with real positions (Leaflet)
- [ ] Port congestion index computed
- [ ] PDF upload → auto-tagged by ticker in document vault
- [ ] Document full-text search returns correct results
- [ ] Goal created → Monte Carlo P10/P50/P90 chart renders
- [ ] Goal probability % visible and non-zero
- [ ] Net worth tracker shows assets + liabilities + portfolio total

### Phase 5 — AI + Reports + Production

- [ ] Monthly report email received with correct data on 1st of month
- [ ] AI chat: "What's my best stock?" returns real holding name
- [ ] AI chat: SSE streaming (response appears progressively)
- [ ] AI chat: supply chain tool used when asked about crude oil
- [ ] Jaeger: full distributed trace visible for `/portfolios/{id}/performance`
- [ ] Kibana: error rate + latency dashboard per service
- [ ] k6: dashboard P95 < 2s at 50 concurrent users
- [ ] OWASP: cross-user access attempt returns 403
- [ ] Playwright: all 10 E2E flows green
- [ ] Kubernetes: blue-green deploy succeeds with zero downtime
- [ ] HashiCorp Vault: no secrets in env variables or K8s YAML

---

## Priority Queue

> What to build next. Update this section weekly.

```
🔥 NOW
  └── Step 0.1 — Create monorepo + GitHub repo
  └── Step 0.3 — Docker Compose dev stack (PG + Redis + Kafka + ES + MinIO)
  └── Step 0.4 — Flyway migrations baseline (all 18 tables)

⏭ NEXT
  └── Step 1.1 — Auth Service (JWT + Google OAuth)
  └── Step 1.2 — API Gateway (routing + JWT edge validation)

📋 BACKLOG
  └── Step 1.3 — Portfolio Service
  └── Step 1.4 — Instrument Registry
  └── Step 1.5 — Market Data Service
  └── Step 1.6 — Dashboard v1
  └── Step 1.7 — Phase 1 test suite
  └── Step 1.8 — Jenkins CI/CD for Phase 1 services
```

---

## Repository Structure

```
wealthpilot/
├── frontend/
│   └── wealthpilot-web/          React 18 + TypeScript + Vite
│
├── services/
│   ├── api-gateway/              Spring Cloud Gateway
│   ├── auth-service/             Spring Boot 3 — Java
│   ├── portfolio-service/        Spring Boot 3 — Java
│   ├── market-data-service/      Spring Boot 3 — Java
│   ├── analytics-service/        Spring Boot 3 — Java
│   ├── notification-service/     Spring Boot 3 — Java
│   ├── supply-chain-service/     Spring Boot 3 — Java
│   ├── document-service/         Spring Boot 3 — Java
│   ├── report-service/           Spring Boot 3 — Java
│   ├── tax-service/              Spring Boot 3 — Java
│   ├── news-sentiment-service/   Python FastAPI
│   ├── ai-service/               Python FastAPI + LangChain
│   └── goals-service/            Python FastAPI + NumPy/SciPy
│
├── infrastructure/
│   ├── docker-compose.yml        Dev: all infra containers
│   ├── kafka/                    Topic configs
│   ├── postgres/                 Init scripts
│   ├── redis/                    Redis config
│   ├── elasticsearch/            Index templates
│   ├── minio/                    Bucket configs
│   └── monitoring/               Prometheus + Grafana configs
│
├── deployment/
│   ├── kubernetes/               K8s manifests per service
│   └── helm/                     Helm charts
│
├── docs/
│   ├── phases/                   Phase execution plans
│   │   ├── PHASE_0_SETUP.md
│   │   ├── PHASE_1_MVP.md
│   │   ├── PHASE_2_MARKET.md
│   │   ├── PHASE_3_ANALYTICS.md
│   │   ├── PHASE_4_ADVANCED.md
│   │   ├── PHASE_5_AI_PRODUCTION.md
│   │   └── PHASE_6_EXTENDED.md
│   │
│   ├── services/                 Per-service specs
│   │   ├── AUTH_SERVICE.md
│   │   ├── PORTFOLIO_SERVICE.md
│   │   ├── MARKET_DATA_SERVICE.md
│   │   ├── ANALYTICS_SERVICE.md
│   │   ├── NOTIFICATION_SERVICE.md
│   │   ├── NEWS_SENTIMENT_SERVICE.md
│   │   ├── AI_SERVICE.md
│   │   ├── SUPPLY_CHAIN_SERVICE.md
│   │   ├── GOALS_SERVICE.md
│   │   ├── DOCUMENT_SERVICE.md
│   │   ├── REPORT_SERVICE.md
│   │   └── TAX_SERVICE.md
│   │
│   ├── infrastructure/
│   │   ├── DOCKER_SETUP.md
│   │   ├── KUBERNETES.md
│   │   ├── CI_CD_PIPELINE.md
│   │   ├── DATABASE_SCHEMA.md
│   │   └── OBSERVABILITY.md
│   │
│   └── architecture/
│       ├── DESIGN_PATTERNS.md
│       ├── KAFKA_TOPOLOGY.md
│       └── API_CONVENTIONS.md
│
└── README.md                     ← You are here
```

---

## Tech Stack

| Domain | Technology |
|--------|-----------|
| **Language (Backend)** | Java 21, Python 3.12 |
| **Frameworks** | Spring Boot 3, Spring Cloud Gateway, Spring WebFlux, FastAPI |
| **Security** | Spring Security 6, JWT, OAuth2 (Google) |
| **ORM** | Spring Data JPA + Hibernate 6 |
| **Databases** | PostgreSQL 16, TimescaleDB 2.x |
| **Cache** | Redis 7 |
| **Messaging** | Apache Kafka 3.7 |
| **Search / Storage** | Elasticsearch 8, MinIO |
| **AI/ML** | LangChain, Claude API (claude-sonnet-4-6), FinBERT, Celery |
| **Frontend** | React 18, TypeScript, Vite, TanStack Query, Zustand, Recharts, TradingView Charts |
| **Testing** | JUnit 5, Mockito, Testcontainers, Playwright, Newman (k6 for load) |
| **CI/CD** | Jenkins, SonarQube, Docker, Kubernetes |
| **Observability** | OpenTelemetry, Jaeger, Prometheus, Grafana, ELK Stack, structlog |
| **Secrets** | HashiCorp Vault |
| **Email** | AWS SES + Spring Mail + Thymeleaf |
| **Scheduler** | Quartz |
| **Resilience** | Resilience4j (Circuit Breaker) |

---

## Documentation Index

| Document | What it covers |
|----------|---------------|
| [Phase 0 — Setup](docs/phases/PHASE_0_SETUP.md) | Repo, Docker Compose, Flyway, design patterns |
| [Phase 1 — MVP](docs/phases/PHASE_1_MVP.md) | Auth, API Gateway, Portfolio, Market Data, Dashboard |
| [Phase 2 — Market](docs/phases/PHASE_2_MARKET.md) | TimescaleDB, WebSocket, Dividends, Alerts, SIP |
| [Phase 3 — Analytics](docs/phases/PHASE_3_ANALYTICS.md) | P&L engine, Risk metrics, News/Sentiment, Tax |
| [Phase 4 — Advanced](docs/phases/PHASE_4_ADVANCED.md) | Event calendar, Supply chain, Documents, Goals |
| [Phase 5 — AI & Production](docs/phases/PHASE_5_AI_PRODUCTION.md) | AI assistant, Reports, Observability, K8s |
| [Phase 6 — Extended](docs/phases/PHASE_6_EXTENDED.md) | Family tracking, SEC AI, Options, Insider data |
| [Auth Service](docs/services/AUTH_SERVICE.md) | JWT, OAuth2, Redis blacklist, full API |
| [Portfolio Service](docs/services/PORTFOLIO_SERVICE.md) | FIFO, CSV import, Kafka events, full API |
| [Market Data Service](docs/services/MARKET_DATA_SERVICE.md) | Provider cascade, WebSocket, TimescaleDB |
| [Analytics Service](docs/services/ANALYTICS_SERVICE.md) | TWR, XIRR, Sharpe, VaR, correlation |
| [AI Service](docs/services/AI_SERVICE.md) | ReAct agent, 13 tools, SSE streaming, prompt patterns |
| [Docker Setup](docs/infrastructure/DOCKER_SETUP.md) | Dev stack, container configs, health checks |
| [CI/CD Pipeline](docs/infrastructure/CI_CD_PIPELINE.md) | 9-stage Jenkins pipeline, blue-green deploy |
| [Database Schema](docs/infrastructure/DATABASE_SCHEMA.md) | All 18 tables, ERD, TimescaleDB hypertables |
| [Kafka Topology](docs/architecture/KAFKA_TOPOLOGY.md) | All 10 topics, producers, consumers, DLQ |
| [Design Patterns](docs/architecture/DESIGN_PATTERNS.md) | 9 patterns mapped to WealthPilot components |
| [Observability](docs/infrastructure/OBSERVABILITY.md) | OTel, Jaeger, ELK, Prometheus, structured logs |

---

## External API Providers

| Provider | Purpose | Tier | Phase |
|----------|---------|------|-------|
| Yahoo Finance (yfinance) | Primary price feed | Free (unofficial) | 1 |
| Twelve Data | Secondary price feed | 800 req/day free | 1 |
| Alpha Vantage | Tertiary price feed | 25 req/day free | 1 |
| Finnhub | News, dividends, earnings | 60 req/min free | 2 |
| ECB / RBI | Exchange rates | Free | 1 |
| Claude API (Anthropic) | AI chat + entity extraction | Pay-per-token | 3 |
| AWS SES | Email delivery | 62k/month free | 1 |
| MarineTraffic | Vessel tracking | ~$100/mo | 4 |
| AviationStack | Cargo flights | 100 req/mo free | 4 |

---

*Last updated: —  ·  Active sprint: Phase 0*
