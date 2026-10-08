# Observability

**Structured Logging** · **Distributed Tracing** · **Metrics** · **Log Aggregation**

Stack: OpenTelemetry + Jaeger + Prometheus + Grafana + ELK (Elasticsearch + Logstash + Kibana) + structlog

---

## Overview

```
Request enters API Gateway
        │
        ▼
Root traceId generated (UUID)
        │
        ▼
Injected as X-Correlation-Id header into every downstream call
        │
        ▼
OpenTelemetry agent auto-instruments:
  Spring MVC controllers → spans
  JPA queries → spans
  Kafka producers/consumers → spans
  Redis commands → spans
        │
        ▼
Spans exported to Jaeger (OTLP protocol)
        │
        ▼
Jaeger UI: full request path visible as waterfall trace
```

---

## 1. Structured Logging

### Java Services (Logback + logstash-logback-encoder)

Every log statement emits JSON on a single line. No multi-line stack traces. Fields are consistent across all 10 Java services.

**Required fields on every log line:**
```json
{
  "timestamp": "2026-08-01T10:30:00Z",
  "level": "INFO",
  "service": "portfolio-service",
  "traceId": "abc123def456789",
  "spanId": "789ghi012jkl",
  "userId": "uuid",
  "event": "holding_created",
  "message": "Holding created successfully"
}
```

**Additional context fields (business events):**
```json
{
  "holdingId": "uuid",
  "portfolioId": "uuid",
  "symbol": "AAPL",
  "quantity": 100,
  "durationMs": 23
}
```

**`traceId` and `spanId`** are injected automatically by the OTel agent via MDC (Mapped Diagnostic Context). No manual code.

**Logback config (`logback-spring.xml`):**
```xml
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeCallerData>false</includeCallerData>
    <customFields>{"service":"${SERVICE_NAME}"}</customFields>
  </encoder>
</appender>
```

### Python Services (structlog)

Same JSON schema as Java services. Configured once in `config/logging.py`, imported by all modules.

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

log = structlog.get_logger()

# Usage:
log.info("article_scored",
    article_id=article_id,
    sentiment=score,
    tickers=tickers,
    latency_ms=elapsed,
    service="news-sentiment"
)
```

---

## 2. Distributed Tracing (OpenTelemetry + Jaeger)

### OTel Java Agent

Attached to every Java service via JVM flag in Dockerfile:
```dockerfile
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
ENV OTEL_SERVICE_NAME=portfolio-service
ENV OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production

ENTRYPOINT ["java", \
  "-javaagent:/otel-agent.jar", \
  "-Dotel.service.name=${SERVICE_NAME}", \
  "-jar", "app.jar"]
```

**Auto-instrumented (zero code changes):**
- `@RestController` methods → spans with HTTP method, path, status code
- `@Repository` JPA queries → spans with SQL query (truncated)
- Kafka `KafkaProducer.send()` + `KafkaConsumer.poll()` → spans with topic name
- `RedisTemplate` operations → spans with key and command
- `WebClient` outbound HTTP → spans with target URL and status

### Correlation ID Propagation

API Gateway generates a UUID at request entry:
```java
// API Gateway GlobalFilter
String correlationId = UUID.randomUUID().toString();
exchange.getRequest().mutate()
    .header("X-Correlation-Id", correlationId)
    .header("traceparent", "00-" + correlationId + "-0-01");
```

The `traceparent` header is the W3C Trace Context standard — OTel agents on downstream services automatically pick it up and create child spans under the same trace.

### Jaeger Trace Example

**Request:** `GET /api/v1/portfolios/{id}/performance?period=1Y`

```
API Gateway                     5ms  (root span)
  └─ Portfolio Service         15ms
       └─ Holdings Query (PG)   8ms  -- "SELECT * FROM holdings WHERE portfolio_id = ?"
  └─ Market Data Service       45ms
       └─ Redis cache hit        2ms  -- "GET quote:AAPL"
       └─ Redis cache hit        2ms  -- "GET quote:INFY"
  └─ Analytics Service        120ms
       └─ Price History Query   85ms  -- "SELECT close FROM price_history WHERE..."
       └─ TimescaleDB agg.      30ms  -- continuous aggregate query
       └─ Redis write            3ms  -- cache analytics result
Total: ~185ms
```

Searchable in Jaeger UI by: `traceId`, `userId` (custom tag), service name, operation name.

### OTel Python (for Python services)

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

# FastAPI auto-instrumented
FastAPIInstrumentor().instrument()
# HTTPX (used for service calls) auto-instrumented
HTTPXClientInstrumentor().instrument()
```

---

## 3. Metrics (Prometheus + Grafana)

### Spring Boot Actuator

All Java services expose:
```
GET /actuator/prometheus
```

Prometheus scrapes every pod every 15 seconds.

**Key metrics collected:**

| Metric | What it measures |
|--------|----------------|
| `http_server_requests_seconds` | Latency histogram per endpoint |
| `jvm_memory_used_bytes` | JVM heap + non-heap usage |
| `jvm_gc_pause_seconds` | GC pause time |
| `hikaricp_connections_pending` | Connection pool pressure |
| `kafka_producer_record_send_rate` | Kafka publish throughput |
| `kafka_consumer_lag` | Consumer lag per topic-partition |
| `cache_gets_total{result="hit"}` | Redis cache hit rate |

### Grafana Dashboards

**Dashboard 1: Service Health Overview**
- Request rate per endpoint (last 1 hour)
- Error rate (4xx + 5xx) per service
- P50 / P95 / P99 latency per service
- Active pods per service (from K8s)

**Dashboard 2: JVM Internals**
- Heap used vs max
- GC pause time (ms, last 5 min)
- Thread pool active vs idle
- HikariCP connections in use

**Dashboard 3: Infrastructure**
- Kafka consumer lag per topic (alert if > 10,000)
- Redis memory usage
- Redis hit rate % (alert if < 80%)
- TimescaleDB chunk count + compressed vs uncompressed

**Dashboard 4: AI Service**
- Claude API call latency (P95)
- Claude API token usage (daily spend)
- Tool call frequency per tool name
- SSE connection count (active chat sessions)

### Alerts (Prometheus AlertManager)

| Alert | Condition | Severity |
|-------|-----------|---------|
| High error rate | Error rate > 5% for 5 minutes | Warning |
| Very high error rate | Error rate > 20% for 2 minutes | Critical |
| High latency | P95 > 2s for 5 minutes | Warning |
| Kafka consumer lag | Lag > 10,000 for 3 minutes | Warning |
| Redis hit rate low | Hit rate < 60% for 5 minutes | Warning |
| Pod crash loop | Restart count > 3 in 5 minutes | Critical |

---

## 4. Log Aggregation (ELK Stack)

### Architecture

```
Docker containers (all services)
        │  JSON logs to stdout
        ▼
Filebeat (runs on each K8s node)
        │  reads container log files
        ▼
Logstash
        │  parse JSON, add kubernetes metadata
        ▼
Elasticsearch: indices logs-{service}-{date}
        │
        ▼
Kibana: search, dashboards, alerts
```

### Logstash Pipeline

```ruby
input { beats { port => 5044 } }

filter {
  json { source => "message" }
  mutate {
    add_field => {
      "k8s_pod" => "%{[kubernetes][pod][name]}"
      "k8s_namespace" => "%{[kubernetes][namespace]}"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"
  }
}
```

### Kibana Queries

**Find all errors for a service:**
```
service: "portfolio-service" AND level: "ERROR"
```

**Find all requests for a specific user:**
```
userId: "uuid-here" AND event: "holding_created"
```

**Find slow requests:**
```
service: "analytics-service" AND durationMs: >500
```

**Trace a specific request across all services:**
```
traceId: "abc123def456789"
```

### Kibana Dashboards

**Dashboard: Error Rate by Service (last 24h)**
- Bar chart: error count per service per hour
- Table: top 10 error messages by frequency

**Dashboard: API Latency Heatmap**
- Heatmap: latency buckets (0-50ms, 50-200ms, 200-500ms, >500ms) per endpoint

**Dashboard: AI Usage**
- Timeline: Claude API calls per hour
- Tool call distribution (which tools called most)
- Daily token spend

---

## 5. Health Checks

Every service exposes Spring Boot Actuator health endpoints:

```
GET /actuator/health          → overall health (liveness probe)
GET /actuator/health/readiness → readiness (traffic routing)
GET /actuator/health/liveness  → liveness (restart trigger)
```

Kubernetes pod spec:
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3   # 3 consecutive failures → pod restart

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
  failureThreshold: 3   # 3 consecutive failures → pod removed from service
```

Actuator health includes DB connectivity, Redis ping, Kafka producer state, Elasticsearch ping — all configurable contributors.
