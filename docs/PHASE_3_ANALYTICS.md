# Phase 3 — Analytics + News Intelligence + Tax

> Weeks 13–20

**Status:** 🔴 Not Started  
**Goal:** Advanced portfolio analytics, news sentiment tied to your holdings, and a working tax engine.

---

## Steps

| Step | Title | Service | Status |
|------|-------|---------|--------|
| 3.1 | Analytics Service: P&L Engine | Analytics | 🔴 |
| 3.2 | Analytics Service: Risk Metrics | Analytics | 🔴 |
| 3.3 | News & Sentiment Service | News/Sentiment | 🔴 |
| 3.4 | Prompt Engineering Catalog | AI / News | 🔴 |
| 3.5 | Tax Service: STCG/LTCG + Tax Harvest | Tax | 🔴 |
| 3.6 | Rebalancing Engine | Portfolio | 🔴 |

---

## Step 3.1 — Analytics Service: P&L Engine

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `TimescaleDB` · `Redis` · `Apache Kafka` · `SQL` · `Query Optimization` · `DSA`

### What it does
Portfolio-level P&L, Time-Weighted Return, XIRR, period-based attribution, and benchmark comparison.

### Cache invalidation pattern
Consumes `portfolio.events` from Kafka → deletes Redis key `analytics:{userId}:{portfolioId}:*`. Lazy recompute — next API request triggers fresh calculation. No eager recompute.

### TWR algorithm (removes cash-flow timing distortion)
```
Sub-periods: divided by each BUY/SELL event
Sub-period return = (end_value / start_value) - 1
TWR = PRODUCT(1 + r_i for all i) - 1
```
Portfolio value at each boundary reconstructed from `price_history` × holdings at that date.

### XIRR algorithm
Newton-Raphson (identical to Step 2.7). Applied at portfolio level: all BUY = negative cash flows, all SELL = positive, current value = final positive. Max 200 iterations, tolerance 1e-7.

### Benchmark comparison
`GET /analytics/performance?benchmark=NIFTY50` fetches NIFTY 50 `price_history` and computes normalized return from user's first investment date. Both series rebased to 100.

### FX-adjusted P&L
Per-holding: `asset_pnl`, `fx_pnl`, `total_pnl` (from Step 2.5 formula). Summed at portfolio level.

### Sector attribution
Group holdings by sector → sum unrealized P&L per sector → return sorted by absolute contribution.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/analytics/pnl` | Overall P&L: unrealized + realized + TWR + XIRR |
| `GET` | `/api/v1/analytics/pnl/realized` | Realized P&L per holding + overall |
| `GET` | `/api/v1/analytics/performance` | `?period=1Y&benchmark=NIFTY50` |
| `GET` | `/api/v1/analytics/sector` | Sector P&L attribution |
| `GET` | `/api/v1/analytics/net-worth/history` | Net worth time-series |

### Done when
- [ ] TWR matches manual calculation for a known cash flow sequence
- [ ] XIRR matches Excel for same portfolio inputs
- [ ] Benchmark overlay data returned correctly for NIFTY 50
- [ ] Sector attribution sums to 100% of portfolio P&L
- [ ] Redis cache invalidated on `portfolio.events` receipt
- [ ] Second request hits Redis (response time < 10ms vs first 200ms+)

---

## Step 3.2 — Analytics Service: Risk Metrics

**Tech:** `Spring Boot 3` · `PostgreSQL` · `TimescaleDB` · `SQL` · `Query Optimization` · `DSA`

### Risk metric formulas

**Daily returns** (foundation for all):
```
daily_return[i] = (close[i] - close[i-1]) / close[i-1]
```
Portfolio daily return = weighted sum of each holding's daily return by allocation %.

**Sharpe Ratio:**
```
(annualized_return - risk_free_rate) / (std_dev(daily_returns) × sqrt(252))
```
Risk-free rate: current 91-day T-bill (India) or Fed Funds rate (US), stored in `reference_rates` table.

**Sortino Ratio:**
```
(annualized_return - risk_free_rate) / (std_dev(negative_daily_returns) × sqrt(252))
```
Only penalizes downside volatility.

**Beta:**
```
cov(portfolio_returns, benchmark_returns) / var(benchmark_returns)
```

**Maximum Drawdown:**
```
For each day: drawdown = (current_value - peak_value) / peak_value
              peak_value = MAX(portfolio_value up to that day)
Max Drawdown = MIN(drawdown_series)
```

**Value at Risk (95%, historical simulation):**
```
VaR_pct = 5th percentile of sorted daily_returns (ascending)
VaR_amount = VaR_pct × current_portfolio_value
```
Non-parametric — no normal distribution assumption.

**Correlation matrix:**
Pairwise Pearson correlation of daily returns for all N holdings. N×N symmetric matrix stored as JSONB in Redis.

### Critical query optimization
Fetch 252 days of daily returns for all holdings in one query:
```sql
SELECT instrument_id, ts, close
FROM price_history
WHERE instrument_id = ANY($1)
  AND ts >= NOW() - INTERVAL '1 year'
ORDER BY instrument_id, ts
```
Composite index `(instrument_id, ts DESC)` makes this a single index scan per instrument.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/analytics/risk` | Sharpe + Sortino + Beta + VaR + Max Drawdown |
| `GET` | `/api/v1/analytics/correlation-matrix` | N×N matrix |
| `GET` | `/api/v1/analytics/currency-exposure` | Currency allocation breakdown |

### Done when
- [ ] Sharpe Ratio result matches textbook formula for known inputs
- [ ] VaR 95% value is the 5th percentile of sorted daily returns (verified)
- [ ] Correlation matrix is symmetric, values between -1 and 1
- [ ] Risk metrics query returns in < 1s (EXPLAIN ANALYZE verified)

---

## Step 3.3 — News & Sentiment Service

**Tech:** `Python FastAPI` · `FinBERT` · `LangChain` · `Claude API` · `Celery` · `Apache Kafka` · `Elasticsearch` · `PostgreSQL` · `NLP` · `LLM Orchestration` · `Prompt Engineering` · `GenAI Pipelines` · `Event-Driven Architecture` · `Structured Logging`

### Pipeline overview

```
Finnhub / NewsAPI
       ↓  (poll every 5 min)
FastAPI background task: dedup by URL hash
       ↓  publish
Kafka: news.raw
       ↓  consume
Celery Worker:
  1. FinBERT inference → sentiment_score, sentiment_label
  2. Claude API (async) → ticker entity extraction
  3. Publish to Kafka: news.scored
       ↓  consume (3 independent consumers)
  A. Elasticsearch index: news-articles
  B. PostgreSQL: news_articles table
  C. Notification Service: alerts for sentiment < -0.7
```

### FinBERT inference
Model: `ProsusAI/finbert` loaded once at Celery worker startup (singleton — avoids 2–3s load per request). Input: article title + first 2 sentences (max 512 tokens). Output: `{label: POSITIVE/NEGATIVE/NEUTRAL, score: 0.0–1.0}`. Mapped to internal: `{sentiment_score: -1.0 to +1.0, sentiment_label: BULLISH/BEARISH/NEUTRAL}`.

### Claude entity extraction
Separate Celery task (async after FinBERT). Prompt: extracts ticker symbols from article text. Response parsed with `json.loads()`. Result updates the article record when ready. FinBERT result is not delayed by Claude latency.

### Dead Letter Queue
Articles failing FinBERT after 3 retries → `news.raw.dlq`. Does not block main pipeline. DLQ messages are inspectable manually.

### Near-duplicate detection
Article title → SimHash. Hamming distance < 3 with any indexed article → skip. Prevents 10 outlets publishing the same Reuters story from filling the feed.

### Elasticsearch index
Index: `news-articles`. Mappings: `title (text, analyzed)`, `source (keyword)`, `tickers (keyword array)`, `sentiment_score (float)`, `published_at (date)`.

### Structured logging (Python)
All Python services use `structlog` with JSON output:
```json
{"event": "article_scored", "article_id": "...", "sentiment": 0.73, "tickers": ["AAPL"], "latency_ms": 45, "service": "news-sentiment", "ts": "..."}
```
Ingested by Logstash → Elasticsearch → Kibana dashboards.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/news` | Latest news for user's held tickers |
| `GET` | `/api/v1/news/{symbol}` | Last 20 articles for a symbol |
| `GET` | `/api/v1/news/sentiment/{symbol}` | Sentiment score + 30-day trend |
| `GET` | `/api/v1/news/sector-sentiment` | Sector-level sentiment heatmap data |
| `GET` | `/api/v1/news/search` | Full-text search across indexed articles |

### Done when
- [ ] Finnhub poller runs every 5 minutes, new articles appear in Elasticsearch
- [ ] FinBERT scores articles: inspect `sentiment_score` field in index
- [ ] Claude extracts tickers: verify `tickers` field populated in article record
- [ ] Dashboard news feed shows only articles for held symbols
- [ ] Sector sentiment heatmap returns non-empty data
- [ ] DLQ: manually corrupt an article payload → it appears in `news.raw.dlq`
- [ ] `structlog` JSON visible in Docker logs for every article processed

---

## Step 3.4 — Prompt Engineering Catalog

**Tech:** `Prompt Engineering` · `LLM Orchestration` · `GenAI Pipelines`

### What it does
Documents every Claude API call pattern in the system. Structured prompts, response parsing strategies, token budgets. Stored in `docs/architecture/PROMPT_CATALOG.md`.

### Ticker entity extraction prompt (News Service)
```
System: You are a financial data extractor. Extract all stock ticker symbols mentioned 
        in the article. Return ONLY a JSON array: ["AAPL", "MSFT"]. 
        If no tickers, return []. Do not include index symbols like SPX, NDX.

User: {article_title}

{article_summary}
```
Response parsed with `json.loads()`. Validated against `instruments` table after extraction. `max_tokens: 200`.

### Article summary prompt (News Service — secondary)
```
System: You are a financial news summarizer. Summarize in exactly 2 sentences.
        Focus on: what happened, which companies/assets are affected, market impact.
        Return only the 2-sentence summary, no preamble.

User: {article_title}

{article_body}
```
`max_tokens: 150`.

### PDF document extraction prompt (Document Service)
```
System: Extract all brokerage transactions from this document.
        Return JSON only: {"transactions": [{"date": "YYYY-MM-DD", "symbol": "...", 
        "type": "BUY|SELL|DIVIDEND", "quantity": 0, "price": 0.0, "fees": 0.0}]}
        If a field is not found, use null. No other output.

User: {extracted_pdf_text}
```
`max_tokens: 4000`.

### AI chat system prompt (AI Service)
```
System: You are WealthPilot, a personal investment advisor. 
        You have access to the user's portfolio data via tools.
        Today is {date}. The user's base currency is {base_currency}.
        
        Current portfolio summary:
        Total value: {total_value}
        Top 10 holdings: {holdings_table}
        
        Use tools to fetch precise data before answering. 
        Do not guess numbers — always call a tool.
        Be concise. Respond in 3–5 sentences unless a table is more appropriate.

User: {message}
```

### Token budget management
Every Claude API call sets `max_tokens` appropriate to task:

| Use case | max_tokens |
|----------|-----------|
| Ticker extraction | 200 |
| Article summary | 150 |
| PDF extraction | 4000 |
| AI chat response | 1000 |
| Deep analysis report | 2000 |

Monthly spend tracked via Anthropic API usage endpoint. Hard cap enforced: if month-to-date spend > $50, AI chat returns "monthly AI budget reached" message.

### Done when
- [ ] `PROMPT_CATALOG.md` documents all 5 Claude call patterns with exact prompts
- [ ] Each prompt has a `max_tokens` set and response parsed deterministically (no free-text parsing)
- [ ] Token budget tracking implemented in AI Service

---

## Step 3.5 — Tax Service: STCG/LTCG + Tax Harvest

**Tech:** `Spring Boot 3` · `JPA/Hibernate` · `PostgreSQL` · `SQL` · `Apache Kafka` · `Event-Driven Architecture`

### What it does
Classifies realized gains, identifies tax-loss harvesting candidates, generates tax report for ITR filing.

### India tax rules
| Type | Condition | Rate |
|------|-----------|------|
| Equity STCG | Held < 1 year | 15% flat |
| Equity LTCG | Held ≥ 1 year | 10% above ₹1L annual exemption |
| Debt MF (post-Apr 2023) | Any period | Slab rate |

### US rules
- Short-term (< 1 year): ordinary income rate
- Long-term (≥ 1 year): preferential capital gains rate
- Wash sale: loss disallowed if same/substantially identical security purchased within 30 days before/after sale at a loss

### Kafka trigger
Consumes `portfolio.events` on SELL events → fetches realized gain from Portfolio Service → classifies.

### Tax-loss harvesting
`GET /api/v1/tax/harvest` queries holdings where `current_price < avg_cost_price`. For each: `potential_loss = (avg_cost_price - current_price) × quantity`, `tax_saving = potential_loss × applicable_rate`. Returns ranked list.

### Sale simulation (dry-run)
`POST /api/v1/tax/simulate` — computes tax impact without inserting any transaction. Same classification logic, writes nothing to DB.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/tax/summary` | STCG + LTCG totals + estimated tax |
| `GET` | `/api/v1/tax/transactions` | All realized transactions with tax classification |
| `GET` | `/api/v1/tax/harvest` | Tax-loss harvesting candidates ranked by savings |
| `GET` | `/api/v1/tax/estimate` | Quarterly estimated tax liability |
| `POST` | `/api/v1/tax/simulate` | Dry-run sale → tax impact without committing |
| `GET` | `/api/v1/tax/report` | Full year tax report for ITR filing |

### Done when
- [ ] STCG/LTCG classification correct for a sold position (holding period < 1 year = STCG at 15%)
- [ ] Wash sale detection fires for US holding (same symbol bought within 30 days of loss-sale)
- [ ] Tax harvest candidates list non-empty with correct savings amounts
- [ ] Dry-run sale returns correct tax impact without creating transaction record

---

## Step 3.6 — Rebalancing Engine

**Tech:** `Spring Boot 3` · `PostgreSQL` · `SQL` · `REST API`

### What it does
Compares current portfolio allocation vs target allocation and generates exact trade suggestions to restore balance. Tax-aware: prefers selling LTCG positions.

### Drift calculation
```
current_pct[class] = sum(holding_value[class]) / total_portfolio_value × 100
drift[class] = current_pct[class] - target_pct[class]
```

### Trade suggestion generation
Over-weight class: suggest selling from holdings in that class, sorted by highest LTCG first (minimize tax).  
Under-weight class: suggest buying. Total sell proceeds = total buy amounts (no net cash out).

### Monthly drift alert
Notification Service checks drift on 1st of each month. Drift > threshold (default 5%) → sends rebalancing email.

### API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/portfolios/{id}/rebalance` | Drift analysis + trade suggestions |
| `PUT` | `/api/v1/portfolios/{id}/target-allocation` | Set target allocation |
| `GET` | `/api/v1/portfolios/{id}/allocation` | Current vs target allocation |

### Done when
- [ ] Target allocation set: 60% Equity / 30% Debt / 10% Gold
- [ ] After buying too much equity: drift detected, correct sell suggestions generated
- [ ] Trade suggestions sum correctly (sell proceeds = buy amounts)

---

## Phase 3 Exit Criteria

- [ ] Sharpe / Sortino / VaR / Max Drawdown visible in analytics UI
- [ ] Correlation heatmap renders
- [ ] TWR and XIRR visible and correct
- [ ] News feed shows articles for held tickers with sentiment badges
- [ ] Sector sentiment heatmap colored correctly
- [ ] STCG/LTCG classification correct for a sold position
- [ ] Tax harvest candidates ranked by savings amount
- [ ] All Claude prompts documented with `max_tokens` and deterministic response parsing

→ Next: [Phase 4 — Calendar + Supply Chain + Documents + Goals](PHASE_4_ADVANCED.md)
