# API Testing

## Table of Contents

1. [Overview](#overview)
2. [Testing Pyramid for APIs](#testing-pyramid-for-apis)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [Contract Testing](#contract-testing)
6. [End-to-End Testing](#end-to-end-testing)
7. [API Mocking](#api-mocking)
8. [Performance Testing](#performance-testing)
9. [Security Testing](#security-testing)
10. [Testing in CI/CD](#testing-in-cicd)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

API testing validates that APIs meet functional, performance, and security requirements. Unlike UI testing, API tests interact directly with the service layer — making them faster to run, more stable, and more precise.

### Why API Testing Matters

| Concern | How Testing Helps |
|---|---|
| **Correctness** | Verify endpoints return expected data and status codes |
| **Compatibility** | Detect breaking changes before they reach production |
| **Reliability** | Ensure error handling works under failure conditions |
| **Performance** | Identify slow endpoints and bottlenecks early |
| **Security** | Catch authentication, authorization, and injection vulnerabilities |

---

## Testing Pyramid for APIs

```
                    ┌──────────┐
                    │   E2E    │  Few — slow, brittle, expensive
                    │  Tests   │
                 ┌──┴──────────┴──┐
                 │  Integration   │  Moderate — test real dependencies
                 │    Tests       │
              ┌──┴────────────────┴──┐
              │   Contract Tests    │  Moderate — verify API contracts
              │                     │
           ┌──┴─────────────────────┴──┐
           │      Unit Tests          │  Many — fast, isolated, cheap
           └───────────────────────────┘
```

---

## Unit Testing

Test individual handlers, validators, and business logic in isolation. Mock external dependencies.

```python
# Python example — testing a handler
def test_create_order_validates_empty_items():
    request = {"items": []}
    result = validate_create_order(request)
    assert result.is_valid is False
    assert "items" in result.errors

def test_create_order_calculates_total():
    items = [
        {"product_id": "prod_1", "quantity": 2, "price": 10.00},
        {"product_id": "prod_2", "quantity": 1, "price": 25.00},
    ]
    total = calculate_order_total(items)
    assert total == 45.00
```

---

## Integration Testing

Test the API with real (or containerized) dependencies — databases, caches, message queues.

```python
# Python example — integration test with test database
import requests

BASE_URL = "http://localhost:8080/v1"

def test_create_and_get_order(auth_headers):
    # Create
    create_response = requests.post(
        f"{BASE_URL}/orders",
        json={"items": [{"product_id": "prod_abc", "quantity": 1}]},
        headers=auth_headers,
    )
    assert create_response.status_code == 201
    order_id = create_response.json()["id"]

    # Read back
    get_response = requests.get(
        f"{BASE_URL}/orders/{order_id}",
        headers=auth_headers,
    )
    assert get_response.status_code == 200
    assert get_response.json()["id"] == order_id
    assert get_response.json()["status"] == "pending"
```

### Using Testcontainers

```python
# Spin up real dependencies in Docker for tests
from testcontainers.postgres import PostgresContainer

def test_with_real_database():
    with PostgresContainer("postgres:16") as postgres:
        connection_url = postgres.get_connection_url()
        # Run migrations and seed data
        # Start API server pointing at this database
        # Run integration tests
```

---

## Contract Testing

Contract testing verifies that an API producer and consumer agree on the API shape (request/response schemas) — without requiring both to be running at the same time.

### Consumer-Driven Contract Testing

```
Consumer A ──┐
Consumer B ──┼──→ Pact Broker ──→ Provider verification
Consumer C ──┘

1. Consumer writes a Pact (expected interactions)
2. Pact is published to a broker
3. Provider runs verification against all consumer Pacts
4. Both sides deploy independently with confidence
```

### Schema Validation

Validate responses against OpenAPI schemas in tests:

```python
# Validate response matches OpenAPI schema
from openapi_core import validate_response

def test_list_orders_matches_schema(openapi_spec):
    response = requests.get(f"{BASE_URL}/orders")
    result = validate_response(openapi_spec, response)
    assert not result.errors, f"Schema violations: {result.errors}"
```

### Breaking Change Detection

```bash
# Detect breaking changes between OpenAPI spec versions
npx @opticdev/optic diff openapi.yaml --base main

# Example output:
# BREAKING: Removed field 'legacy_id' from Order response
# BREAKING: Changed 'status' enum — removed value 'processing'
# OK: Added optional field 'tracking_url' to Order response
```

---

## End-to-End Testing

Test complete user workflows across multiple API calls and services.

```python
def test_full_order_lifecycle(auth_headers):
    # 1. Create order
    order = create_order(items=[{"product_id": "prod_1", "quantity": 1}])
    assert order["status"] == "pending"

    # 2. Confirm payment
    payment = confirm_payment(order_id=order["id"], amount=order["total"])
    assert payment["status"] == "succeeded"

    # 3. Verify order updated
    updated_order = get_order(order["id"])
    assert updated_order["status"] == "confirmed"

    # 4. Cancel order
    cancelled = cancel_order(order["id"])
    assert cancelled["status"] == "cancelled"
```

---

## API Mocking

Mocking allows testing against fake API responses without running the real service.

### Tools

| Tool | Type | Use Case |
|---|---|---|
| **WireMock** | Standalone server | Java ecosystem, flexible request matching |
| **Prism** | OpenAPI mock server | Auto-generates responses from OpenAPI spec |
| **MockServer** | Standalone | Language-agnostic, Docker-friendly |
| **MSW** | In-browser/Node | JavaScript — intercepts fetch/XHR at network level |
| **json-server** | Standalone | Quick REST API from a JSON file |

### Example: Prism Mock Server

```bash
# Generate mock server from OpenAPI spec
npx @stoplight/prism-cli mock openapi.yaml

# Mock server is now running — returns example responses
curl http://localhost:4010/v1/orders
# Returns example data based on schema definitions
```

---

## Performance Testing

### Load Testing

| Tool | Language | Strengths |
|---|---|---|
| **k6** | JavaScript | Developer-friendly, CI-integrated, Grafana ecosystem |
| **Locust** | Python | Distributed, code-based test scenarios |
| **wrk** | C | Ultra-lightweight HTTP benchmarking |
| **Gatling** | Scala | Detailed HTML reports, enterprise features |
| **Artillery** | JavaScript | YAML-based scenarios, easy setup |

### Example: k6 Load Test

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "1m", target: 50 },   // ramp up
    { duration: "3m", target: 50 },   // sustain
    { duration: "1m", target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(99)<500"],  // 99th percentile < 500ms
    http_req_failed: ["rate<0.01"],    // error rate < 1%
  },
};

export default function () {
  const res = http.get("https://api.example.com/v1/orders", {
    headers: { Authorization: "Bearer test-token" },
  });
  check(res, {
    "status is 200": (r) => r.status === 200,
    "response time < 200ms": (r) => r.timings.duration < 200,
  });
  sleep(1);
}
```

---

## Security Testing

| Test Type | Tool | What It Checks |
|---|---|---|
| **DAST** | OWASP ZAP, Burp Suite | Injection, XSS, auth bypass at runtime |
| **SAST** | Semgrep, SonarQube | Insecure code patterns in source |
| **Dependency scanning** | Snyk, Dependabot | Known vulnerabilities in libraries |
| **Fuzz testing** | RESTler, Schemathesis | Unexpected inputs generated from OpenAPI spec |

```bash
# Schemathesis — auto-generate test cases from OpenAPI spec
schemathesis run https://api.example.com/openapi.yaml \
  --checks all \
  --hypothesis-max-examples 500
```

---

## Testing in CI/CD

```yaml
# GitHub Actions example
name: API Tests
on: [push, pull_request]

jobs:
  test:
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

      - name: Unit tests
        run: pytest tests/unit/ -v

      - name: Start API server
        run: |
          python -m api_server &
          sleep 5

      - name: Integration tests
        run: pytest tests/integration/ -v

      - name: Contract validation
        run: npx @redocly/cli lint openapi.yaml

      - name: Breaking change check
        run: npx @opticdev/optic diff openapi.yaml --base origin/main
```

---

## Best Practices

1. **Test at multiple levels** — unit, integration, contract, and E2E
2. **Validate against your OpenAPI spec** — schema drift is a contract violation
3. **Detect breaking changes in CI** — block merges that remove fields or change types
4. **Use contract testing** — for multi-team or multi-consumer APIs
5. **Automate security scanning** — integrate DAST and fuzz testing in CI
6. **Run performance tests regularly** — catch regressions before production
7. **Test error paths** — 4xx and 5xx responses matter as much as 2xx
8. **Use realistic test data** — edge cases, Unicode, large payloads, empty arrays
9. **Isolate tests** — each test should be independent and repeatable
10. **Mock external dependencies** — use WireMock/Prism for third-party APIs

---

## Next Steps

| File | Topic |
|---|---|
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Comprehensive API design best practices |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Common API design mistakes |
| [07-DOCUMENTATION.md](07-DOCUMENTATION.md) | OpenAPI spec and documentation tools |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Structured learning guide |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API testing documentation |
