# Phase 6 — Extended Features + Polish

> Weeks 37+

**Status:** 🔴 Not Started  
**Prerequisite:** All Phase 5 exit criteria met. Production deployment stable.  
**Goal:** Premium features that differentiate WealthPilot from existing tools. Each feature here is independent — build in any order based on personal interest.

---

## Features

| Feature | Service | Priority | Status |
|---------|---------|----------|--------|
| 6.1 | Family Portfolio Tracking | Auth + Portfolio | High | 🔴 |
| 6.2 | Insider Trading + Institutional Holdings Tracker | New Service | Medium | 🔴 |
| 6.3 | SEC / SEBI Filing AI Summarization | AI Service | Medium | 🔴 |
| 6.4 | Option Chain Analysis | Market Data | Low | 🔴 |
| 6.5 | WhatsApp Notification Channel | Notification | Low | 🔴 |

---

## 6.1 — Family Portfolio Tracking

**Tech:** `Spring Boot 3` · `PostgreSQL` · `Spring Security` · `JPA/Hibernate`

### What it does
One "family account" can view portfolios of linked member accounts (spouse, parents) in a consolidated view. Linked members have read-only access — they cannot modify the owner's holdings.

### Data model

```
family_members (
  id UUID PK,
  owner_user_id UUID FK → users.id,
  member_user_id UUID FK → users.id,
  access_level ENUM(READ, NONE),
  invited_at TIMESTAMPTZ,
  accepted_at TIMESTAMPTZ,
  status ENUM(PENDING, ACTIVE, REVOKED)
)
```

### Access control
`@PreAuthorize` on every cross-user data endpoint:
```java
@PreAuthorize("hasPermission(#targetUserId, 'PORTFOLIO', 'READ')")
```
Custom `PermissionEvaluator` checks `family_members` table: if `owner_user_id = requestingUser AND member_user_id = targetUserId AND access_level = READ AND status = ACTIVE` → allow.

### Consolidated family view
`GET /api/v1/family/net-worth` → sums all portfolios of all linked members + owner. Returns breakdown per member.

### Invitation flow
Owner sends invite by email → member receives invite link → member accepts → `family_members.status` → ACTIVE. Invite link expires in 48 hours.

### Done when
- [ ] User A invites User B → B accepts → A can see B's portfolio
- [ ] User B cannot modify User A's holdings (403 on any write attempt)
- [ ] Family net worth sums all member portfolios correctly
- [ ] Revoking access → B's data no longer visible to A

---

## 6.2 — Insider Trading + Institutional Holdings Tracker

**Tech:** `Spring Boot 3` · `Python FastAPI` · `Elasticsearch` · `Apache Kafka` · `PostgreSQL`

### What it does
Fetches and indexes SEBI bulk/block deal disclosures (India) and SEC Form 4 insider filings (US). Tracks FII/DII/MF institutional holdings changes quarter over quarter.

### Data sources
- SEBI bulk deals: [sebi.gov.in bulk deals CSV](https://www.sebi.gov.in/sebiweb/other/OtherAction.do?doBulkDeal=yes) — scraped weekly
- SEC EDGAR Form 4: `https://www.sec.gov/cgi-bin/browse-edgar` — fetched via EDGAR API
- BSE shareholding pattern: quarterly PDF from BSE → AI extraction

### Pipeline
```
Scraper (Python) → new filings → Kafka: institutional.events
        ↓
Elasticsearch index: institutional-filings
        ↓
Dashboard widget: "FII buying INFY for 3 consecutive weeks"
        ↓
Notification: alert if FII holding % changes > 2% in one quarter
```

### Dashboard widget
"Institutional activity for your holdings" — per held stock: FII holding % (current + QoQ change), DII holding %, promoter holding %, public float %.

### Done when
- [ ] SEBI bulk deals indexed for last 90 days
- [ ] Dashboard shows FII/DII % for at least 5 held India stocks
- [ ] Alert fires when FII holding changes > 2% for a held stock

---

## 6.3 — SEC / SEBI Filing AI Summarization

**Tech:** `Python FastAPI` · `LangChain` · `Claude API` · `Prompt Engineering` · `LLM Orchestration`

### What it does
New AI Service tool: `summarize_filing`. Fetches SEC 10-K or 10-Q (or SEBI annual report) text, chunks it to fit Claude context, and returns a structured summary.

### Prompt
```
System: You are a financial analyst. Extract from this annual report:
        1. Top 3 risk factors (one sentence each)
        2. Revenue trend: last 3 years with % YoY change
        3. Management guidance for next year (verbatim if available)
        4. One key concern from the auditor's notes (if any)
        Return JSON only: {risks: [...], revenue: [...], guidance: "...", auditor_concern: "..."}

User: {filing_text_chunk}
```

### Chunking strategy
SEC 10-K can be 200+ pages. Strategy:
1. Fetch full text from SEC EDGAR
2. Split into 6,000-token chunks with 200-token overlap
3. Run Claude on each chunk with the extraction prompt
4. Merge results: deduplicate risks, aggregate revenue data, take first guidance mention

### Caching
Redis: `filing:{ticker}:{form_type}:{year}` → cached summary. TTL 7 days. Same filing never re-summarized.

### Done when
- [ ] `summarize_filing(ticker="AAPL", form="10-K", year=2025)` returns structured JSON
- [ ] Summary cached in Redis — second call returns instantly
- [ ] Risk factors, revenue trend, and guidance all populated

---

## 6.4 — Option Chain Analysis

**Tech:** `Spring Boot 3` · `TimescaleDB` · `WebSocket` · `React 18`

### What it does
Fetches option chain data for stocks. Displays strike price grid with OI, volume, IV, and Greeks. Live OI change updates via WebSocket.

### Data sources
- NSE options (India): NSE public option chain API
- US options: Tradier API or TD Ameritrade

### Data model
Extension of `instruments` to support `type=OPTION` with metadata:
```json
{
  "underlying": "INFY",
  "expiry": "2026-09-25",
  "strike": 1800.0,
  "option_type": "CE",
  "lot_size": 300
}
```

### Option chain table (frontend)
Grid: rows = strike prices, columns = CE OI / CE Volume / CE IV / Strike / PE IV / PE Volume / PE OI. Cells colored by OI concentration (highest OI = max pain signal). Updated every 3 minutes via WebSocket.

### Done when
- [ ] Option chain table renders for an India stock (e.g., INFY)
- [ ] OI and volume columns populated correctly
- [ ] Live update refreshes data every 3 minutes

---

## 6.5 — WhatsApp Notification Channel

**Tech:** `Spring Boot 3` · `Notification Service` · `Kafka`

### What it does
Delivers price alerts and weekly digest via WhatsApp in addition to email + push.

### Integration
WhatsApp Business API (via Twilio or Meta Cloud API). User adds their WhatsApp number in preferences. `alert_channels JSONB` updated to include `{"whatsapp": "+919876543210"}`.

Notification Service Kafka consumer on `alerts.triggered` and `reports.scheduled` → if `channels` includes WhatsApp → call WhatsApp Business API to send template message.

### Message templates (pre-approved by Meta)
- Price alert: "WealthPilot: {{symbol}} crossed ₹{{price}} (your alert). View at {{url}}"
- Weekly digest: "WealthPilot: Your weekly digest is ready. View at {{url}}"

### Done when
- [ ] WhatsApp number added in preferences
- [ ] Price alert fires → WhatsApp message received within 60 seconds
- [ ] Weekly digest → WhatsApp link received on Monday

---

## Phase 6 Priority Order

Build in this order based on impact-to-effort:

```
1. Family Portfolio (high impact, moderate effort — extends existing Auth/Portfolio)
2. SEC/SEBI Filing AI (high impact, low effort — extends existing AI Service tool catalog)
3. Insider Tracker (high impact, moderate effort — new scraping pipeline)
4. Option Chain (medium impact, high effort — new data model + frontend)
5. WhatsApp Channel (low-medium impact, low effort — extends Notification Service)
```

→ Return to [README](../../README.md) for overall status tracking.
