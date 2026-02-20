# Testing in Pipelines

A comprehensive guide to testing in CI/CD pipelines — covering the testing pyramid, unit and integration testing, end-to-end testing, contract testing, performance testing, test parallelization, quality gates, test reporting, flaky test management, and test environment provisioning.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [The Testing Pyramid in CI/CD Context](#the-testing-pyramid-in-cicd-context)
   - [Pyramid Layers](#pyramid-layers)
   - [Cost vs Confidence Tradeoff](#cost-vs-confidence-tradeoff)
   - [Where Each Test Type Runs](#where-each-test-type-runs)
3. [Unit Testing in Pipelines](#unit-testing-in-pipelines)
   - [Fast Feedback](#fast-feedback)
   - [Test Isolation](#test-isolation)
   - [Test Runners](#test-runners)
   - [Coverage Reporting](#coverage-reporting)
4. [Integration Testing in Pipelines](#integration-testing-in-pipelines)
   - [Database Testing](#database-testing)
   - [Service Dependencies](#service-dependencies)
   - [Testcontainers](#testcontainers)
   - [Docker Compose for Testing](#docker-compose-for-testing)
5. [End-to-End Testing in Pipelines](#end-to-end-testing-in-pipelines)
   - [Browser Testing](#browser-testing)
   - [Playwright and Cypress](#playwright-and-cypress)
   - [API End-to-End Testing](#api-end-to-end-testing)
   - [Flaky Test Management](#flaky-test-management)
6. [Contract Testing](#contract-testing)
   - [Consumer-Driven Contracts](#consumer-driven-contracts)
   - [Pact](#pact)
   - [Provider Verification](#provider-verification)
7. [Performance Testing in CI](#performance-testing-in-ci)
   - [Load Testing with k6](#load-testing-with-k6)
   - [Load Testing with Locust](#load-testing-with-locust)
   - [Benchmark Tests](#benchmark-tests)
   - [Regression Detection](#regression-detection)
8. [Test Parallelization](#test-parallelization)
   - [Splitting Strategies](#splitting-strategies)
   - [Parallel Execution](#parallel-execution)
   - [Test Sharding](#test-sharding)
   - [Matrix Builds](#matrix-builds)
9. [Quality Gates](#quality-gates)
   - [Coverage Thresholds](#coverage-thresholds)
   - [Mutation Testing](#mutation-testing)
   - [Code Quality Scores](#code-quality-scores)
   - [Gate Enforcement](#gate-enforcement)
10. [Test Reporting and Visibility](#test-reporting-and-visibility)
    - [JUnit XML](#junit-xml)
    - [Test Summaries](#test-summaries)
    - [Trend Analysis](#trend-analysis)
    - [Failure Notifications](#failure-notifications)
11. [Test Data Management](#test-data-management)
    - [Fixtures and Factories](#fixtures-and-factories)
    - [Database Seeding](#database-seeding)
    - [Test Isolation](#test-isolation-1)
12. [Flaky Test Detection and Quarantine](#flaky-test-detection-and-quarantine)
    - [Identifying Flaky Tests](#identifying-flaky-tests)
    - [Quarantine Strategies](#quarantine-strategies)
    - [Retry Mechanisms](#retry-mechanisms)
13. [Test Environment Provisioning](#test-environment-provisioning)
    - [Ephemeral Environments](#ephemeral-environments)
    - [Preview Environments](#preview-environments)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

Testing is the backbone of every reliable CI/CD pipeline. Without automated tests running on every commit, continuous integration is just continuous building, and continuous delivery is just continuous hoping. A well-designed test strategy catches defects early, provides fast feedback to developers, and enforces quality standards before code reaches production.

This document covers the full spectrum of testing in CI/CD pipelines: from fast unit tests that validate individual functions to end-to-end tests that verify entire user workflows, from contract tests that guard service boundaries to performance tests that catch regressions before users notice.

### Target Audience

- **DevOps Engineers** designing and maintaining CI/CD pipeline test stages
- **Developers** writing tests that run effectively in automated pipelines
- **QA Engineers** building test automation frameworks for continuous delivery
- **Platform Engineers** providing test infrastructure and tooling to development teams

### Scope

- The testing pyramid and how it maps to pipeline stages
- Unit, integration, and end-to-end testing strategies for CI/CD
- Contract testing and performance testing in pipelines
- Test parallelization, sharding, and matrix builds
- Quality gates with coverage thresholds and mutation testing
- Test reporting, flaky test management, and environment provisioning

---

## The Testing Pyramid in CI/CD Context

### Pyramid Layers

The testing pyramid is a model that describes the ideal distribution of test types in a software project. In a CI/CD context, the pyramid directly informs which tests run at which pipeline stage and how much time each stage should consume.

```
The Testing Pyramid
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ╱ ╲
                   ╱ E2E ╲              Fewer tests
                  ╱───────╲             Slower, brittle
                 ╱         ╲            High confidence
                ╱ Integration╲          Real dependencies
               ╱──────────────╲
              ╱                ╲
             ╱   Unit Tests     ╲       Many tests
            ╱────────────────────╲      Fast, stable
           ╱                      ╲     Low cost per test
          ╱________________________╲    Isolated

   Speed   ████████████████████████████  ← Unit (fastest)
           ██████████████                ← Integration
           ██████                        ← E2E (slowest)

   Count   ████████████████████████████  ← Unit (most)
           ██████████████                ← Integration
           ██████                        ← E2E (fewest)
```

| Layer | Count | Speed | Dependencies | Confidence | Pipeline Stage |
|---|---|---|---|---|---|
| **Unit** | Hundreds–thousands | Milliseconds each | None (mocked) | Low per test, high in aggregate | First (every commit) |
| **Integration** | Tens–hundreds | Seconds each | Databases, APIs, queues | Medium–high | Second (after unit) |
| **E2E** | Tens | Minutes each | Full stack deployed | Highest per test | Last (pre-deploy or post-deploy) |

### Cost vs Confidence Tradeoff

Every test type involves a tradeoff between the cost to write, run, and maintain the test and the confidence it provides.

```
Cost vs Confidence Tradeoff
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Confidence
  ▲
  │                                          ● E2E
  │                                       ╱
  │                                    ╱
  │                              ● Integration
  │                           ╱
  │                        ╱
  │                ● Unit
  │             ╱
  │          ╱
  │       ╱
  └──────────────────────────────────────────► Cost
         Write    Run     Maintain    Debug

  Unit:        Low cost,    moderate confidence
  Integration: Medium cost, high confidence
  E2E:         High cost,   highest confidence
```

| Factor | Unit Tests | Integration Tests | E2E Tests |
|---|---|---|---|
| **Write time** | Minutes | Hours | Hours–days |
| **Execution time** | Milliseconds | Seconds | Minutes |
| **Maintenance cost** | Low | Medium | High |
| **Flakiness risk** | Very low | Low–medium | High |
| **Debug difficulty** | Easy (small scope) | Medium | Hard (large scope) |
| **Infrastructure needed** | None | Databases, services | Full environment |

### Where Each Test Type Runs

```yaml
# GitHub Actions — test stages mapped to the pyramid
name: CI Pipeline
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage
    # ⏱ Target: < 2 minutes

  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration
    # ⏱ Target: < 10 minutes

  e2e-tests:
    needs: integration-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e
    # ⏱ Target: < 20 minutes
```

---

## Unit Testing in Pipelines

### Fast Feedback

Unit tests are the first line of defense in a CI/CD pipeline. They must be fast enough that developers get feedback within minutes of pushing code. If unit tests take longer than 5 minutes, developers will stop waiting for results and merge without checking.

```
Fast Feedback Loop
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer pushes commit
        │
        ▼
  ┌──────────────┐   ≤ 30s    ┌──────────────┐
  │  Checkout &  │──────────▶│  Run unit    │
  │  install     │            │  tests       │
  └──────────────┘            └──────┬───────┘
                                     │
                              ≤ 2 min│
                                     ▼
                              ┌──────────────┐
                              │  Report      │
                              │  results     │
                              └──────┬───────┘
                                     │
                     ┌───────────────┼───────────────┐
                     ▼               ▼               ▼
                ✅ Pass         ❌ Fail         📊 Coverage
                (continue)     (block merge)    (gate check)
```

**Target benchmarks for unit test stages:**

| Metric | Target | Unacceptable |
|---|---|---|
| **Total execution** | < 2 minutes | > 5 minutes |
| **Feedback to developer** | < 3 minutes | > 10 minutes |
| **Individual test** | < 100ms | > 1 second |
| **Test suite startup** | < 5 seconds | > 30 seconds |

### Test Isolation

Unit tests in pipelines must be completely isolated — no shared state between tests, no dependency on execution order, and no reliance on external services.

```yaml
# GitHub Actions — isolated unit tests with clean environment
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    env:
      NODE_ENV: test
      CI: true
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run test:unit -- --forceExit --detectOpenHandles
```

Common isolation problems in CI:

| Problem | Symptom | Solution |
|---|---|---|
| **Shared file state** | Tests pass alone, fail together | Use temp directories, clean up in `afterEach` |
| **Shared database** | Non-deterministic failures | Use transactions, rollback after each test |
| **Time-dependent tests** | Fail at midnight or on weekends | Mock `Date.now()`, use deterministic clocks |
| **Environment leakage** | Works on developer machine, fails in CI | Set explicit `NODE_ENV`, `CI` variables |
| **Port conflicts** | Bind errors in parallel runs | Use dynamic port allocation |

### Test Runners

Different languages have standard test runners with CI-specific configuration options.

| Language | Runner | CI Command | Report Format |
|---|---|---|---|
| **JavaScript/TypeScript** | Jest | `jest --ci --coverage --reporters=default --reporters=jest-junit` | JUnit XML |
| **Python** | pytest | `pytest --junitxml=report.xml --cov=src --cov-report=xml` | JUnit XML, Cobertura |
| **Java** | JUnit 5 + Maven | `mvn test -Dmaven.test.failure.ignore=false` | Surefire XML |
| **Go** | `go test` | `go test -v -race -coverprofile=coverage.out ./...` | Native, convert to JUnit |
| **C# / .NET** | xUnit/NUnit | `dotnet test --logger "trx" --collect:"XPlat Code Coverage"` | TRX, Cobertura |
| **Rust** | cargo test | `cargo test -- --format=json` | JSON, convert to JUnit |

### Coverage Reporting

Code coverage in CI is a quality signal, not a quality guarantee. Coverage measures which lines were executed during tests — it does not measure whether the tests are meaningful.

```yaml
# GitHub Actions — upload coverage to multiple services
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage --coverageReporters=lcov

      # Upload to Codecov
      - uses: codecov/codecov-action@v4
        with:
          files: coverage/lcov.info
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

      # Or upload as GitHub Actions artifact
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
```

```yaml
# Azure Pipelines — coverage with Cobertura
steps:
  - task: Npm@1
    inputs:
      command: custom
      customCommand: run test:unit -- --coverage --coverageReporters=cobertura
  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: coverage/cobertura-coverage.xml
      failIfCoverageEmpty: true
```

---

## Integration Testing in Pipelines

### Database Testing

Integration tests that interact with real databases are essential for verifying queries, migrations, and data access logic. CI environments provide ephemeral databases via service containers.

```yaml
# GitHub Actions — PostgreSQL integration tests
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run db:migrate
      - run: npm run test:integration
```

```yaml
# Azure Pipelines — MySQL service container
resources:
  containers:
    - container: mysql
      image: mysql:8.0
      ports:
        - 3306:3306
      env:
        MYSQL_ROOT_PASSWORD: testpass
        MYSQL_DATABASE: testdb

steps:
  - script: npm run test:integration
    env:
      DATABASE_URL: mysql://root:testpass@localhost:3306/testdb
```

### Service Dependencies

Integration tests often need multiple services running simultaneously. Managing these dependencies in CI requires explicit health checks and startup ordering.

```
Service Dependency Graph
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  PostgreSQL  │     │   Redis     │     │  RabbitMQ   │
  │  :5432       │     │  :6379      │     │  :5672      │
  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
         │                   │                   │
         │  health-check     │  health-check     │  health-check
         │  pg_isready       │  redis-cli ping   │  rabbitmq-diagnostics
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │  Run migrations│
                    │  Seed data     │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │  Integration   │
                    │  Tests         │
                    └────────────────┘
```

### Testcontainers

Testcontainers provides programmatic control over Docker containers from within test code. Instead of configuring service containers in YAML, tests define their own infrastructure.

```yaml
# GitHub Actions — Testcontainers with Docker-in-Docker
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: temurin
      # Testcontainers uses the Docker socket on the runner
      - run: mvn test -pl integration-tests
        env:
          TESTCONTAINERS_RYUK_DISABLED: true
```

| Approach | Pros | Cons |
|---|---|---|
| **CI service containers** | Simple YAML config, native support | Static config, no programmatic control |
| **Testcontainers** | Programmatic, test-scoped lifecycle | Requires Docker socket, slower startup |
| **Docker Compose** | Multi-service orchestration | Separate process, harder to synchronize |
| **Embedded/in-memory** | Fastest, no Docker needed | Not production-representative |

### Docker Compose for Testing

Docker Compose can orchestrate multi-service integration test environments in CI.

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build:
      context: .
      target: test
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb
    command: npm run test:integration
```

```yaml
# GitHub Actions — integration tests with Docker Compose
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from app
      - run: docker compose -f docker-compose.test.yml down -v
        if: always()
```

---

## End-to-End Testing in Pipelines

### Browser Testing

End-to-end browser tests verify complete user workflows through the actual UI. Running these in CI requires headless browsers, proper timeouts, and artifact collection for debugging failures.

```
E2E Test Flow in CI
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  Build app   │───▶│  Start app   │───▶│  Run E2E     │
  │              │    │  server      │    │  tests       │
  └──────────────┘    └──────────────┘    └──────┬───────┘
                                                 │
                                    ┌────────────┼────────────┐
                                    ▼            ▼            ▼
                               Screenshots   Videos     Trace files
                               on failure    on failure  on failure
                                    │            │            │
                                    └────────────┼────────────┘
                                                 ▼
                                          Upload as CI
                                          artifacts
```

### Playwright and Cypress

Playwright and Cypress are the two most popular E2E testing frameworks for browser automation in CI/CD pipelines.

```yaml
# GitHub Actions — Playwright E2E tests
jobs:
  e2e-playwright:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npx playwright test
        env:
          CI: true
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

```yaml
# GitHub Actions — Cypress E2E tests
jobs:
  e2e-cypress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cypress-io/github-action@v6
        with:
          build: npm run build
          start: npm run start
          wait-on: http://localhost:3000
          wait-on-timeout: 120
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots/
```

| Feature | Playwright | Cypress |
|---|---|---|
| **Browser support** | Chromium, Firefox, WebKit | Chromium, Firefox, WebKit (experimental) |
| **Language support** | JS/TS, Python, Java, .NET | JS/TS only |
| **Parallelization** | Built-in sharding | Cypress Cloud (paid) or third-party |
| **Auto-wait** | Yes | Yes |
| **Network interception** | Yes | Yes |
| **Trace viewer** | Built-in (HTML) | Cypress Cloud |
| **Multi-tab/multi-origin** | Yes | Limited |
| **CI speed** | Fast (install per browser) | Slower (full install) |
| **Debugging** | Trace viewer, VS Code extension | Time-travel debugging |

### API End-to-End Testing

API E2E tests verify that the full request/response cycle works correctly — from HTTP request through middleware, business logic, database, and back.

```yaml
# GitHub Actions — API E2E tests with a running server
jobs:
  api-e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run db:migrate
      - run: npm run build
      - run: npm run start &
      - run: npx wait-on http://localhost:3000/health --timeout 30000
      - run: npm run test:api-e2e
```

### Flaky Test Management

Flaky tests — tests that pass and fail non-deterministically — are the single biggest threat to CI/CD pipeline reliability. A pipeline with flaky tests trains developers to ignore failures.

| Flakiness Source | Example | Mitigation |
|---|---|---|
| **Timing/race conditions** | Element not yet visible | Use auto-wait, explicit waits, avoid `sleep` |
| **Shared state** | Previous test left data in DB | Isolate test data, clean up in `beforeEach` |
| **Network variability** | External API timeout | Mock external services, increase timeouts |
| **Resource contention** | CI runner under load | Request dedicated runners, reduce parallelism |
| **Non-deterministic order** | Tests depend on execution order | Randomize test order, fix hidden dependencies |
| **Timezone/locale** | Date formatting differences | Set explicit `TZ=UTC` in CI environment |

---

## Contract Testing

### Consumer-Driven Contracts

Contract testing verifies that services can communicate correctly without requiring them to run simultaneously. In consumer-driven contract testing, the consumer of an API defines the contract, and the provider verifies it.

```
Consumer-Driven Contract Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  CONSUMER PIPELINE                    PROVIDER PIPELINE
  ┌──────────────────┐                ┌──────────────────┐
  │  Consumer tests  │                │  Provider tests  │
  │  generate        │                │  verify          │
  │  contract (pact) │                │  contract (pact) │
  └────────┬─────────┘                └────────┬─────────┘
           │                                   │
           ▼                                   ▼
  ┌──────────────────┐                ┌──────────────────┐
  │  Publish pact to │                │  Fetch pact from │
  │  Pact Broker     │──────────────▶│  Pact Broker     │
  └──────────────────┘                └──────────────────┘
           │                                   │
           ▼                                   ▼
  ┌──────────────────┐                ┌──────────────────┐
  │  can-i-deploy?   │◀──────────────│  Publish         │
  │  (check provider │                │  verification    │
  │   verification)  │                │  results         │
  └────────┬─────────┘                └──────────────────┘
           │
           ▼
      Deploy consumer
```

### Pact

Pact is the most widely adopted contract testing framework. It supports HTTP and message-based interactions across multiple languages.

```yaml
# GitHub Actions — consumer contract tests with Pact
jobs:
  consumer-contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:pact
      - run: npx pact-broker publish pacts/
          --consumer-app-version=${{ github.sha }}
          --branch=${{ github.ref_name }}
          --broker-base-url=${{ secrets.PACT_BROKER_URL }}
          --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
```

```yaml
# GitHub Actions — provider verification with Pact
jobs:
  provider-contract-verification:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:pact:provider
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          PACT_PROVIDER_VERSION: ${{ github.sha }}
          PACT_PROVIDER_BRANCH: ${{ github.ref_name }}
```

### Provider Verification

Provider verification runs during the provider's CI pipeline. The provider fetches all consumer contracts from the Pact Broker and replays each interaction against its real API.

| Contract Testing Concept | Description |
|---|---|
| **Consumer** | The service that makes API calls (e.g., frontend, downstream service) |
| **Provider** | The service that serves the API (e.g., backend, upstream service) |
| **Pact** | A JSON file containing expected interactions (request/response pairs) |
| **Pact Broker** | Central server that stores pacts and verification results |
| **can-i-deploy** | CLI tool that checks if a version is safe to deploy based on verification status |
| **Webhook** | Triggers provider verification when a new consumer pact is published |

---

## Performance Testing in CI

### Load Testing with k6

k6 is a developer-centric load testing tool that integrates well with CI/CD pipelines. Tests are written in JavaScript, and results can be exported in multiple formats.

```yaml
# GitHub Actions — k6 load test as a quality gate
jobs:
  performance-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
      - uses: grafana/run-k6-action@v1
        with:
          path: tests/performance/load-test.js
        env:
          K6_THRESHOLDS_HTTP_REQ_DURATION: "p(95)<500"
          K6_THRESHOLDS_HTTP_REQ_FAILED: "rate<0.01"
```

```javascript
// tests/performance/load-test.js — k6 test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // ramp up
    { duration: '1m',  target: 20 },   // steady state
    { duration: '30s', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95th percentile < 500ms
    http_req_failed: ['rate<0.01'],     // < 1% errors
  },
};

export default function () {
  const res = http.get('http://localhost:3000/api/health');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

### Load Testing with Locust

Locust is a Python-based load testing framework. It uses a code-first approach and is well-suited for teams that prefer Python.

```yaml
# GitHub Actions — Locust load test
jobs:
  performance-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install locust
      - run: |
          locust -f tests/performance/locustfile.py \
            --headless \
            --users 50 \
            --spawn-rate 10 \
            --run-time 2m \
            --host http://localhost:3000 \
            --csv results/perf \
            --exit-code-on-error 1
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: performance-results
          path: results/
```

### Benchmark Tests

Benchmark tests measure the performance of specific code paths and detect regressions between commits.

```yaml
# GitHub Actions — Go benchmark with regression detection
jobs:
  benchmarks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - run: go test -bench=. -benchmem -count=5 ./... | tee bench-current.txt
      - uses: actions/cache/restore@v4
        with:
          path: bench-baseline.txt
          key: benchmark-${{ github.base_ref }}
      - name: Compare benchmarks
        run: |
          if [ -f bench-baseline.txt ]; then
            go install golang.org/x/perf/cmd/benchstat@latest
            benchstat bench-baseline.txt bench-current.txt
          fi
      - uses: actions/cache/save@v4
        if: github.ref == 'refs/heads/main'
        with:
          path: bench-current.txt
          key: benchmark-main
```

### Regression Detection

Performance regression detection compares current results against a baseline and fails the pipeline if performance degrades beyond acceptable thresholds.

| Tool | Language | Regression Detection | CI Integration |
|---|---|---|---|
| **k6** | JavaScript | Thresholds in script | Native exit codes |
| **Locust** | Python | Custom assertions | Exit code on error |
| **benchstat** | Go | Statistical comparison | Manual threshold |
| **JMH** | Java | Baseline comparison | Plugin-based |
| **hyperfine** | Any (CLI) | Statistical comparison | Exit codes |
| **pytest-benchmark** | Python | Comparison with `--benchmark-compare` | pytest exit code |

---

## Test Parallelization

### Splitting Strategies

Running tests sequentially in a single job does not scale. As test suites grow, parallelization becomes essential to maintain fast feedback.

```
Sequential vs Parallel Execution
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SEQUENTIAL (single runner)
  ┌─────────────────────────────────────────────────────┐
  │ Test1 │ Test2 │ Test3 │ Test4 │ Test5 │ Test6 │ ... │
  └─────────────────────────────────────────────────────┘
  Total time: sum of all tests = 30 min
                                 ════════

  PARALLEL (3 runners)
  ┌───────────────────┐
  │ Test1 │ Test3 │ T5│  Runner 1
  ├───────────────────┤
  │ Test2 │ Test4 │ T6│  Runner 2
  ├───────────────────┤
  │ Test7 │ Test8 │ T9│  Runner 3
  └───────────────────┘
  Total time: max of any runner = 10 min
                                  ════════
```

| Splitting Strategy | Description | Best For |
|---|---|---|
| **File-based** | Distribute test files across runners | Simple, predictable suites |
| **Time-based** | Use historical timing data to balance | Unevenly sized tests |
| **Test-count** | Equal number of tests per runner | Uniform test duration |
| **Directory-based** | Each runner gets a directory/module | Modular projects |
| **Custom tags** | Tests tagged with shard ID | Fine-grained control |

### Parallel Execution

```yaml
# GitHub Actions — parallel test execution with matrix
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx jest --shard=${{ matrix.shard }}/4

  merge-results:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - run: echo "All shards passed"
```

```yaml
# GitHub Actions — Playwright parallel sharding
jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test --shard=${{ matrix.shard }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: blob-report-${{ strategy.job-index }}
          path: blob-report/

  merge-e2e-reports:
    needs: e2e-tests
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports/
      - run: npx playwright merge-reports --reporter=html all-blob-reports/
      - uses: actions/upload-artifact@v4
        with:
          name: playwright-merged-report
          path: playwright-report/
```

### Test Sharding

Test sharding divides a test suite into roughly equal parts and runs each part on a separate CI runner. The key challenge is maintaining even distribution so no single shard becomes a bottleneck.

```yaml
# GitHub Actions — pytest sharding with pytest-split
jobs:
  tests:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        group: [1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: |
          pytest --splits 3 --group ${{ matrix.group }} \
            --splitting-algorithm least_duration \
            --junitxml=results-${{ matrix.group }}.xml
      - uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.group }}
          path: results-${{ matrix.group }}.xml
```

### Matrix Builds

Matrix builds run the same test suite across multiple configurations — different OS versions, language versions, or dependency versions.

```yaml
# GitHub Actions — matrix build for cross-platform testing
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [18, 20, 22]
        exclude:
          - os: macos-latest
            node-version: 18
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      - run: npm ci
      - run: npm test
```

| Matrix Dimension | Purpose | Example Values |
|---|---|---|
| **OS** | Cross-platform compatibility | `ubuntu-latest`, `macos-latest`, `windows-latest` |
| **Language version** | Version compatibility | Node 18, 20, 22 / Python 3.10, 3.11, 3.12 |
| **Database version** | Database compatibility | PostgreSQL 14, 15, 16 |
| **Dependency set** | Dependency range testing | `locked` (lockfile), `latest` (newest) |

---

## Quality Gates

### Coverage Thresholds

Coverage thresholds enforce minimum code coverage percentages. When coverage drops below the threshold, the pipeline fails.

```yaml
# GitHub Actions — enforce coverage thresholds
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage --coverageThreshold='{
          "global": {
            "branches": 80,
            "functions": 80,
            "lines": 85,
            "statements": 85
          }
        }'
```

```yaml
# Azure Pipelines — coverage gate in build validation
steps:
  - script: dotnet test --collect:"XPlat Code Coverage"
  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: "**/coverage.cobertura.xml"
  # Build validation policy enforces minimum coverage
```

| Coverage Metric | Description | Recommended Threshold |
|---|---|---|
| **Line coverage** | Percentage of lines executed | 80–90% |
| **Branch coverage** | Percentage of branches taken | 75–85% |
| **Function coverage** | Percentage of functions called | 80–90% |
| **Statement coverage** | Percentage of statements executed | 80–90% |
| **Diff coverage** | Coverage of changed lines only | 90–100% |

### Mutation Testing

Mutation testing evaluates the quality of your tests by introducing small changes (mutations) to your code and checking whether your tests detect them. If a test suite has high coverage but low mutation score, the tests are executing code without truly verifying behavior.

```yaml
# GitHub Actions — mutation testing with Stryker (JavaScript)
jobs:
  mutation-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx stryker run
      - uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: reports/mutation/
```

| Mutation Type | What It Changes | Example |
|---|---|---|
| **Arithmetic** | `+` → `-`, `*` → `/` | `a + b` becomes `a - b` |
| **Conditional** | `>` → `>=`, `==` → `!=` | `if (x > 0)` becomes `if (x >= 0)` |
| **Boolean** | `true` → `false`, `&&` → `\|\|` | `if (a && b)` becomes `if (a \|\| b)` |
| **Return value** | Change return values | `return true` becomes `return false` |
| **Remove call** | Delete method calls | `list.sort()` removed |
| **String** | Replace string literals | `"hello"` becomes `""` |

### Code Quality Scores

Code quality tools measure maintainability, complexity, duplication, and code smells as quality gate criteria.

```yaml
# GitHub Actions — SonarQube quality gate
jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: SonarSource/sonarqube-scan-action@v5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      - uses: SonarSource/sonarqube-quality-gate-action@v1
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

| Quality Metric | Tool | Gate Condition |
|---|---|---|
| **Code coverage** | Codecov, SonarQube | No decrease from baseline |
| **Duplication** | SonarQube, jscpd | < 3% duplicated lines |
| **Cyclomatic complexity** | ESLint, SonarQube | No function > 15 complexity |
| **Technical debt ratio** | SonarQube | < 5% debt ratio |
| **Security hotspots** | SonarQube, Snyk | Zero unreviewed hotspots |
| **Code smells** | SonarQube | No new code smells on changed code |

### Gate Enforcement

Quality gates must be enforced, not advisory. This means configuring branch protection rules to require gate checks to pass before merging.

```
Quality Gate Enforcement
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pull Request opened
        │
        ▼
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ Unit tests   │   │ Integration  │   │ Quality      │
  │ ✅ Pass      │   │ tests        │   │ gate         │
  │ Coverage: 87%│   │ ✅ Pass      │   │ ✅ Pass      │
  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │ All checks   │
                    │ passed?      │
                    └──────┬───────┘
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
            ✅ Yes              ❌ No
            Merge allowed       Merge blocked
```

---

## Test Reporting and Visibility

### JUnit XML

JUnit XML is the de facto standard for test result reporting across CI systems. Most test runners can output JUnit XML, and most CI platforms can parse it for rich test result displays.

```xml
<!-- Example JUnit XML output -->
<?xml version="1.0" encoding="UTF-8"?>
<testsuites tests="42" failures="1" errors="0" time="12.345">
  <testsuite name="UserService" tests="10" failures="1" time="3.456">
    <testcase name="should create user" classname="UserService" time="0.123"/>
    <testcase name="should validate email" classname="UserService" time="0.045">
      <failure message="Expected valid email">
        AssertionError: expected 'invalid' to match /^[^@]+@[^@]+$/
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

```yaml
# GitHub Actions — publish JUnit test results
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --reporters=jest-junit
        env:
          JEST_JUNIT_OUTPUT_DIR: test-results
      - uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Unit Tests
          path: test-results/*.xml
          reporter: jest-junit
          fail-on-error: true
```

```yaml
# Azure Pipelines — publish test results natively
steps:
  - script: pytest --junitxml=test-results/results.xml
  - task: PublishTestResults@2
    condition: always()
    inputs:
      testResultsFormat: JUnit
      testResultsFiles: "test-results/*.xml"
      mergeTestResults: true
      failTaskOnFailedTests: true
```

### Test Summaries

GitHub Actions supports job summaries that appear directly on the workflow run page.

```yaml
# GitHub Actions — write test summary to $GITHUB_STEP_SUMMARY
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: |
          npm test -- --json --outputFile=results.json 2>&1 || true
      - name: Generate summary
        if: always()
        run: |
          echo "## Test Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          TOTAL=$(jq '.numTotalTests' results.json)
          PASSED=$(jq '.numPassedTests' results.json)
          FAILED=$(jq '.numFailedTests' results.json)
          echo "| Metric | Count |" >> $GITHUB_STEP_SUMMARY
          echo "|---|---|" >> $GITHUB_STEP_SUMMARY
          echo "| ✅ Passed | $PASSED |" >> $GITHUB_STEP_SUMMARY
          echo "| ❌ Failed | $FAILED |" >> $GITHUB_STEP_SUMMARY
          echo "| 📊 Total | $TOTAL |" >> $GITHUB_STEP_SUMMARY
```

### Trend Analysis

Tracking test metrics over time reveals patterns that point-in-time results miss — growing test duration, increasing flakiness, or declining coverage.

| Metric to Track | What It Reveals | Action Trigger |
|---|---|---|
| **Test count** | Suite growth rate | Plan parallelization when count doubles |
| **Test duration** | Speed degradation | Optimize when total exceeds 10 minutes |
| **Flaky test rate** | Reliability decline | Quarantine when flakiness exceeds 5% |
| **Coverage trend** | Quality trajectory | Alert on 3 consecutive coverage drops |
| **Failure rate** | Code stability | Review process when failure rate exceeds 10% |

### Failure Notifications

Failed tests should notify the right people through the right channels. Notification fatigue is as dangerous as no notifications.

```yaml
# GitHub Actions — Slack notification on test failure
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "❌ Tests failed on ${{ github.ref_name }} by ${{ github.actor }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Test Failure* in `${{ github.repository }}`\nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                  }
                }
              ]
            }
```

---

## Test Data Management

### Fixtures and Factories

Test data must be predictable, isolated, and easy to create. Fixtures provide static test data while factories generate dynamic test data.

| Approach | Description | Best For |
|---|---|---|
| **Static fixtures** | JSON/YAML files with known data | Read-only tests, snapshot testing |
| **Factories** | Programmatic data generation with defaults | Tests needing unique data each run |
| **Builders** | Fluent API for constructing test objects | Complex object graphs |
| **Fakers** | Realistic random data generation | Load testing, stress testing |

```yaml
# GitHub Actions — seed test database before integration tests
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run db:migrate
      - run: npm run db:seed:test
      - run: npm run test:integration
```

### Database Seeding

Database seeding creates a known baseline state for integration tests. The seed should be idempotent and fast.

```
Database Seeding Strategy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Option A: Seed once, clean per test (fast)
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Migrate      │───▶│ Seed base    │───▶│ Run tests    │
  │              │    │ data         │    │ (transaction │
  │              │    │              │    │  rollback)   │
  └──────────────┘    └──────────────┘    └──────────────┘

  Option B: Clean slate per test (safe)
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Migrate      │───▶│ Truncate all │───▶│ Seed + run   │
  │              │    │ tables       │    │ single test  │
  │              │    │              │    │              │
  └──────────────┘    └──────┬───────┘    └──────────────┘
                             │ repeat for each test
                             └──────────────────────────▶
```

| Strategy | Speed | Isolation | Complexity |
|---|---|---|---|
| **Transaction rollback** | Fast | High | Low |
| **Truncate + reseed** | Slow | Highest | Medium |
| **Database per test** | Slowest | Highest | High |
| **Schema snapshots** | Fast | High | Medium |

### Test Isolation

Each test must create and clean up its own data. When tests share data, failures cascade unpredictably.

```yaml
# GitHub Actions — parallel integration tests with isolated databases
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      # Each shard gets its own database
      - run: |
          PGPASSWORD=testpass psql -h localhost -U postgres \
            -c "CREATE DATABASE testdb_shard_${{ matrix.shard }};"
      - run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb_shard_${{ matrix.shard }}
      - run: npm run test:integration -- --shard=${{ matrix.shard }}/3
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb_shard_${{ matrix.shard }}
```

---

## Flaky Test Detection and Quarantine

### Identifying Flaky Tests

A flaky test is one that produces different results (pass or fail) on the same code without any changes. Detecting flaky tests requires tracking test results over time.

```
Flaky Test Detection Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Test run history (same code, same branch):

  Run 1:  ✅ ✅ ✅ ❌ ✅ ✅ ✅ ✅ ✅ ✅
  Run 2:  ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅
  Run 3:  ✅ ✅ ✅ ❌ ✅ ✅ ✅ ✅ ✅ ✅
  Run 4:  ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅
                   ▲
                   │
            This test is flaky
            (fails ~50% of runs with no code change)

  Detection method:
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Collect test │───▶│ Compare      │───▶│ Flag tests   │
  │ results from │    │ results      │    │ that flip    │
  │ N runs       │    │ across runs  │    │ pass ↔ fail  │
  └──────────────┘    └──────────────┘    └──────────────┘
```

Common detection approaches:

| Method | How It Works | Automation Level |
|---|---|---|
| **Re-run on failure** | Re-run failed tests; if they pass on retry, flag as flaky | Semi-automatic |
| **Multi-run analysis** | Run suite N times and compare results | Fully automatic |
| **Historical tracking** | Track pass/fail per test over builds | Fully automatic |
| **Deflake mode** | Run each test 3x, only report consistent failures | Built into some frameworks |

### Quarantine Strategies

Once identified, flaky tests should be quarantined — separated from the main test suite so they do not block pipelines while they are being fixed.

```yaml
# GitHub Actions — quarantined test suite runs separately
jobs:
  stable-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --testPathIgnorePatterns="quarantine"
    # This job blocks merges

  quarantined-tests:
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --testPathPattern="quarantine"
    # This job does NOT block merges
```

| Quarantine Strategy | Approach | Risk |
|---|---|---|
| **Separate test suite** | Move flaky tests to `quarantine/` directory | Tests may be forgotten |
| **Tag-based skip** | Tag with `@flaky`, filter in CI | Tests may never be untagged |
| **Auto-retry** | Retry failed tests N times | Hides real failures |
| **Parallel monitoring** | Run quarantine suite as non-blocking job | Extra CI cost |
| **Time-boxed quarantine** | Auto-delete quarantine tag after N days | Forces resolution |

### Retry Mechanisms

Test retries can be a pragmatic solution for occasional flakiness, but they must be used carefully to avoid masking real failures.

```yaml
# GitHub Actions — retry flaky E2E tests with Playwright
jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      # Playwright retries are configured in playwright.config.ts:
      #   retries: process.env.CI ? 2 : 0
      - run: npx playwright test
```

```yaml
# GitHub Actions — retry the entire job on failure
jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 1
      matrix:
        attempt: [1, 2]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:e2e
```

---

## Test Environment Provisioning

### Ephemeral Environments

Ephemeral environments are short-lived, on-demand environments created for each pipeline run or pull request. They provide isolated, production-like infrastructure for testing.

```
Ephemeral Environment Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  PR opened / commit pushed
        │
        ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Provision    │───▶│ Deploy app   │───▶│ Run E2E      │
  │ infrastructure│   │ to ephemeral │    │ tests against│
  │ (Terraform,  │    │ environment  │    │ ephemeral    │
  │  K8s namespace)   │              │    │ env          │
  └──────────────┘    └──────────────┘    └──────┬───────┘
                                                 │
                                      ┌──────────┴──────────┐
                                      ▼                     ▼
                                 Tests pass            Tests fail
                                      │                     │
                                      ▼                     ▼
                              ┌──────────────┐    ┌──────────────┐
                              │ Tear down    │    │ Keep env for │
                              │ environment  │    │ debugging    │
                              └──────────────┘    │ (TTL: 2 hrs)│
                                                  └──────────────┘
  PR closed / merged
        │
        ▼
  ┌──────────────┐
  │ Tear down    │
  │ environment  │
  └──────────────┘
```

```yaml
# GitHub Actions — Kubernetes ephemeral namespace
jobs:
  deploy-ephemeral:
    runs-on: ubuntu-latest
    env:
      NAMESPACE: pr-${{ github.event.pull_request.number }}
    steps:
      - uses: actions/checkout@v4
      - uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      - run: kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
      - run: helm upgrade --install app ./charts/app
          --namespace $NAMESPACE
          --set image.tag=${{ github.sha }}
          --wait --timeout 5m
      - run: npm run test:e2e
        env:
          BASE_URL: http://app.${{ env.NAMESPACE }}.svc.cluster.local

  cleanup-ephemeral:
    needs: deploy-ephemeral
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      - run: kubectl delete namespace pr-${{ github.event.pull_request.number }} --ignore-not-found
```

### Preview Environments

Preview environments extend ephemeral environments with a public URL, allowing team members to manually review and test pull request changes before merging.

```yaml
# GitHub Actions — preview environment with Vercel
jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        id: deploy
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}

  e2e-on-preview:
    needs: deploy-preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test
        env:
          BASE_URL: ${{ needs.deploy-preview.outputs.url }}
```

| Environment Type | Lifetime | Use Case | Infrastructure |
|---|---|---|---|
| **Ephemeral** | Minutes (test run) | Automated testing | Kubernetes namespace, Docker Compose |
| **Preview** | Lifetime of PR | Manual review + automated tests | Vercel, Netlify, Kubernetes |
| **Staging** | Permanent | Pre-production validation | Dedicated cluster/server |
| **Production** | Permanent | Live traffic | Production infrastructure |

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Foundational CI/CD concepts, principles, and pipeline design |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build, test, and integration automation |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment strategies, release management, and delivery pipelines |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflow authoring, actions marketplace, and CI/CD with GitHub |
| [04-AZURE-DEVOPS.md](04-AZURE-DEVOPS.md) | Azure DevOps | Azure Pipelines, Boards, Repos, and Artifacts |
| [05-JENKINS.md](05-JENKINS.md) | Jenkins | Jenkins pipelines, distributed builds, and plugin ecosystem |
| [06-GITOPS.md](06-GITOPS.md) | GitOps | ArgoCD, Flux, and Git-based deployment workflows |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Testing in Pipelines documentation |
