# AI Service

**Language:** Python 3.12  
**Framework:** FastAPI  
**Phase:** 5  
**Status:** 🔴 Not Started

---

## Responsibility
Conversational AI assistant for portfolio queries. LangChain ReAct agent with 13 portfolio-specific tools. Streaming responses via SSE. Document PDF analysis. All LLM calls go through this service.

## Stack
| Component | Technology |
|-----------|-----------|
| Framework | FastAPI (async) |
| LLM | Claude API (claude-sonnet-4-6) |
| Agent | LangChain ReAct + tool calling |
| Output parsing | Pydantic + PydanticOutputParser |
| Conversation history | Redis (list, 50-message rolling window) |
| Streaming | FastAPI StreamingResponse + SSE |
| PDF parsing | PyMuPDF (fitz) |
| Task queue | Celery + Redis (async document analysis) |
| Logging | structlog (JSON output) |

## Package Structure
```
ai-service/
├── main.py                    -- FastAPI app + router registration
├── config/
│   └── settings.py            -- Vault-sourced secrets (Claude API key, service URLs)
├── routers/
│   └── chat.py                -- POST /chat, POST /chat/stream, POST /chat/new
├── agent/
│   ├── react_agent.py         -- LangChain ReAct agent setup
│   ├── system_prompt.py       -- Jinja2 system prompt template
│   └── tools/
│       ├── portfolio_tools.py     -- get_portfolio_summary, get_allocation
│       ├── analytics_tools.py     -- get_pnl, get_top_performers, get_sector_exposure, get_risk_metrics, get_dividend_forecast, get_correlation_pairs
│       ├── market_tools.py        -- get_live_quote, get_events_upcoming
│       ├── tax_tools.py           -- run_tax_harvest
│       ├── goals_tools.py         -- run_goal_projection
│       ├── news_tools.py          -- get_news_sentiment
│       └── supply_chain_tools.py  -- get_supply_chain_context
├── memory/
│   └── conversation_store.py  -- Redis chat history CRUD
├── document/
│   ├── extractor.py           -- PyMuPDF text extraction
│   └── analyzer.py            -- Claude structured extraction
└── schemas/
    ├── chat.py                -- ChatRequest, ChatResponse
    └── document.py            -- ExtractionResult
```

## ReAct Agent Flow

```
User message
     ↓
Build system prompt (Jinja2 + live portfolio snapshot)
     ↓
Inject conversation history from Redis
     ↓
Claude API call (model: claude-sonnet-4-6)
     ↓
Claude output: THOUGHT → ACTION → tool args
     ↓
LangChain dispatcher: call named tool function
     ↓
Tool returns data (from downstream service REST call)
     ↓
Inject OBSERVATION into next Claude call
     ↓
Repeat until FINAL_ANSWER or max_iterations=5
     ↓
Stream response tokens to client via SSE
```

## Tool Catalog

| Tool name | Calls | Returns |
|-----------|-------|---------|
| `get_portfolio_summary` | Portfolio Service `/portfolios/consolidated` | Holdings list, total value, allocation % |
| `get_pnl` | Analytics Service `/analytics/pnl` | Unrealized + realized P&L, period |
| `get_top_performers` | Analytics Service `/analytics/pnl` | Holdings sorted by return %, best and worst |
| `get_sector_exposure` | Analytics Service `/analytics/sector` | Sector breakdown % |
| `get_dividend_forecast` | Analytics Service `/analytics/dividends` | Projected 12-month income |
| `get_live_quote` | Market Data `/market/quote/{symbol}` | Price + change |
| `run_tax_harvest` | Tax Service `/tax/harvest` | Candidates + savings amounts |
| `run_goal_projection` | Goals Service `/goals/{id}/projection` | P50 + probability % |
| `get_news_sentiment` | News Service `/news/sentiment/{symbol}` | Score + last 5 headlines |
| `get_events_upcoming` | Market Data `/events/upcoming` | Next 7 days' events |
| `get_risk_metrics` | Analytics Service `/analytics/risk` | Sharpe + Sortino + VaR + Beta |
| `get_allocation` | Portfolio Service `/portfolios/{id}/allocation` | Current asset allocation |
| `get_correlation_pairs` | Analytics Service `/analytics/correlation-matrix` | N×N matrix |
| `get_supply_chain_context` | Supply Chain `/supply-chain/analytics` | Vessel count + port congestion |

## Tool Description Pattern (prevents wrong tool selection)

Each tool's LangChain `@tool` description is precise and mutual-exclusive:

```python
@tool
def get_pnl(period: str = "ALL") -> dict:
    """
    Fetch portfolio profit and loss data.
    Use this when the user asks about gains, losses, returns, or performance over a time period.
    period options: "1D", "1W", "1M", "3M", "1Y", "ALL"
    
    Do NOT use for:
    - Current allocation (use get_allocation instead)
    - Individual stock prices (use get_live_quote instead)
    - Risk metrics like Sharpe (use get_risk_metrics instead)
    """
```

## System Prompt Template (Jinja2)

```jinja
You are WealthPilot, a personal investment advisor for {{ user.name }}.
Today is {{ today }}. User's base currency is {{ user.base_currency }}.

Current portfolio snapshot:
Total value: {{ total_value | currency }}
Day's gain/loss: {{ day_change | currency }} ({{ day_change_pct }}%)

Top 10 holdings:
| Symbol | Value | P&L % |
{% for h in top_holdings %}
| {{ h.symbol }} | {{ h.value | currency }} | {{ h.pnl_pct }}% |
{% endfor %}

Instructions:
- Use tools to fetch precise data. Never guess numbers.
- Be concise: 3-5 sentences unless a table is more appropriate.
- If you cannot find the data with 5 tool calls, say so honestly.
- Always cite which data came from which tool.
```

## Conversation History (Redis)

```
Key: chat:{userId}:history
Type: Redis List (RPUSH to append, LRANGE to read all)
Rolling window: LTRIM to keep last 50 messages

Entry format: JSON string
{
  "role": "user" | "assistant",
  "content": "message text",
  "ts": "2026-08-01T10:00:00Z"
}
```

`POST /api/v1/ai/chat/new` → `DEL chat:{userId}:history`

## SSE Streaming

```python
@router.post("/chat/stream")
async def chat_stream(request: ChatRequest, user_id: str = Header(alias="X-User-Id")):
    async def generate():
        async for token in claude_stream(messages, tools):
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

React frontend connects via `EventSource` and renders each token as it arrives.

## Document PDF Extraction

```
POST /api/v1/documents/analyze (called by Document Service after PyMuPDF text extraction)

Input: {documentId, userId, extractedText}
Output: {transactions: [{date, symbol, type, quantity, price, fees}]}

Process:
1. Split extractedText into 6,000-token chunks (200-token overlap)
2. Run Claude extraction prompt on each chunk
3. json.loads() each response (no free-text parsing)
4. Merge results, deduplicate by (symbol, date, type, quantity)
5. Return merged transaction list
```

## Token Budget

| Call type | max_tokens |
|-----------|-----------|
| Chat response | 1,000 |
| Entity extraction | 200 |
| Article summary | 150 |
| PDF extraction (per chunk) | 2,000 |
| Deep analysis | 2,000 |

Monthly spend: tracked via Anthropic API. Hard cap: `$50/month`. If exceeded → return "monthly AI budget reached" with 429 status.

## Build Checklist
- [ ] FastAPI app starts, `/api/v1/ai/chat` responds
- [ ] "What's my best stock?" → `get_top_performers` tool called → real holding name in response
- [ ] SSE streaming: tokens appear progressively in React UI
- [ ] Conversation history maintained across 5+ turns
- [ ] New conversation clears Redis history
- [ ] `get_supply_chain_context` tool called when asked about crude oil (verify in logs)
- [ ] Token budget: after hitting cap, 429 returned with clear message
- [ ] `structlog` JSON visible in Docker logs for every tool call
