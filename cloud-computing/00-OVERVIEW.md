# Cloud Computing Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is Cloud Computing](#what-is-cloud-computing)
3. [Cloud Service Models](#cloud-service-models)
4. [Shared Responsibility Model](#shared-responsibility-model)
5. [Cloud Deployment Models](#cloud-deployment-models)
6. [Cloud Service Categories](#cloud-service-categories)
7. [Cloud-Native Concepts](#cloud-native-concepts)
8. [Choosing a Cloud Provider](#choosing-a-cloud-provider)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to cloud computing fundamentals. It covers the core concepts of cloud service models, deployment models, the shared responsibility model, major service categories across leading providers, cloud-native application design, and guidance for choosing a cloud provider.

### Target Audience

- **Developers** building applications that run on cloud infrastructure and consume cloud-managed services
- **Architects** designing cloud-native systems, selecting service models, and defining deployment strategies
- **DevOps Engineers** provisioning cloud resources, managing infrastructure as code, and operating cloud workloads
- **Engineering Managers** evaluating cloud providers, understanding cost models, and planning cloud adoption strategies

### Scope

- What cloud computing is and why it transformed the technology industry
- The five essential characteristics of cloud computing as defined by NIST
- Cloud service models: IaaS, PaaS, SaaS, and FaaS/serverless
- The shared responsibility model and how security duties are divided between provider and customer
- Cloud deployment models: public, private, hybrid, and multi-cloud
- Core service categories: compute, storage, database, and networking
- Cloud-native concepts: 12-factor apps, microservices, containers, orchestration, CI/CD, and infrastructure as code
- Criteria for selecting a cloud provider and a comparison of AWS, Azure, and GCP

---

## What is Cloud Computing

**Cloud computing** is the delivery of computing resources — servers, storage, databases, networking, software, analytics, and intelligence — over the internet ("the cloud") on a pay-as-you-go basis. Instead of buying and maintaining physical data centers and servers, organizations rent access to technology resources from a cloud provider and pay only for what they use.

### A Brief History

Cloud computing evolved through several phases:

- **1960s** — John McCarthy proposed that computing could be organized as a public utility, similar to electricity or water.
- **2002** — Amazon Web Services (AWS) launched its first suite of cloud services.
- **2006** — AWS launched EC2 and S3, marking the beginning of modern cloud computing.
- **2010** — Microsoft launched Windows Azure (now Microsoft Azure).
- **2011** — NIST published its formal definition of cloud computing (SP 800-145).
- **2014** — Kubernetes was open-sourced by Google. GCP expanded its services.
- **2017–present** — Serverless computing, multi-cloud strategies, and edge computing became mainstream.

### Key Characteristics (NIST Definition)

The National Institute of Standards and Technology (NIST) defines five essential characteristics that distinguish cloud computing from traditional IT hosting:

**1. On-Demand Self-Service**

A consumer can provision computing capabilities — such as server time, storage, or network resources — automatically without requiring human interaction with the service provider. Resources are available through a web portal, CLI, or API at any time.

**2. Broad Network Access**

Capabilities are available over the network and accessed through standard mechanisms. Cloud services are reachable from any device with an internet connection — laptops, phones, tablets, and other workstations.

**3. Resource Pooling**

The provider's computing resources are pooled to serve multiple consumers using a multi-tenant model. Physical and virtual resources are dynamically assigned and reassigned according to demand. The customer generally has no control or knowledge over the exact location of the provided resources but may be able to specify location at a higher level of abstraction (e.g., country, region, or availability zone).

**4. Rapid Elasticity**

Capabilities can be elastically provisioned and released, in some cases automatically, to scale outward and inward commensurate with demand. To the consumer, the capabilities available for provisioning often appear to be unlimited and can be appropriated in any quantity at any time.

**5. Measured Service**

Cloud systems automatically control and optimize resource use by leveraging a metering capability appropriate to the type of service (e.g., storage, processing, bandwidth, or active user accounts). Resource usage can be monitored, controlled, and reported, providing transparency for both the provider and consumer.

### Traditional IT vs Cloud Computing

| Aspect | Traditional IT | Cloud Computing |
|--------|---------------|-----------------|
| Capital expenditure | Large upfront investment in hardware | No upfront cost; pay-as-you-go |
| Provisioning time | Weeks to months for hardware procurement | Minutes to seconds via API or portal |
| Scaling | Manual capacity planning; risk of over/under-provisioning | Elastic scaling on demand |
| Maintenance | Customer manages hardware, patching, cooling, power | Provider manages underlying infrastructure |
| Geographic reach | Limited to owned/leased data center locations | Global presence with regions and availability zones |
| Cost model | CapEx-heavy; depreciation over 3–5 years | OpEx-based; pay only for consumption |
| Disaster recovery | Expensive secondary site required | Built-in redundancy and cross-region replication |
| Innovation speed | Slow; constrained by procurement cycles | Fast; new services available immediately |
| Security staffing | Customer hires and manages all security staff | Provider employs dedicated security teams at scale |

---

## Cloud Service Models

Cloud services are delivered in three primary models, each representing a different level of abstraction and management responsibility. A fourth model — Functions as a Service (FaaS) — has emerged as a specialized variant.

```
Cloud Service Models — Abstraction Levels
──────────────────────────────────────────

  More Customer Control                    More Provider Control
  ◄──────────────────────────────────────────────────────────────►

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │     IaaS     │  │     PaaS     │  │     FaaS     │  │     SaaS     │
  │              │  │              │  │              │  │              │
  │  You manage: │  │  You manage: │  │  You manage: │  │  You manage: │
  │  OS, runtime │  │  App code,   │  │  Function    │  │  Data,       │
  │  middleware, │  │  data        │  │  code, data  │  │  config      │
  │  app, data   │  │              │  │              │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
       VMs             App hosting       Serverless         Software
       Bare metal      Containers        Event-driven       End-user apps
```

### Infrastructure as a Service (IaaS)

IaaS provides the fundamental building blocks of cloud IT. It gives access to networking features, virtual or dedicated computers, and data storage space. IaaS delivers the highest level of flexibility and management control over IT resources and is the closest to traditional on-premises environments.

**What you get:**

- Virtual machines with configurable CPU, memory, and storage
- Virtual networks, subnets, firewalls, and load balancers
- Full control over the operating system, middleware, and runtime

**What the provider manages:**

- Physical servers, storage hardware, and networking equipment
- Data center facilities — power, cooling, physical security
- Hypervisor and virtualization layer

**When to use IaaS:**

- You need full control over the operating system and software stack
- You are migrating existing on-premises workloads to the cloud ("lift and shift")
- You need to run custom or legacy software that cannot run on managed platforms

**Examples:**

| Provider | IaaS Compute | IaaS Storage | IaaS Networking |
|----------|-------------|-------------|-----------------|
| AWS | EC2 | EBS, S3 | VPC, ELB |
| Azure | Virtual Machines | Managed Disks, Blob Storage | VNet, Load Balancer |
| GCP | Compute Engine | Persistent Disk, Cloud Storage | VPC, Cloud Load Balancing |

### Platform as a Service (PaaS)

PaaS removes the need for managing underlying infrastructure (hardware and operating systems) and allows the customer to focus on deploying and managing applications. The provider handles the OS, runtime, middleware, and servers.

**What you get:**

- A managed platform to deploy application code
- Automatic OS patching and runtime updates
- Built-in scaling, load balancing, and health monitoring

**What the provider manages:**

- Everything in IaaS, plus:
- Operating system maintenance and patching
- Runtime environment (e.g., JVM, .NET CLR, Python interpreter)

**When to use PaaS:**

- You want to focus on writing code, not managing infrastructure
- You need fast iteration and deployment cycles
- You want built-in scaling without manual capacity management

**Examples:**

| Provider | PaaS Compute | PaaS Database | PaaS Containers |
|----------|-------------|--------------|-----------------|
| AWS | Elastic Beanstalk, App Runner | RDS, Aurora | ECS, Fargate |
| Azure | App Service, Spring Apps | SQL Database, Cosmos DB | Container Apps |
| GCP | App Engine, Cloud Run | Cloud SQL, Firestore | Cloud Run |

### Software as a Service (SaaS)

SaaS provides a complete product that is run and managed by the service provider. The customer simply uses the software — typically through a web browser — without worrying about how the service is maintained, how the infrastructure is managed, or how the software is deployed.

**What you get:**

- Fully functional software accessible via a browser or API
- Automatic updates and new feature releases
- Multi-device access

**What the provider manages:**

- Everything: infrastructure, platform, application code, and operations
- Data backups, availability, and disaster recovery
- Security patching, feature updates, and bug fixes

**When to use SaaS:**

- You need standard business functionality (email, CRM, collaboration)
- You want zero operational overhead for the application
- You do not need deep customization of the application internals

**Examples:** Gmail, Microsoft 365, Salesforce, Slack, Jira, Datadog, Snowflake

### Functions as a Service (FaaS) / Serverless

FaaS is an event-driven compute model where the customer writes small, single-purpose functions that are executed in response to events. The cloud provider manages all infrastructure, automatically scales from zero to thousands of concurrent executions, and charges only for the compute time consumed.

**What you get:**

- Execute code in response to events (HTTP requests, queue messages, file uploads, schedules)
- Automatic scaling from zero to high concurrency
- Sub-second billing (per invocation and execution duration)

**What the provider manages:**

- All infrastructure, OS, runtime, and execution environment
- Scaling, load balancing, and fault tolerance
- Cold start optimization and execution isolation

**When to use FaaS:**

- Event-driven workloads: API backends, file processing, webhooks
- Intermittent or unpredictable traffic patterns
- Cost optimization for workloads with idle periods

**Examples:**

| Provider | FaaS Service |
|----------|-------------|
| AWS | Lambda |
| Azure | Azure Functions |
| GCP | Cloud Functions |

### Service Model Comparison — Customer vs Provider Responsibilities

| Layer | On-Premises | IaaS | PaaS | FaaS | SaaS |
|-------|------------|------|------|------|------|
| **Data** | Customer | Customer | Customer | Customer | Shared |
| **Application** | Customer | Customer | Customer | Customer | Provider |
| **Runtime** | Customer | Customer | Provider | Provider | Provider |
| **Middleware** | Customer | Customer | Provider | Provider | Provider |
| **Operating System** | Customer | Customer | Provider | Provider | Provider |
| **Virtualization** | Customer | Provider | Provider | Provider | Provider |
| **Servers** | Customer | Provider | Provider | Provider | Provider |
| **Storage** | Customer | Provider | Provider | Provider | Provider |
| **Networking** | Customer | Provider | Provider | Provider | Provider |
| **Physical Security** | Customer | Provider | Provider | Provider | Provider |

---

## Shared Responsibility Model

The shared responsibility model defines which security and compliance tasks are handled by the cloud provider and which are the customer's responsibility. Understanding this model is critical — misunderstanding it is one of the most common causes of cloud security incidents.

### The Core Concept

In every cloud deployment, security responsibilities are **shared** between the provider and the customer. The provider is responsible for the **security of the cloud** — the physical infrastructure, hypervisor, network backbone, and data center operations. The customer is responsible for **security in the cloud** — the data, applications, identity management, and configurations they place on top of the provider's infrastructure.

The exact split depends on the service model being used.

### Responsibility Matrix

| Responsibility | IaaS | PaaS | SaaS |
|---------------|------|------|------|
| Data classification and accountability | Customer | Customer | Customer |
| Identity and access management | Customer | Customer | Shared |
| Application-level security | Customer | Customer | Provider |
| Network controls (firewalls, ACLs) | Customer | Shared | Provider |
| Operating system patching | Customer | Provider | Provider |
| Host infrastructure security | Provider | Provider | Provider |
| Physical data center security | Provider | Provider | Provider |
| Environmental controls (power, cooling) | Provider | Provider | Provider |

### Shared Responsibility by Service Model

```
Shared Responsibility Model — IaaS vs PaaS vs SaaS
────────────────────────────────────────────────────

                  IaaS              PaaS              SaaS
             ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
             │    Data      │  │    Data      │  │    Data      │
             │  (Customer)  │  │  (Customer)  │  │  (Shared)    │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │ Application  │  │ Application  │  │ Application  │
             │  (Customer)  │  │  (Customer)  │  │  (Provider)  │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │  Runtime     │  │  Runtime     │  │  Runtime     │
             │  (Customer)  │  │  (Provider)  │  │  (Provider)  │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │    OS        │  │    OS        │  │    OS        │
             │  (Customer)  │  │  (Provider)  │  │  (Provider)  │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │  Network     │  │  Network     │  │  Network     │
             │  (Customer)  │  │  (Shared)    │  │  (Provider)  │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │ Hypervisor   │  │ Hypervisor   │  │ Hypervisor   │
             │  (Provider)  │  │  (Provider)  │  │  (Provider)  │
             ├─────────────┤  ├─────────────┤  ├─────────────┤
             │  Physical    │  │  Physical    │  │  Physical    │
             │  (Provider)  │  │  (Provider)  │  │  (Provider)  │
             └─────────────┘  └─────────────┘  └─────────────┘

  Legend:
    Customer  = You are responsible
    Provider  = Cloud provider is responsible
    Shared    = Both share responsibility
```

### Common Misunderstandings

- **"The cloud provider secures everything."** — False. The provider secures the infrastructure, but you must secure your data, applications, and configurations.
- **"I moved to SaaS so I have no security responsibilities."** — False. You still own data classification, access management, and compliance obligations.
- **"The provider is responsible if my data is breached."** — It depends. If the breach resulted from a misconfigured customer resource (e.g., a public S3 bucket), the customer is responsible.
- **"Compliance is the provider's job."** — Shared. The provider maintains compliance certifications for the infrastructure, but you must ensure your workloads comply with regulations.

### Best Practices

- **Know the boundary** — For each service you use, understand what the provider manages and what you manage.
- **Encrypt data** — Encrypt data at rest and in transit. Use customer-managed keys for sensitive workloads.
- **Manage identities carefully** — Use least-privilege access, enforce multi-factor authentication, and rotate credentials.
- **Monitor continuously** — Enable logging and monitoring for all cloud resources to detect anomalies.
- **Automate compliance** — Use policy-as-code tools to enforce security guardrails across your environment.

---

## Cloud Deployment Models

Cloud deployment models describe where cloud infrastructure is located, who manages it, and who can access it. Each model balances trade-offs between cost, control, compliance, and complexity.

### Public Cloud

In a public cloud, the infrastructure is owned and operated by a third-party cloud provider and shared among multiple organizations (tenants). Resources are provisioned on demand over the internet.

**Characteristics:**

- Multi-tenant infrastructure shared across customers
- Pay-as-you-go pricing with no upfront capital expenditure
- Global availability with regions and availability zones
- Managed entirely by the provider

**When to use:**

- You want to minimize operational overhead and capital costs
- You need rapid provisioning and elastic scaling
- You are building new applications without strict data residency constraints

**Examples:** AWS, Microsoft Azure, Google Cloud Platform, Oracle Cloud, IBM Cloud

### Private Cloud

A private cloud is cloud infrastructure dedicated to a single organization. It can be hosted on-premises in the organization's data center or by a third-party provider, but it is not shared with other organizations.

**Characteristics:**

- Single-tenant infrastructure dedicated to one organization
- Greater control over hardware, network, and data location
- Higher upfront investment and ongoing operational costs
- May use cloud technologies like OpenStack, VMware, or Azure Stack

**When to use:**

- Regulatory or compliance requirements mandate data residency and isolation
- You need complete control over the hardware and network stack
- Your organization has the expertise and budget to manage infrastructure

**Examples:** VMware vSphere, OpenStack, Azure Stack Hub, AWS Outposts (on-premises)

### Hybrid Cloud

A hybrid cloud combines public and private cloud environments, allowing data and applications to move between them. This model enables organizations to keep sensitive workloads on private infrastructure while leveraging public cloud for scalable, less-sensitive workloads.

**Characteristics:**

- Workloads run across both public and private environments
- Requires networking (VPN, dedicated interconnects) between environments
- Enables "cloud bursting" — scaling to public cloud during demand spikes

**When to use:**

- You have existing on-premises investments you cannot immediately migrate
- Some workloads have regulatory constraints requiring private infrastructure
- You are executing a phased cloud migration strategy

**Examples:** Azure Arc, AWS Outposts + AWS Cloud, Google Anthos, VMware Cloud on AWS

### Multi-Cloud

A multi-cloud strategy uses services from two or more public cloud providers. Unlike hybrid cloud (which mixes public and private), multi-cloud is about distributing workloads across multiple public clouds.

**Characteristics:**

- Workloads distributed across two or more public cloud providers
- Avoids vendor lock-in by spreading dependencies
- Leverages best-of-breed services from each provider
- Increases operational complexity — different APIs, tools, billing, and IAM systems

**When to use:**

- You want to avoid dependence on a single provider
- You want to leverage best-in-class services from different providers
- Regulatory requirements mandate geographic distribution across providers

**Examples:** Running Kubernetes workloads on both AWS EKS and Azure AKS, using GCP BigQuery for analytics while running production services on AWS

### Deployment Model Comparison

| Aspect | Public Cloud | Private Cloud | Hybrid Cloud | Multi-Cloud |
|--------|-------------|--------------|-------------|-------------|
| **Cost** | Low upfront; pay-as-you-go | High upfront; ongoing maintenance | Medium; mixed cost model | Variable; potential duplication |
| **Control** | Limited; provider-managed | Full; customer-managed | Mixed; depends on workload placement | Mixed; varies by provider |
| **Compliance** | Shared responsibility; limited data control | Full control; easier to meet strict regulations | Flexible; sensitive data stays private | Complex; must meet requirements per provider |
| **Complexity** | Low; single provider | Medium; requires infrastructure expertise | High; integration between environments | Highest; multiple providers to manage |
| **Scalability** | Excellent; near-unlimited | Limited by owned capacity | Good; public cloud absorbs bursts | Excellent; multiple providers available |
| **Vendor lock-in** | Risk of lock-in to one provider | No cloud vendor lock-in | Moderate; partial lock-in | Low; workloads spread across providers |

---

## Cloud Service Categories

Cloud providers offer hundreds of services, but they organize into a few core categories. Understanding these categories — and how they map across AWS, Azure, and GCP — helps you navigate any cloud platform.

### Compute

Compute services provide the processing power to run applications. The three primary compute models are virtual machines, containers, and serverless functions.

**Virtual Machines (VMs)**

VMs emulate physical servers and provide a complete operating system environment. They are the most flexible compute option and support any workload that can run on a traditional server.

- Full control over the OS, runtime, and software stack
- Available in a wide range of sizes (CPU, memory, GPU, storage-optimized)
- Support both Linux and Windows

**Containers**

Containers package an application and its dependencies into a lightweight, portable unit. They share the host OS kernel but run in isolated user spaces, making them faster to start and more resource-efficient than VMs.

- Consistent environment from development to production
- Faster startup than VMs (seconds vs minutes)
- Requires a container orchestrator (Kubernetes, ECS) for production workloads

**Serverless Functions**

Serverless functions execute code in response to events without provisioning or managing servers. The provider handles all infrastructure. You pay only for execution time.

- Event-driven: triggered by HTTP requests, queue messages, file uploads, schedules
- Automatic scaling from zero to thousands of concurrent executions
- Sub-second billing granularity
- Cold start latency on first invocation

**Compute Services Across Providers:**

| Compute Type | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| Virtual Machines | EC2 | Virtual Machines | Compute Engine |
| Managed Containers | ECS, Fargate | Container Instances, Container Apps | Cloud Run, GKE Autopilot |
| Kubernetes | EKS | AKS | GKE |
| Serverless Functions | Lambda | Azure Functions | Cloud Functions |
| PaaS Compute | Elastic Beanstalk, App Runner | App Service | App Engine |

### Storage

Cloud storage services provide durable, scalable data storage. The three primary storage types serve different access patterns and use cases.

**Block Storage**

Block storage provides raw storage volumes that attach to VMs. Each volume behaves like a physical hard drive — it can be formatted with a file system and used for any purpose.

- Low latency, high IOPS for databases and transactional workloads
- Volumes are attached to a single VM at a time (typically)
- Data persists independently of the VM lifecycle

**Object Storage**

Object storage stores data as objects in a flat namespace (no directory hierarchy). Each object consists of the data itself, metadata, and a unique identifier. Object storage is optimized for storing large amounts of unstructured data.

- Virtually unlimited capacity
- Accessible via HTTP/HTTPS APIs
- Built-in redundancy and durability (99.999999999% — "eleven 9s")
- Ideal for backups, media files, static website hosting, and data lakes

**File Storage**

File storage provides a shared file system accessible by multiple VMs simultaneously over standard protocols (NFS, SMB/CIFS). It behaves like a traditional network file share.

- Shared access across multiple compute instances
- POSIX-compliant file system semantics
- Suitable for legacy applications, content management, and home directories

**Storage Services Across Providers:**

| Storage Type | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| Block Storage | EBS | Managed Disks | Persistent Disk |
| Object Storage | S3 | Blob Storage | Cloud Storage |
| File Storage | EFS, FSx | Azure Files | Filestore |
| Archive Storage | S3 Glacier | Archive Storage | Archive Storage |

### Database

Cloud database services provide managed database engines that handle provisioning, patching, backups, and scaling. The three primary categories address different data models and access patterns.

**Relational Databases (SQL)**

Relational databases store data in structured tables with rows and columns. They enforce schemas, support ACID transactions, and use SQL for queries. Managed relational databases handle backups, patching, replication, and failover.

- Strong consistency and ACID transactions
- Schema enforcement ensures data integrity
- Suitable for transactional workloads, ERP, CRM, and financial systems

**NoSQL Databases**

NoSQL databases use flexible data models — key-value, document, wide-column, or graph. They are designed for specific access patterns and can scale horizontally across many nodes.

- Flexible schema — no predefined structure required
- Horizontal scaling across distributed nodes
- Optimized for specific patterns: key-value lookups, document queries, graph traversals

**Data Warehouses**

Data warehouses are optimized for analytical queries over large datasets. They use columnar storage and massively parallel processing (MPP) to execute complex aggregations and joins across terabytes or petabytes of data.

- Columnar storage for efficient analytical queries
- Massively parallel processing for fast aggregations
- Suitable for business intelligence, reporting, and data analytics

**Database Services Across Providers:**

| Database Type | AWS | Azure | GCP |
|--------------|-----|-------|-----|
| Relational (managed) | RDS, Aurora | SQL Database, SQL Managed Instance | Cloud SQL, AlloyDB |
| NoSQL (document) | DynamoDB | Cosmos DB | Firestore |
| NoSQL (wide-column) | Keyspaces | Cosmos DB (Cassandra API) | Bigtable |
| In-Memory Cache | ElastiCache | Cache for Redis | Memorystore |
| Data Warehouse | Redshift | Synapse Analytics | BigQuery |

### Networking

Cloud networking services provide the virtual network infrastructure that connects compute, storage, and database resources — and connects them to the internet and on-premises networks.

**Virtual Private Cloud (VPC) / Virtual Network**

A VPC is an isolated virtual network that you define in the cloud. It is the foundational networking construct — all cloud resources (VMs, databases, load balancers) are deployed into a VPC.

- Isolated network with customer-defined IP address ranges (CIDR blocks)
- Subnets segment the VPC into public and private tiers
- Route tables control traffic flow between subnets and to the internet
- Security groups and network ACLs filter inbound and outbound traffic

**DNS**

Cloud DNS services translate domain names to IP addresses. They provide global, highly available, and low-latency name resolution.

- Authoritative DNS hosting for your domains
- Health checks and routing policies (latency-based, geolocation, failover)
- Integration with other cloud services for automatic DNS record management

**Content Delivery Network (CDN)**

A CDN caches content at edge locations around the world, reducing latency for end users by serving content from the nearest edge point.

- Static content caching (images, CSS, JavaScript, videos)
- DDoS protection at the edge
- TLS termination at edge locations

**Load Balancer**

Load balancers distribute incoming traffic across multiple compute instances to ensure high availability and reliability.

- Layer 4 (TCP/UDP) and Layer 7 (HTTP/HTTPS) load balancing
- Health checks automatically remove unhealthy instances
- SSL/TLS termination offloads encryption from backend instances

**Networking Services Across Providers:**

| Networking Service | AWS | Azure | GCP |
|-------------------|-----|-------|-----|
| Virtual Network | VPC | VNet | VPC |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| CDN | CloudFront | Azure CDN / Front Door | Cloud CDN |
| Load Balancer | ALB, NLB, ELB | Load Balancer, Application Gateway | Cloud Load Balancing |
| VPN Gateway | VPN Gateway | VPN Gateway | Cloud VPN |
| Direct Connect | Direct Connect | ExpressRoute | Cloud Interconnect |

---

## Cloud-Native Concepts

Cloud-native is an approach to building and running applications that fully exploits the advantages of cloud computing. Rather than simply running existing applications in the cloud ("lift and shift"), cloud-native applications are designed from the ground up for cloud environments.

### The Twelve-Factor App

The Twelve-Factor App is a methodology for building software-as-a-service applications. It was published by Heroku co-founder Adam Wiggins and has become a foundational guide for cloud-native application design.

| Factor | Principle | Description |
|--------|-----------|-------------|
| I. Codebase | One codebase, many deploys | A single codebase tracked in version control, deployed to multiple environments |
| II. Dependencies | Explicitly declare and isolate | Never rely on system-wide packages; declare all dependencies in a manifest |
| III. Config | Store config in the environment | Configuration that varies between deploys (database URLs, credentials) is stored in environment variables |
| IV. Backing Services | Treat as attached resources | Databases, caches, queues are accessed via URLs; swappable without code changes |
| V. Build, Release, Run | Strictly separate stages | Build creates an artifact, release combines it with config, run executes it |
| VI. Processes | Execute as stateless processes | App processes are stateless and share-nothing; persistent data is stored in backing services |
| VII. Port Binding | Export services via port binding | The app is self-contained and exports HTTP by binding to a port |
| VIII. Concurrency | Scale out via the process model | Scale by running multiple instances of stateless processes |
| IX. Disposability | Maximize robustness with fast startup and graceful shutdown | Processes start quickly and shut down gracefully on SIGTERM |
| X. Dev/Prod Parity | Keep development, staging, and production as similar as possible | Minimize gaps between environments to reduce deployment surprises |
| XI. Logs | Treat logs as event streams | Apps write logs to stdout; the execution environment captures and routes them |
| XII. Admin Processes | Run admin/management tasks as one-off processes | Database migrations and scripts run as one-off processes in the same environment |

### Microservices

Microservices architecture decomposes an application into small, independently deployable services, each responsible for a specific business capability. Each service:

- Owns its own data store (database per service)
- Communicates with other services via APIs or messaging
- Can be deployed, scaled, and updated independently
- Is built and maintained by a small, autonomous team

Microservices are a natural fit for cloud-native environments because they align with cloud scaling models, container packaging, and CI/CD pipelines.

### Containers

Containers package an application and all its dependencies into a single, portable image. Unlike VMs, containers share the host OS kernel and are lightweight — they start in seconds and use minimal resources.

```
Containers vs Virtual Machines
──────────────────────────────

  Virtual Machines                 Containers
  ┌───────────┐ ┌───────────┐    ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │   App A   │ │   App B   │    │App A│ │App B│ │App C│ │App D│
  ├───────────┤ ├───────────┤    ├─────┤ ├─────┤ ├─────┤ ├─────┤
  │  Bins/Libs│ │  Bins/Libs│    │Bins/│ │Bins/│ │Bins/│ │Bins/│
  ├───────────┤ ├───────────┤    │Libs │ │Libs │ │Libs │ │Libs │
  │  Guest OS │ │  Guest OS │    └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
  └─────┬─────┘ └─────┬─────┘       │      │      │      │
        │              │          ┌──┴──────┴──────┴──────┴──┐
  ┌─────┴──────────────┴─────┐   │      Container Runtime    │
  │        Hypervisor         │   ├──────────────────────────┤
  ├──────────────────────────┤   │         Host OS           │
  │         Host OS           │   ├──────────────────────────┤
  ├──────────────────────────┤   │        Hardware           │
  │         Hardware          │   └──────────────────────────┘
  └──────────────────────────┘

  - VMs: Each VM runs a full guest OS (GBs of overhead)
  - Containers: Share host OS kernel (MBs of overhead)
  - Containers start in seconds; VMs take minutes
  - Containers have higher density per host
```

**Key tools:** Docker (container runtime), container registries (Docker Hub, Amazon ECR, Azure Container Registry, Google Artifact Registry)

### Container Orchestration

Container orchestration platforms automate the deployment, scaling, networking, and management of containerized applications. Kubernetes is the dominant orchestration platform.

**What orchestration provides:**

- Automatic scheduling of containers across a cluster of nodes
- Self-healing: restarts failed containers, replaces unresponsive nodes
- Horizontal scaling based on CPU, memory, or custom metrics
- Service discovery and load balancing between containers
- Rolling updates and rollbacks with zero downtime

**Managed Kubernetes services:** AWS EKS, Azure AKS, Google GKE

### CI/CD (Continuous Integration / Continuous Delivery)

CI/CD automates the process of building, testing, and deploying applications. It is a core practice for cloud-native development.

- **Continuous Integration (CI)** — Developers merge code frequently. Each merge triggers an automated build and test pipeline. Failures are detected early.
- **Continuous Delivery (CD)** — Every code change that passes CI is automatically prepared for release to production. Deployment may require manual approval.
- **Continuous Deployment** — Extends CD by automatically deploying every change that passes all pipeline stages to production with no manual intervention.

**Common tools:** GitHub Actions, GitLab CI, Jenkins, Azure DevOps Pipelines, AWS CodePipeline, Google Cloud Build

### Infrastructure as Code (IaC)

Infrastructure as Code (IaC) manages and provisions cloud resources through machine-readable definition files rather than manual processes or interactive tools. IaC treats infrastructure the same way developers treat application code — it is version-controlled, peer-reviewed, tested, and deployed through automated pipelines.

**Benefits:**

- **Repeatability** — The same template produces identical environments every time
- **Version control** — Changes to infrastructure are tracked in Git
- **Speed** — Provision entire environments in minutes
- **Drift detection** — Detect when actual infrastructure diverges from the defined state

**Common tools:**

| Tool | Type | Provider Support |
|------|------|-----------------|
| Terraform | Declarative, multi-cloud | AWS, Azure, GCP, and 1000+ providers |
| AWS CloudFormation | Declarative, AWS-only | AWS |
| Azure Bicep / ARM | Declarative, Azure-only | Azure |
| Google Cloud Deployment Manager | Declarative, GCP-only | GCP |
| Pulumi | Imperative, multi-cloud | AWS, Azure, GCP, Kubernetes |
| Ansible | Procedural, multi-cloud | Any SSH/API-accessible system |

---

## Choosing a Cloud Provider

Selecting a cloud provider is a strategic decision that affects architecture, cost, team skills, and vendor relationships. There is no universally "best" provider — the right choice depends on your organization's needs, workloads, and constraints.

### Selection Criteria

**1. Service Breadth and Depth**

Does the provider offer the specific services you need? While all major providers cover compute, storage, and databases, they differ in specialized areas like machine learning, IoT, data analytics, and industry-specific solutions.

**2. Pricing Model**

Cloud pricing is complex. Evaluate:

- On-demand vs reserved vs spot/preemptible pricing
- Data transfer costs (egress fees can be significant)
- Free tier availability for experimentation
- Pricing calculators and cost management tools
- Committed use discounts and enterprise agreements

**3. Compliance and Certifications**

Does the provider meet your industry's regulatory requirements? Consider:

- Compliance certifications (SOC 2, HIPAA, PCI DSS, FedRAMP, ISO 27001)
- Data residency and sovereignty options
- Audit logging and governance capabilities
- Government cloud regions (AWS GovCloud, Azure Government)

**4. Ecosystem and Tooling**

Evaluate the surrounding ecosystem:

- Marketplace of third-party integrations and solutions
- IaC support (Terraform providers, native IaC tools)
- CI/CD integration and observability tooling
- Documentation quality and community resources

**5. Team Skills and Experience**

Your team's existing expertise significantly impacts productivity:

- If your team knows Azure Active Directory and .NET, Azure may be a natural fit
- If your team is strong in open-source tooling, AWS or GCP may align better
- Consider training costs, certification paths, and the local talent market

### Provider Strengths at a Glance

| Criterion | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| **Market share** | Largest; ~31% of market | Second; ~25% of market | Third; ~11% of market |
| **Service breadth** | Broadest catalog; 200+ services | Very broad; strong enterprise integration | Focused catalog; strong in data and ML |
| **Compute** | Widest VM family selection | Strong Windows and .NET support | Competitive pricing; live migration |
| **Data and analytics** | Redshift, EMR, Athena | Synapse, Fabric, Power BI | BigQuery (leader in serverless analytics) |
| **Machine learning** | SageMaker | Azure ML, OpenAI Service | Vertex AI, TensorFlow ecosystem |
| **Kubernetes** | EKS | AKS | GKE (most mature managed K8s) |
| **Enterprise integration** | Broad partner ecosystem | Deepest Microsoft 365 and Active Directory integration | Strong Google Workspace integration |
| **Open source** | Strong OSS service support | Growing OSS commitment | Born from open-source culture (K8s, TF) |
| **Pricing** | Complex; most discount options | Competitive; Azure Hybrid Benefit for Windows | Aggressive pricing; sustained use discounts |
| **Global infrastructure** | Most regions and availability zones | Second most regions; strong government cloud | Fewer regions; strong network backbone |

### Avoiding Vendor Lock-In

Vendor lock-in occurs when your architecture depends so heavily on a specific provider's proprietary services that switching becomes prohibitively expensive. Strategies to mitigate lock-in:

- **Use open standards** — Prefer Kubernetes over proprietary orchestration, PostgreSQL over proprietary databases, and OpenTelemetry over proprietary observability.
- **Abstract provider-specific services** — Use IaC tools like Terraform that support multiple providers. Isolate provider-specific calls behind abstraction layers.
- **Design for portability** — Containerize workloads. Use environment variables for configuration. Avoid deep integration with proprietary services where open alternatives exist.
- **Accept strategic lock-in** — Some proprietary services (e.g., BigQuery, DynamoDB) provide capabilities that are hard to replicate. Use them where the benefit outweighs the portability cost.

---

## Prerequisites

Before diving into the cloud computing topics, you should be familiar with:

- **Networking fundamentals** — IP addressing, DNS, HTTP/HTTPS, TCP/UDP, firewalls, and load balancing concepts
- **Operating system basics** — Linux command line, file systems, process management, and SSH
- **Programming fundamentals** — At least one backend language (Python, JavaScript, C#, Java, Go)
- **Version control** — Git basics for managing infrastructure as code and application deployments
- **Database fundamentals** — SQL, relational vs non-relational concepts, and basic data modeling
- **Docker basics** — Building container images, running containers, and understanding Dockerfiles

---

## Next Steps

Continue your cloud computing learning journey:

| File | Topic | Description |
|---|---|---|
| [01-AWS.md](01-AWS.md) | Amazon Web Services | AWS core services, account structure, IAM, networking, and best practices |
| [02-AZURE.md](02-AZURE.md) | Microsoft Azure | Azure core services, resource groups, RBAC, networking, and best practices |
| [03-GCP.md](03-GCP.md) | Google Cloud Platform | GCP core services, projects, IAM, networking, and best practices |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured curriculum with hands-on exercises |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Cloud Computing Fundamentals documentation |
