# API Documentation

## Table of Contents

1. [Overview](#overview)
2. [OpenAPI Specification](#openapi-specification)
3. [Documentation Tools](#documentation-tools)
4. [Developer Portals](#developer-portals)
5. [Writing Good API Documentation](#writing-good-api-documentation)
6. [Code Examples](#code-examples)
7. [Documentation as Code](#documentation-as-code)
8. [Best Practices](#best-practices)
9. [Next Steps](#next-steps)
10. [Version History](#version-history)

---

## Overview

API documentation is the primary interface between an API and its consumers. Well-documented APIs reduce support burden, accelerate onboarding, and increase adoption. Poorly documented APIs — even technically excellent ones — go unused.

### Why Documentation Matters

| Outcome | Impact |
|---|---|
| **Faster integration** | Developers self-serve instead of asking questions |
| **Fewer support tickets** | Clear docs reduce confusion and misuse |
| **Higher adoption** | Developers choose well-documented APIs over undocumented alternatives |
| **Fewer breaking changes** | Documented contracts create shared understanding |
| **Better developer experience** | Interactive docs let consumers try before they code |

---

## OpenAPI Specification

The OpenAPI Specification (OAS, formerly Swagger) is the industry standard for describing RESTful APIs. It is a machine-readable format (YAML or JSON) that describes endpoints, parameters, request/response bodies, authentication, and more.

### Basic Structure

```yaml
openapi: 3.1.0
info:
  title: Order API
  version: 1.0.0
  description: API for managing customer orders
  contact:
    name: API Team
    email: api-team@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /orders:
    get:
      summary: List orders
      operationId: listOrders
      tags:
        - Orders
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: per_page
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: A list of orders
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Order'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      summary: Create an order
      operationId: createOrder
      tags:
        - Orders
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  schemas:
    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [pending, confirmed, shipped, delivered, cancelled]
        total:
          type: number
          format: decimal
        created_at:
          type: string
          format: date-time

    CreateOrderRequest:
      type: object
      required:
        - items
      properties:
        items:
          type: array
          items:
            type: object
            required: [product_id, quantity]
            properties:
              product_id:
                type: string
              quantity:
                type: integer
                minimum: 1

    Pagination:
      type: object
      properties:
        page:
          type: integer
        per_page:
          type: integer
        total:
          type: integer

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

### OpenAPI Versions

| Version | Status | Key Features |
|---|---|---|
| **2.0** (Swagger) | Legacy | Original spec, widely supported |
| **3.0** | Current | Components, multiple servers, improved schema |
| **3.1** | Latest | Full JSON Schema support, webhooks |

---

## Documentation Tools

### Rendering Tools

| Tool | Type | Strengths |
|---|---|---|
| **Swagger UI** | Open source | Interactive try-it-out, widely recognized |
| **Redoc** | Open source | Clean three-panel layout, excellent for large APIs |
| **Stoplight Elements** | Open source | Modern UI, embedded components |
| **RapiDoc** | Open source | Web component, highly customizable |

### Design-First Tools

| Tool | Type | Strengths |
|---|---|---|
| **Stoplight Studio** | Commercial/Free | Visual OpenAPI editor, Git integration |
| **SwaggerHub** | Commercial | Collaborative editing, hosted docs, mocking |
| **Postman** | Commercial/Free | Collections, testing, documentation in one tool |
| **Insomnia** | Open source | API client with design features |

### Code Generation

```bash
# Generate server stubs
openapi-generator generate -i openapi.yaml -g spring -o ./server

# Generate client SDKs
openapi-generator generate -i openapi.yaml -g typescript-axios -o ./client

# Validate an OpenAPI spec
npx @redocly/cli lint openapi.yaml
```

---

## Developer Portals

A developer portal is a self-service website where API consumers discover, learn, and integrate with your APIs. It goes beyond raw API reference documentation.

### Key Components

| Component | Purpose |
|---|---|
| **Getting started guide** | Walk new users from zero to first API call |
| **API reference** | Auto-generated from OpenAPI spec |
| **Authentication guide** | Step-by-step instructions for obtaining and using credentials |
| **Code examples** | Copy-paste snippets in multiple languages |
| **Changelog** | Track what changed between versions |
| **Status page** | Current API health and incident history |
| **SDKs and libraries** | Official client libraries for popular languages |
| **Rate limit documentation** | Published limits per tier |
| **Support/Community** | Forums, Slack channels, or issue trackers |

### Popular Developer Portal Platforms

| Platform | Type | Notes |
|---|---|---|
| **Readme.com** | SaaS | Hosted developer hubs with API explorer |
| **Backstage** | Open source | Spotify's developer portal framework |
| **Stoplight** | SaaS | Design-first with hosted documentation |
| **Redocly** | SaaS/Self-hosted | From the makers of Redoc |
| **Custom** | Self-hosted | Built with static site generators + Swagger UI |

---

## Writing Good API Documentation

### Structure Every Endpoint

Each endpoint should document:

1. **Summary** — one-line description
2. **Description** — detailed explanation of behavior
3. **Parameters** — path, query, header parameters with types and constraints
4. **Request body** — schema with examples
5. **Response codes** — every possible status code with body schema
6. **Authentication** — required scopes or permissions
7. **Examples** — request and response pairs
8. **Error cases** — common errors and how to handle them

### Example: Well-Documented Endpoint

```markdown
## Create Order

Creates a new order for the authenticated user.

**POST** `/v1/orders`

### Authentication
Requires `Bearer` token with `orders:write` scope.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | array | Yes | List of order items |
| items[].product_id | string | Yes | Product identifier |
| items[].quantity | integer | Yes | Quantity (min: 1) |
| shipping_address_id | string | No | Saved address ID |

### Example Request

    curl -X POST https://api.example.com/v1/orders \
      -H "Authorization: Bearer eyJhbG..." \
      -H "Content-Type: application/json" \
      -d '{
        "items": [
          {"product_id": "prod_abc", "quantity": 2}
        ]
      }'

### Responses

**201 Created** — Order successfully created
**400 Bad Request** — Invalid input (see error body)
**401 Unauthorized** — Missing or invalid token
**422 Unprocessable Entity** — Product unavailable
```

---

## Code Examples

Provide examples in the languages your consumers use most. At minimum, include cURL and one popular language.

### Multi-Language Examples

```bash
# cURL
curl -X GET "https://api.example.com/v1/orders?page=1&per_page=10" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

```python
# Python (requests)
import requests

response = requests.get(
    "https://api.example.com/v1/orders",
    headers={"Authorization": "Bearer YOUR_TOKEN"},
    params={"page": 1, "per_page": 10},
)
orders = response.json()
```

```javascript
// JavaScript (fetch)
const response = await fetch(
  "https://api.example.com/v1/orders?page=1&per_page=10",
  {
    headers: { Authorization: "Bearer YOUR_TOKEN" },
  }
);
const orders = await response.json();
```

---

## Documentation as Code

Treat API documentation like source code — version it, review it, test it, and automate its publication.

### Workflow

```
1. Write OpenAPI spec alongside code
2. Validate spec in CI (lint + breaking change detection)
3. Auto-generate API reference from spec
4. Deploy docs site on merge to main
```

### CI/CD Integration

```yaml
# GitHub Actions example
name: API Docs
on:
  push:
    paths:
      - 'openapi/**'
      - 'docs/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint OpenAPI spec
        run: npx @redocly/cli lint openapi/openapi.yaml
      - name: Check for breaking changes
        run: npx @opticdev/optic diff openapi/openapi.yaml --base main

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build docs
        run: npx @redocly/cli build-docs openapi/openapi.yaml -o docs/index.html
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs
```

---

## Best Practices

1. **Start with OpenAPI** — write the spec before writing code (design-first)
2. **Keep docs next to code** — version the OpenAPI spec in the same repository as the API
3. **Validate in CI** — lint specs and detect breaking changes automatically
4. **Provide runnable examples** — use cURL, Postman collections, or interactive try-it-out
5. **Include error examples** — document what happens when things go wrong, not just the happy path
6. **Show authentication setup** — step-by-step from "get a key" to "make your first call"
7. **Maintain a changelog** — clearly communicate what changed between versions
8. **Use consistent terminology** — define a glossary and stick to it across all endpoints
9. **Test your docs** — have someone unfamiliar with the API try to integrate using only the docs
10. **Keep docs up to date** — stale documentation is worse than no documentation

---

## Next Steps

| File | Topic |
|---|---|
| [08-TESTING.md](08-TESTING.md) | Contract testing, API mocking, integration testing |
| [01-REST.md](01-REST.md) | RESTful API conventions |
| [04-API-VERSIONING.md](04-API-VERSIONING.md) | Versioning strategies and deprecation |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Comprehensive API design best practices |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API documentation guide |
