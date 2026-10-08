# CI/CD Pipeline

**Tool:** Jenkins  
**Per-service:** One `Jenkinsfile` per service in `infrastructure/jenkins/`  
**Shared library:** `infrastructure/jenkins/shared-lib/` — common stage definitions  
**Total stages:** 9

---

## Pipeline Overview

```
git push → webhook → Jenkins
    │
    ▼
Stage 1: Checkout
    ↓
Stage 2: Build
    ↓
Stage 3: Unit Tests          ← JUnit 5 + Mockito
    ↓
Stage 4: Integration Tests   ← Testcontainers (real PG + Redis + Kafka)
    ↓
Stage 5: Static Analysis     ← SonarQube + SpotBugs (Java) / Bandit (Python)
    ↓
Stage 6: Docker Build + Push ← Tagged with Git SHA
    ↓
Stage 7: Deploy to Staging   ← kubectl + rollout status
    ↓
Stage 8: Integration Tests on Staging ← Newman (Postman) + Playwright
    ↓
Stage 9: Promote to Production ← Manual approval → blue-green deploy
```

---

## Stage Breakdown

### Stage 1 — Checkout
```groovy
checkout scm
// Sets GIT_COMMIT environment variable
env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
```
Triggered by GitHub webhook on PR open or push to `main`.

**Failure behavior:** N/A (always succeeds)

---

### Stage 2 — Build

**Java services:**
```bash
mvn clean package -DskipTests -Dmaven.test.skip=true
```
Produces fat JAR at `target/{service}-{version}.jar`.

**Python services:**
```bash
pip install -r requirements.txt
python -m py_compile main.py  # syntax check
```

**Failure behavior:** Stop pipeline. PR blocked. Compilation error must be fixed before merge.

**Target time:** < 2 minutes

---

### Stage 3 — Unit Tests

**Java services:**
```bash
mvn test
```
JUnit 5 + Mockito. No containers. Pure in-memory. Fast.

**Python services:**
```bash
pytest tests/unit/ -v --tb=short
```

**Failure behavior:** Stop pipeline. PR blocked. Failing test must be fixed.

**Target time:** < 30 seconds per service

**What Mockito mocks:**
- `KafkaProducer` — verify event payload without real Kafka
- External HTTP clients (`MarketDataProvider`, `YahooFinanceProvider`) — WireMock or pure Mock
- `EmailSender`, `WhatsAppSender` — avoid real delivery in unit scope
- `RedisTemplate` — in-memory for pure logic tests

---

### Stage 4 — Integration Tests

**Java services:**
```bash
mvn verify -P integration-tests
```
Testcontainers starts: PostgreSQL 16 (TimescaleDB), Redis 7, Kafka, Elasticsearch — one set per service per CI run.

`@Container` + static lifecycle: containers start once per test class, not per test method. ~3-minute startup amortized across all tests in the class.

**Python services:**
```bash
pytest tests/integration/ -v --tb=short
```
Testcontainers Python equivalents for PG + Redis.

**Failure behavior:** Stop pipeline. Real infrastructure integration bug — must be fixed.

**Target time:** < 5 minutes per service

---

### Stage 5 — Static Analysis

**Java (SonarQube):**
```bash
mvn sonar:sonar \
  -Dsonar.projectKey=wealthpilot-{service} \
  -Dsonar.host.url=${SONAR_URL} \
  -Dsonar.login=${SONAR_TOKEN}
```

**Quality gate (blocks pipeline on failure):**
- Line coverage ≥ 80%
- No CRITICAL or BLOCKER severity issues
- No new MAJOR issues on changed code

**Java (SpotBugs):**
```bash
mvn spotbugs:check
```
Finds null pointer dereferences, thread safety issues, resource leaks.

**Python (Bandit):**
```bash
bandit -r . -ll  # medium and high severity only
```
Finds hardcoded passwords, SQL injection patterns, insecure temp file usage.

**Failure behavior:** Stop pipeline. No exceptions. SonarQube coverage gate is non-negotiable.

---

### Stage 6 — Docker Build + Push

```bash
docker build \
  -t wealthpilot/${SERVICE_NAME}:${GIT_SHA} \
  -t wealthpilot/${SERVICE_NAME}:latest \
  -f services/${SERVICE_NAME}/Dockerfile \
  services/${SERVICE_NAME}/

docker push wealthpilot/${SERVICE_NAME}:${GIT_SHA}
docker push wealthpilot/${SERVICE_NAME}:latest
```

**Dockerfile pattern (Java):**
```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S wealthpilot && adduser -S wealthpilot -G wealthpilot
WORKDIR /app
COPY target/*.jar app.jar
USER wealthpilot
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-javaagent:/otel-agent.jar", "-jar", "app.jar"]
```

**Image tagging:** `{service}:{git-sha}` — every image uniquely traceable to a commit. No mutable `latest` in production.

**Failure behavior:** Stop pipeline. Broken image not pushed.

---

### Stage 7 — Deploy to Staging

```bash
kubectl set image deployment/${SERVICE_NAME} \
  ${SERVICE_NAME}=wealthpilot/${SERVICE_NAME}:${GIT_SHA} \
  --namespace=staging

kubectl rollout status deployment/${SERVICE_NAME} \
  --namespace=staging \
  --timeout=120s
```

**Failure behavior:** Stop pipeline if rollout times out (120s) or pods crash-loop. Alert sent. Never proceeds to Stage 8 on failed rollout.

---

### Stage 8 — Integration Tests on Staging

**API tests (Newman/Postman):**
```bash
newman run infra/postman/${SERVICE_NAME}-collection.json \
  --environment infra/postman/staging-env.json \
  --reporters cli,junit \
  --reporter-junit-export newman-results.xml
```

**E2E tests (Playwright — Phase 5 only):**
```bash
npx playwright test --project=chromium --reporter=junit
```

**Failure behavior:** Stop pipeline. Production never receives broken code.

**Target:** 100% of P1 API endpoints covered in Newman collection.

---

### Stage 9 — Promote to Production

**Manual approval gate:**
```groovy
input message: "Deploy ${SERVICE_NAME}:${GIT_SHA} to production?", ok: "Approve"
```
A designated approver clicks "Approve" in Jenkins UI. No timeout — waits until approved or rejected.

**Blue-green deployment:**
```bash
# 1. Update green deployment
kubectl set image deployment/${SERVICE_NAME}-green \
  ${SERVICE_NAME}=wealthpilot/${SERVICE_NAME}:${GIT_SHA} \
  --namespace=production

# 2. Wait for green to be ready
kubectl rollout status deployment/${SERVICE_NAME}-green --timeout=120s

# 3. Switch service selector from blue to green
kubectl patch service ${SERVICE_NAME} \
  -p '{"spec":{"selector":{"slot":"green"}}}' \
  --namespace=production

# 4. Keep blue running for 30 minutes (fast rollback)
# Automated rollback check (separate monitoring job):
# If green pods restart_count > 3 within 5 min → switch selector back to blue
```

---

## Jenkinsfile Location

```
infrastructure/jenkins/
├── Jenkinsfile.auth-service
├── Jenkinsfile.portfolio-service
├── Jenkinsfile.market-data-service
├── Jenkinsfile.analytics-service
├── Jenkinsfile.notification-service
├── Jenkinsfile.news-sentiment-service
├── Jenkinsfile.ai-service
├── Jenkinsfile.supply-chain-service
├── Jenkinsfile.goals-service
├── Jenkinsfile.document-service
├── Jenkinsfile.report-service
├── Jenkinsfile.tax-service
├── Jenkinsfile.api-gateway
└── shared-lib/
    ├── vars/
    │   ├── buildJava.groovy      -- stages 2-5 for Java services
    │   ├── buildPython.groovy    -- stages 2-5 for Python services
    │   ├── dockerBuildPush.groovy
    │   └── deployToStaging.groovy
    └── src/
        └── com/wealthpilot/jenkins/
            └── PipelineUtils.groovy
```

## Docker Image Tags

| Environment | Tag | Mutable? |
|-------------|-----|---------|
| Production | `{git-sha}` | No — every deploy is a unique image |
| Staging | `{git-sha}` | No — same image promoted from staging to prod |
| Local dev | `latest` | Yes — for convenience only |

## Timing Targets (per service)

| Stage | Target |
|-------|--------|
| Checkout | < 30s |
| Build | < 2min |
| Unit Tests | < 30s |
| Integration Tests | < 5min |
| Static Analysis | < 3min |
| Docker Build + Push | < 3min |
| Deploy to Staging | < 2min |
| Staging Tests | < 5min |
| **Total (without approval)** | **< 21min** |

## Artifact Storage

- Docker images: Docker Hub (`wealthpilot/`) or AWS ECR
- Test reports: Jenkins archiveArtifacts
- SonarQube reports: SonarQube server
- Postman results: Jenkins JUnit reporter
