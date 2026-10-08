# Design Patterns

> ADR-001 — Applied across WealthPilot microservices

Nine patterns applied. Each entry: what it is, where it lives in WealthPilot, why it was chosen over alternatives.

---

## 1. Strategy Pattern

**Where:** `MarketDataProvider` interface  
**Classes:** `YahooFinanceProvider`, `TwelveDataProvider`, `AlphaVantageProvider`

### What it does
Defines a family of algorithms (price fetching from different providers), encapsulates each one, and makes them interchangeable. `MarketDataService` calls the interface — it doesn't care which provider it's talking to.

### Why chosen
Provider APIs change, go down, or get rate-limited. Switching providers is a config change (`market.data.primary=YAHOO → TWELVE_DATA`), not a code change. Adding a fourth provider means adding one class — zero changes to `MarketDataService`.

### Why not alternatives
- Hard-coded if/else per provider → every provider change requires modifying `MarketDataService`
- Single provider → no resilience when provider is down

---

## 2. Chain of Responsibility

**Where:** Provider cascade in `MarketDataService.fetchQuote(symbol)`  
**Chain:** Yahoo Finance → Twelve Data → Alpha Vantage → Redis stale cache

### What it does
Passes a request along a chain of handlers. Each handler decides: handle it (return result) or pass it to the next. The chain stops at the first successful response.

### How it works in WealthPilot
```
fetchQuote(symbol):
  try YahooFinanceProvider.fetch()     → circuit OPEN → fall through
  try TwelveDataProvider.fetch()       → rate limited → fall through
  try AlphaVantageProvider.fetch()     → success → return
                                       → all fail → return Redis stale
```

Each provider is independently wrapped in a Resilience4j `CircuitBreaker`. A Yahoo outage does not affect the Twelve Data circuit state.

### Why chosen
Fallback logic stays out of `MarketDataService` itself. Adding a new fallback = inserting a new handler in the chain. The caller doesn't know or care which provider answered.

---

## 3. Observer Pattern

**Where:** Kafka topics as the system-wide event bus  
**Key topic:** `portfolio.events` → Analytics, Tax, Goals (three independent observers)

### What it does
Subject (Portfolio Service) publishes an event when state changes. Multiple observers (Analytics, Tax, Goals Services) react independently — none knows about the others.

### How it works in WealthPilot
```
Portfolio Service: publishes portfolio.events (the Subject)
  ↓ Kafka fan-out via consumer groups
Analytics Service: invalidates cache (observer A)
Tax Service:       classifies the realized gain (observer B)
Goals Service:     updates goal current_value (observer C)
```

Each observer has its own consumer group offset — the same Kafka message is read independently by all three.

### Why chosen
Portfolio Service has no compile-time dependency on Analytics, Tax, or Goals services. Adding a fourth observer (e.g., a new Reporting consumer) requires no change to Portfolio Service.

---

## 4. Builder Pattern

**Where:** Complex response DTOs — `PortfolioPerformanceResponse`, `MonthlyReportDTO`  
**Classes:** `PortfolioPerformanceResponse.Builder`, `MonthlyReportDTO.Builder`

### What it does
Constructs a complex object step-by-step using a fluent API. Each `with*` method sets one field and returns `this`. `.build()` validates and constructs the final object.

### Why chosen
`PortfolioPerformanceResponse` has 15+ fields assembled from 4 different service calls. Without Builder, the constructor would have 15 parameters (impossible to read) or require multiple setters (allows partially-constructed invalid state). Builder enforces: all required fields set before `build()` is called.

---

## 5. Repository Pattern

**Where:** All Spring Data JPA repositories across all Java services  
**Examples:** `PortfolioRepository`, `HoldingRepository`, `TransactionLotRepository`

### What it does
Abstracts the data access layer behind an interface. Service classes interact with `PortfolioRepository.findByUserId()`, not with `EntityManager.createQuery(...)`. The persistence mechanism is invisible to the service layer.

### Why chosen
Standard Spring Data pattern. Enables unit testing of service logic with mocked repositories (no DB needed). Switching from JPA to JDBC or a different ORM affects only the repository layer.

---

## 6. Factory Pattern

**Where:** `CostBasisCalculatorFactory`  
**Products:** `FifoCostBasisCalculator`, `LifoCostBasisCalculator`, `WeightedAverageCostBasisCalculator`

### What it does
Creates the correct cost basis calculator based on the user's preference stored in `user_preferences`. The factory reads the config and returns the appropriate implementation. `TransactionService` calls `factory.getCalculator(userId)` — it never knows which implementation it gets.

### Why chosen
Cost basis method is a user setting, not a code path. Adding a new method (e.g., Specific Identification) means adding one class + one factory branch. No changes to `TransactionService`.

---

## 7. Circuit Breaker Pattern

**Where:** Resilience4j wrapping all external provider calls  
**Applied to:** Yahoo Finance, Twelve Data, Alpha Vantage, MarineTraffic, AviationStack, Finnhub

### What it does
Monitors calls to an external service. When the failure rate exceeds a threshold, the circuit "opens" — subsequent calls fail immediately without hitting the external service. After a wait period, the circuit enters "half-open" and allows probe calls. If they succeed, the circuit closes.

```
States:
  CLOSED      → normal operation, all calls go through
  OPEN        → calls fail immediately (fallback used), no external call made
  HALF-OPEN   → probe calls allowed; success → CLOSED, failure → OPEN
```

### Config in WealthPilot (Yahoo Finance example)
```yaml
failure-rate-threshold: 50        # Open if 50% of last 10 calls fail
slow-call-duration-threshold: 2s  # Calls > 2s counted as failures
sliding-window-size: 10           # Last 10 calls evaluated
wait-duration-in-open-state: 30s  # Wait 30s before probing
```

### Why chosen
Without circuit breakers: one slow provider causes requests to pile up waiting for timeouts (cascade failure). With circuit breakers: provider outage is detected in seconds, fallback kicks in immediately, thread pool is not exhausted.

---

## 8. Template Method Pattern

**Where:** `AbstractBrokerCsvParser` and its subclasses  
**Subclasses:** `ZerodhaCsvParser`, `GrowwCsvParser`, `IBKRCsvParser`, `SchwabCsvParser`

### What it does
Defines the skeleton of an algorithm in a base class. Subclasses override specific steps without changing the algorithm's overall structure.

### The template in WealthPilot
```java
// AbstractBrokerCsvParser — the invariant algorithm
public final ImportPreviewDTO parse(MultipartFile file) {
  List<String[]> rows = readFile(file);         // common — OpenCSV
  String format = detectFormat(rows);            // optional override
  List<TransactionDTO> dtos = new ArrayList<>();
  for (String[] row : rows) {
    TransactionDTO dto = mapColumns(row);        // ABSTRACT — each broker overrides this
    if (validateRow(dto)) {                      // common
      if (!isDuplicate(dto)) {                   // common — hash check
        dtos.add(dto);
      }
    }
  }
  return new ImportPreviewDTO(dtos, errors);
}

// Subclass overrides only this:
protected abstract TransactionDTO mapColumns(String[] row);
```

### Why chosen
All 4 broker parsers share: file reading, date parsing, validation, deduplication logic. Only column mapping differs per broker. Without Template Method, this logic would be duplicated in every parser class.

---

## 9. Singleton Pattern

**Where:** FinBERT model loading in the News & Sentiment Service (Python)  
**Class:** `FinBertModelSingleton`

### What it does
Ensures a class has only one instance, loaded at process startup.

### Why chosen in WealthPilot
FinBERT model (`ProsusAI/finbert`) takes 2–3 seconds to load from disk into memory. If loaded per-request: 50 articles processed in parallel → 50 model loads → 2-3 second penalty per article. With Singleton: model loaded once at Celery worker startup → inference in milliseconds per article. Worker restart is the only downtime cost.

### Implementation (Python module-level)
```python
# model_singleton.py — loaded once when module is imported
from transformers import pipeline

_sentiment_pipeline = None

def get_sentiment_pipeline():
    global _sentiment_pipeline
    if _sentiment_pipeline is None:
        _sentiment_pipeline = pipeline(
            "text-classification",
            model="ProsusAI/finbert",
            tokenizer="ProsusAI/finbert"
        )
    return _sentiment_pipeline
```

Celery worker calls `get_sentiment_pipeline()` on first task. Subsequent tasks reuse the same loaded model.

---

## Pattern Application Map

| Pattern | Service | Class/Component |
|---------|---------|----------------|
| Strategy | Market Data | `MarketDataProvider`, `YahooFinanceProvider`, `TwelveDataProvider`, `AlphaVantageProvider` |
| Chain of Responsibility | Market Data | `MarketDataService.fetchQuote()` cascade |
| Observer | All (via Kafka) | `portfolio.events` → Analytics, Tax, Goals consumer groups |
| Builder | Portfolio, Report | `PortfolioPerformanceResponse.Builder`, `MonthlyReportDTO.Builder` |
| Repository | All Java services | All Spring Data JPA `*Repository` interfaces |
| Factory | Portfolio | `CostBasisCalculatorFactory`, `FIFO/LIFO/WeightedAverageCalculator` |
| Circuit Breaker | Market Data, Supply Chain | Resilience4j on all external HTTP calls |
| Template Method | Portfolio | `AbstractBrokerCsvParser`, `ZerodhaCsvParser`, `GrowwCsvParser`, `IBKRCsvParser` |
| Singleton | News/Sentiment | `FinBertModelSingleton` (Python module-level) |
