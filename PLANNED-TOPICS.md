# Planned Topics 🗺️

This document outlines proposed new topics to add to the Awesome Learn repository. Each topic includes a brief description, suggested file structure, and priority level. Topics are grouped by category.

> **Current topics**: Commerce Systems, Git, .NET/C#, Kubernetes, Microservices, Observability

---

## 🔴 High Priority

These topics fill critical gaps in the existing collection and complement current content.

### 1. Docker & Containers (`docker`)

Foundational container knowledge — essential prerequisite for the existing Kubernetes topic.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | What are containers, Docker architecture, OCI standards |
| `01-IMAGES-AND-DOCKERFILES.md` | Building images, Dockerfile best practices, multi-stage builds |
| `02-CONTAINER-RUNTIME.md` | Docker Engine, containerd, runc, container lifecycle |
| `03-NETWORKING.md` | Bridge, host, overlay networks, DNS, port mapping |
| `04-STORAGE-AND-VOLUMES.md` | Volumes, bind mounts, tmpfs, storage drivers |
| `05-DOCKER-COMPOSE.md` | Multi-container apps, compose files, service dependencies |
| `06-REGISTRIES.md` | Docker Hub, ECR, ACR, GCR, private registries |
| `07-SECURITY.md` | Image scanning, rootless containers, secrets management |
| `08-PERFORMANCE.md` | Resource limits, cgroups, image layer optimization |
| `09-BEST-PRACTICES.md` | Production patterns, 12-factor apps, health checks |
| `10-ANTI-PATTERNS.md` | Common container mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 2. CI/CD (`ci-cd`)

Continuous Integration and Continuous Delivery/Deployment — bridges development (Git) and operations (Kubernetes, Observability).

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | CI/CD fundamentals, pipelines, DevOps culture |
| `01-CONTINUOUS-INTEGRATION.md` | Build automation, testing in CI, artifact management |
| `02-CONTINUOUS-DELIVERY.md` | Deployment pipelines, environments, release strategies |
| `03-GITHUB-ACTIONS.md` | Workflows, actions, runners, reusable workflows |
| `04-AZURE-DEVOPS.md` | Azure Pipelines, YAML pipelines, service connections |
| `05-JENKINS.md` | Jenkinsfile, shared libraries, pipeline as code |
| `06-GITOPS.md` | ArgoCD, Flux, pull-based deployments, declarative config |
| `07-TESTING-IN-PIPELINES.md` | Unit/integration/E2E tests, test parallelization, quality gates |
| `08-SECURITY-IN-PIPELINES.md` | SAST, DAST, dependency scanning, supply chain security |
| `09-BEST-PRACTICES.md` | Pipeline patterns, trunk-based development, feature flags |
| `10-ANTI-PATTERNS.md` | Common CI/CD mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 3. API Design (`api-design`)

RESTful APIs, GraphQL, gRPC, and API governance — core to microservices communication.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | API design principles, API-first development |
| `01-REST.md` | RESTful conventions, HTTP methods, status codes, HATEOAS |
| `02-GRAPHQL.md` | Schema design, resolvers, subscriptions, federation |
| `03-GRPC.md` | Protocol Buffers, streaming, service definitions |
| `04-API-VERSIONING.md` | Versioning strategies, backward compatibility |
| `05-AUTHENTICATION-AND-AUTHORIZATION.md` | OAuth 2.0, OpenID Connect, API keys, JWT |
| `06-RATE-LIMITING-AND-THROTTLING.md` | Rate limiting patterns, quotas, backpressure |
| `07-DOCUMENTATION.md` | OpenAPI/Swagger, API documentation tools, developer portals |
| `08-TESTING.md` | Contract testing, API mocking, integration testing |
| `09-BEST-PRACTICES.md` | Naming conventions, pagination, error handling |
| `10-ANTI-PATTERNS.md` | Common API design mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 4. Databases (`databases`)

SQL, NoSQL, data modeling, and database operations — foundational for all backend services.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Database fundamentals, CAP theorem, ACID vs BASE |
| `01-RELATIONAL-DATABASES.md` | SQL, normalization, indexing, PostgreSQL, MySQL |
| `02-NOSQL-DATABASES.md` | Document, key-value, column-family, graph databases |
| `03-DATA-MODELING.md` | Schema design, entity relationships, denormalization |
| `04-QUERY-OPTIMIZATION.md` | Execution plans, indexing strategies, query tuning |
| `05-REPLICATION-AND-SHARDING.md` | Read replicas, horizontal partitioning, consistency |
| `06-MIGRATIONS.md` | Schema migrations, zero-downtime migrations, tooling |
| `07-CACHING.md` | Redis, Memcached, cache strategies, invalidation |
| `08-CONNECTION-MANAGEMENT.md` | Connection pooling, timeouts, resilience |
| `09-BEST-PRACTICES.md` | Production patterns, backup strategies, monitoring |
| `10-ANTI-PATTERNS.md` | Common database mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

---

## 🟡 Medium Priority

These topics expand the breadth of the collection into important areas.

### 5. Security (`security`)

Application security, DevSecOps, and security engineering — cross-cutting concern for all topics.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Security fundamentals, threat modeling, OWASP Top 10 |
| `01-AUTHENTICATION.md` | Password hashing, MFA, SSO, session management |
| `02-AUTHORIZATION.md` | RBAC, ABAC, policy engines, least privilege |
| `03-CRYPTOGRAPHY.md` | Encryption at rest/transit, TLS, key management |
| `04-SUPPLY-CHAIN-SECURITY.md` | Dependency scanning, SBOM, signed artifacts |
| `05-INFRASTRUCTURE-SECURITY.md` | Network policies, secrets management, zero trust |
| `06-SECURE-CODING.md` | Input validation, injection prevention, secure defaults |
| `07-COMPLIANCE.md` | SOC 2, PCI DSS, GDPR, audit logging |
| `08-INCIDENT-RESPONSE.md` | Security incident handling, forensics, post-mortems |
| `09-BEST-PRACTICES.md` | Security checklist, shift-left security, automation |
| `10-ANTI-PATTERNS.md` | Common security mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 6. Infrastructure as Code (`infrastructure-as-code`)

Terraform, Pulumi, CloudFormation — managing infrastructure declaratively.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | IaC fundamentals, declarative vs imperative, state management |
| `01-TERRAFORM.md` | HCL, providers, modules, state backends |
| `02-PULUMI.md` | General-purpose languages for IaC, stacks, automation API |
| `03-CLOUDFORMATION.md` | AWS-native IaC, nested stacks, drift detection |
| `04-STATE-MANAGEMENT.md` | Remote state, locking, import, state surgery |
| `05-MODULES-AND-REUSE.md` | Module design, versioning, registries |
| `06-TESTING.md` | Policy as code, plan validation, integration testing |
| `07-SECRETS-AND-VARIABLES.md` | Sensitive values, variable hierarchies, environments |
| `08-MULTI-CLOUD.md` | Multi-cloud patterns, abstraction layers |
| `09-BEST-PRACTICES.md` | Directory structure, naming, CI/CD integration |
| `10-ANTI-PATTERNS.md` | Common IaC mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 7. Message Queues & Event Streaming (`messaging`)

Kafka, RabbitMQ, event-driven architecture — complements Microservices and Commerce Systems.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Messaging fundamentals, queues vs streams, event-driven architecture |
| `01-APACHE-KAFKA.md` | Topics, partitions, consumer groups, Kafka Streams |
| `02-RABBITMQ.md` | Exchanges, queues, routing, reliability |
| `03-CLOUD-SERVICES.md` | AWS SQS/SNS, Azure Service Bus, Google Pub/Sub |
| `04-PATTERNS.md` | Pub/sub, fan-out, competing consumers, dead letter queues |
| `05-SCHEMA-MANAGEMENT.md` | Schema Registry, Avro, Protobuf, backward compatibility |
| `06-RELIABILITY.md` | At-least-once, exactly-once, idempotency, ordering |
| `07-MONITORING.md` | Consumer lag, throughput, alerting |
| `08-BEST-PRACTICES.md` | Partitioning strategies, retention, capacity planning |
| `09-ANTI-PATTERNS.md` | Common messaging mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 8. System Design (`system-design`)

Architecture patterns, scalability, and distributed systems — ties together multiple topics.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | System design fundamentals, trade-offs, requirements analysis |
| `01-SCALABILITY.md` | Horizontal/vertical scaling, load balancing, CDNs |
| `02-DISTRIBUTED-SYSTEMS.md` | Consensus, consistency models, distributed transactions |
| `03-CACHING-STRATEGIES.md` | Cache-aside, write-through, CDN caching, cache invalidation |
| `04-LOAD-BALANCING.md` | Algorithms, Layer 4 vs Layer 7, health checks |
| `05-DATA-PARTITIONING.md` | Sharding strategies, consistent hashing, rebalancing |
| `06-RELIABILITY.md` | Redundancy, failover, disaster recovery, chaos engineering |
| `07-COMMON-DESIGNS.md` | URL shortener, chat system, notification service, rate limiter |
| `08-BEST-PRACTICES.md` | Design process, capacity estimation, back-of-envelope math |
| `09-ANTI-PATTERNS.md` | Common system design mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

---

## 🟢 Lower Priority

These topics add more language-specific or specialized coverage.

### 9. TypeScript (`typescript`)

TypeScript language, patterns, and ecosystem — widely used frontend and backend language.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | TypeScript fundamentals, type system, compiler options |
| `01-TYPE-SYSTEM.md` | Generics, utility types, conditional types, mapped types |
| `02-PATTERNS.md` | Design patterns in TypeScript, functional patterns |
| `03-NODE-AND-BACKEND.md` | Express, NestJS, server-side TypeScript |
| `04-FRONTEND-FRAMEWORKS.md` | React, Angular, Vue with TypeScript |
| `05-TESTING.md` | Jest, Vitest, type testing, mocking |
| `06-TOOLING.md` | ESLint, Prettier, tsconfig, build tools |
| `07-BEST-PRACTICES.md` | Strict mode, type safety, error handling |
| `08-ANTI-PATTERNS.md` | Common TypeScript mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 10. Python (`python`)

Python language, patterns, and ecosystem — popular for scripting, data, and backend.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Python fundamentals, ecosystem, version management |
| `01-TYPE-HINTS.md` | Type annotations, mypy, runtime validation (Pydantic) |
| `02-WEB-FRAMEWORKS.md` | FastAPI, Django, Flask |
| `03-ASYNC-PROGRAMMING.md` | asyncio, concurrency, multiprocessing |
| `04-PACKAGING.md` | pip, poetry, virtual environments, publishing |
| `05-TESTING.md` | pytest, fixtures, mocking, property-based testing |
| `06-DATA-TOOLING.md` | pandas, NumPy, Jupyter, data pipelines |
| `07-BEST-PRACTICES.md` | PEP 8, project structure, error handling |
| `08-ANTI-PATTERNS.md` | Common Python mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 11. Cloud Computing (`cloud-computing`)

AWS, Azure, GCP fundamentals — broader cloud context for Kubernetes and IaC topics.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Cloud computing models (IaaS, PaaS, SaaS), shared responsibility |
| `01-AWS.md` | Core AWS services, IAM, VPC, EC2, S3, Lambda |
| `02-AZURE.md` | Core Azure services, Entra ID, App Service, Functions |
| `03-GCP.md` | Core GCP services, IAM, Cloud Run, Cloud Functions |
| `04-NETWORKING.md` | VPCs, subnets, peering, DNS, load balancers |
| `05-SERVERLESS.md` | Functions, event-driven, cold starts, scaling |
| `06-COST-OPTIMIZATION.md` | Reserved instances, spot, right-sizing, budgets |
| `07-MULTI-CLOUD.md` | Multi-cloud strategies, portability, abstraction |
| `08-BEST-PRACTICES.md` | Well-Architected Framework, tagging, governance |
| `09-ANTI-PATTERNS.md` | Common cloud mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

### 12. Java & Spring (`java-spring`)

Enterprise Java development — another major backend ecosystem.

| File | Description |
|------|-------------|
| `README.md` | Topic overview and navigation |
| `00-OVERVIEW.md` | Java ecosystem, JVM, Spring Boot fundamentals |
| `01-SPRING-BOOT.md` | Auto-configuration, starters, profiles, actuator |
| `02-DEPENDENCY-INJECTION.md` | IoC container, bean scopes, configuration |
| `03-WEB-AND-REST.md` | Spring MVC, WebFlux, reactive programming |
| `04-DATA-ACCESS.md` | Spring Data JPA, Hibernate, transaction management |
| `05-SECURITY.md` | Spring Security, OAuth 2.0, method security |
| `06-TESTING.md` | JUnit 5, MockMvc, Testcontainers, integration testing |
| `07-BEST-PRACTICES.md` | Project structure, error handling, logging |
| `08-ANTI-PATTERNS.md` | Common Java/Spring mistakes and how to avoid them |
| `LEARNING-PATH.md` | Structured learning guide with exercises |

---

## 📋 Implementation Checklist

For each new topic, the following steps are required (per existing repository conventions):

1. **Create topic directory** with markdown files following the `NN-TOPIC-NAME.md` naming convention
2. **Create symlink in `docs/`** pointing to the topic directory (`ln -s ../topic-name docs/topic-name`)
3. **Create symlink in `writebook/src/`** pointing to the topic directory (`ln -s ../../topic-name writebook/src/topic-name`)
4. **Update `mkdocs.yml`** — add nav entries and update `site_description`
5. **Update `writebook/src/SUMMARY.md`** — add chapter entries
6. **Update root `README.md`** — add topic to the Topics list

## 🗓️ Suggested Rollout Order

| Phase | Topics | Rationale |
|-------|--------|-----------|
| **Phase 1** | Docker, CI/CD | Foundation for existing K8s content; bridges dev and ops |
| **Phase 2** | API Design, Databases | Core backend fundamentals |
| **Phase 3** | Security, IaC | Cross-cutting operational concerns |
| **Phase 4** | Messaging, System Design | Advanced architecture topics |
| **Phase 5** | TypeScript, Python, Cloud, Java/Spring | Language-specific and broader cloud coverage |
