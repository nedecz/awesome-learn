# Microsoft Azure

## Table of Contents

1. [Overview](#overview)
2. [Azure Global Infrastructure](#azure-global-infrastructure)
3. [Identity and Access Management](#identity-and-access-management)
4. [Compute Services](#compute-services)
5. [Storage Services](#storage-services)
6. [Networking](#networking)
7. [Database Services](#database-services)
8. [Serverless and Integration](#serverless-and-integration)
9. [Infrastructure as Code](#infrastructure-as-code)
10. [Monitoring and Governance](#monitoring-and-governance)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Microsoft Azure is the second-largest cloud platform globally, offering 200+ services across compute, storage, networking, AI, and more. Launched in 2010 as Windows Azure, it rebranded to Microsoft Azure in 2014 and has grown into a comprehensive cloud ecosystem with deep integration into the Microsoft technology stack — including Microsoft 365, Dynamics 365, Visual Studio, GitHub, and Azure DevOps.

### Target Audience

- **Developers** building and deploying applications on Azure compute, storage, and database services
- **Architects** designing scalable, resilient cloud solutions using the Azure Well-Architected Framework
- **DevOps Engineers** provisioning infrastructure with ARM templates, Bicep, or Terraform
- **Engineering Managers** evaluating Azure services, cost models, and hybrid cloud strategies

### Scope

- Azure global infrastructure: regions, availability zones, geography pairs
- Identity: Microsoft Entra ID, RBAC, managed identities, resource hierarchy
- Compute: VMs, App Service, AKS, Container Instances, Functions
- Storage: Blob Storage, Azure Files, Disk Storage, Table Storage
- Networking: VNets, subnets, NSGs, Load Balancer, Application Gateway
- Databases: Azure SQL, Cosmos DB, PostgreSQL/MySQL, Cache for Redis
- Serverless: Functions, Logic Apps, Event Grid, Service Bus, API Management
- Infrastructure as code: ARM templates, Bicep, Terraform
- Monitoring: Azure Monitor, Log Analytics, Application Insights, Azure Policy

---

## Azure Global Infrastructure

Azure operates one of the largest global cloud networks, organized into regions, availability zones, and geography pairs.

### Regions

A **region** is a set of data centers deployed within a latency-defined perimeter and connected through a dedicated low-latency network. As of 2025, Azure operates 60+ regions across 140+ countries — more than any other cloud provider. Each region is identified by a name such as `eastus`, `westeurope`, or `southeastasia`.

### Availability Zones

**Availability Zones** are physically separate locations within a region, each with independent power, cooling, and networking. Regions that support zones have a minimum of three.

```
Azure Region: East US
──────────────────────────────────────────────
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Zone 1     │  │   Zone 2     │  │   Zone 3     │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │Data Ctrs │ │  │ │Data Ctrs │ │  │ │Data Ctrs │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       └───── Low-Latency Network ─────────┘
```

Deploy services as **zonal** (pinned to a zone) or **zone-redundant** (spread across zones). Zone-redundant is recommended for production.

### Region Pairs

Most Azure regions are paired with another region in the same **geography** (e.g., East US ↔ West US). Region pairs provide:

- **Sequential updates** — platform updates roll out to one region at a time
- **Disaster recovery** — paired regions are prioritized for restoration during outages
- **Data residency** — both regions reside in the same geography for compliance

### Choosing a Region

| Factor | Consideration |
|--------|--------------|
| **Latency** | Choose a region geographically close to your users |
| **Compliance** | Data residency requirements (e.g., GDPR, sovereignty) may mandate specific regions |
| **Service availability** | Not all services or VM sizes are available in every region |
| **Cost** | Pricing varies by region — compare using the Azure Pricing Calculator |
| **Region pairs** | Consider paired regions for disaster recovery planning |

---

## Identity and Access Management

Azure identity and access management is built on **Microsoft Entra ID** (formerly Azure Active Directory) and a hierarchical resource model.

### Microsoft Entra ID

Microsoft Entra ID is Azure's cloud-based identity and access management service. It provides authentication (SAML, OAuth 2.0, OpenID Connect), single sign-on across Azure and thousands of SaaS apps, multi-factor authentication, and conditional access policies combining signals like user location, device state, and risk level.

A **tenant** is a dedicated Entra ID instance representing an organization. Every Azure subscription is associated with exactly one tenant.

### Azure Resource Hierarchy

Azure organizes resources in a four-level hierarchy. Policies and access controls applied at higher levels are inherited by lower levels.

```
Azure Resource Hierarchy
────────────────────────
Management Groups
 └── Subscription: Production
      ├── Resource Group: rg-webapp-prod
      │    ├── App Service: app-frontend
      │    └── SQL Database: sql-orders
      └── Resource Group: rg-networking-prod
           └── Virtual Network: vnet-main
 └── Subscription: Development
      └── Resource Group: rg-webapp-dev
           └── App Service: app-frontend-dev
```

- **Management Groups** — containers for organizing multiple subscriptions. Apply Azure Policy and RBAC at this level for governance across the organization.
- **Subscriptions** — billing boundaries and access control boundaries. Each subscription trusts a single Entra ID tenant.
- **Resource Groups** — logical containers for resources that share the same lifecycle. Every Azure resource must belong to exactly one resource group.
- **Resources** — individual service instances (VMs, databases, storage accounts, etc.).

### Role-Based Access Control (RBAC)

Azure RBAC uses **role assignments** combining a security principal (who), a role definition (what), and a scope (where).

| Built-in Role | Permissions |
|---------------|-------------|
| **Owner** | Full access including the ability to assign roles to others |
| **Contributor** | Create and manage all resources but cannot grant access |
| **Reader** | View all resources but cannot make changes |
| **User Access Administrator** | Manage user access to Azure resources |

```bash
# Assign the Contributor role to a user at the resource group scope
az role assignment create \
  --assignee "alice@contoso.com" \
  --role "Contributor" \
  --resource-group "rg-webapp-prod"

# List role assignments for a resource group
az role assignment list \
  --resource-group "rg-webapp-prod" \
  --output table
```

### Managed Identities

**Managed identities** eliminate the need to manage credentials in code. Azure automatically handles the identity lifecycle and credential rotation.

- **System-assigned** — tied to a single resource, created and deleted with that resource
- **User-assigned** — standalone identity shared across multiple resources, managed independently

Managed identities authenticate to any service supporting Entra ID authentication (Key Vault, Storage, SQL Database) without storing secrets in code.

### Identity Best Practices

- **Use managed identities** instead of service principal secrets wherever possible
- **Apply least privilege** — assign the most restrictive built-in role that meets requirements
- **Enforce MFA and Conditional Access** for all human users
- **Use management groups** to apply policies consistently across subscriptions

---

## Compute Services

Azure offers compute services spanning virtual machines, managed platforms, containers, and serverless.

### Virtual Machines

Azure VMs provide on-demand, scalable computing with full OS control. VM sizes are organized into families:

| Family | Purpose | Example Sizes |
|--------|---------|---------------|
| **B-series** | Burstable | B2s, B4ms — dev/test, low-traffic web servers |
| **D-series** | General purpose | D4s_v5, D8s_v5 — application servers, databases |
| **F-series** | Compute optimized | F4s_v2, F16s_v2 — batch processing, gaming |
| **E-series** | Memory optimized | E4s_v5, E16s_v5 — in-memory analytics, caches |
| **N-series** | GPU accelerated | NC24ads_A100, ND96asr — ML training, HPC |
| **L-series** | Storage optimized | L8s_v3, L32s_v3 — large databases, data warehousing |

**Availability Sets** distribute VMs across fault domains (separate racks) and update domains (separate maintenance windows). **Virtual Machine Scale Sets** automatically adjust VM count based on demand.

### App Service

Azure App Service is a fully managed platform for hosting web applications, REST APIs, and mobile backends. Supports .NET, Java, Node.js, Python, PHP, and Ruby.

- **App Service Plans** — define compute resources shared by one or more apps
- **Deployment Slots** — staging environments with zero-downtime swap to production
- **Built-in autoscale** — scale based on metrics or schedule

```bash
# Create a web app with a deployment slot
az webapp create \
  --name my-web-app \
  --resource-group rg-webapp-prod \
  --plan my-app-plan \
  --runtime "DOTNET|8.0"

az webapp deployment slot create \
  --name my-web-app \
  --resource-group rg-webapp-prod \
  --slot staging
```

### Azure Kubernetes Service (AKS)

AKS is a managed Kubernetes service where Azure handles the control plane at no charge. You manage and pay for worker nodes. Key features include multiple node pools, cluster autoscaler, Azure CNI networking for native VNet integration, and built-in integration with Azure Monitor, Entra ID, and Key Vault.

### Azure Container Instances (ACI)

ACI is the simplest way to run a container in Azure — no VMs or orchestrator required. Ideal for short-lived tasks, build agents, and burst workloads from AKS via virtual nodes.

### Azure Functions

Azure Functions is the serverless compute platform. Write code triggered by events and Azure manages all infrastructure. Supports C#, JavaScript/TypeScript, Python, Java, and PowerShell.

- **Consumption plan** — auto-scales from zero, pay per execution
- **Premium plan** — pre-warmed instances, VNet connectivity
- **Dedicated plan** — runs on existing App Service Plan

### Compute Service Comparison

| Criteria | Virtual Machines | App Service | AKS | ACI | Functions |
|----------|-----------------|-------------|-----|-----|-----------|
| **Abstraction** | Full VM | Managed platform | Containers (K8s) | Single container | Functions |
| **Control** | Full OS access | Runtime-level | Container + cluster | Container only | Code only |
| **Startup time** | Minutes | Seconds | Seconds (pods) | Seconds | Milliseconds |
| **Max runtime** | Unlimited | Unlimited | Unlimited | Unlimited | 10 min (Consumption) |
| **Scaling** | Scale Sets | Built-in autoscale | Cluster autoscaler | Manual / burst | Auto (event-driven) |
| **Best for** | Legacy, full OS | Web apps, APIs | Microservices | Short-lived tasks | Event-driven, glue |

---

## Storage Services

Azure provides durable, scalable storage for objects, files, disks, and structured data.

### Blob Storage

Azure Blob Storage is an object storage service for unstructured data — documents, images, videos, logs, and backups. Blobs are organized into **containers** within a **storage account**. Types include block blobs (general uploads, up to 190.7 TB), append blobs (logging), and page blobs (VM disks).

### Storage Access Tiers

| Tier | Storage Cost | Access Cost | Min Retention | Best For |
|------|-------------|-------------|---------------|----------|
| **Hot** | Highest | Lowest | None | Frequently accessed data, active workloads |
| **Cool** | Lower | Higher | 30 days | Infrequently accessed, short-term backup |
| **Cold** | Lower still | Higher still | 90 days | Rarely accessed, compliance data |
| **Archive** | Lowest | Highest | 180 days | Long-term retention, regulatory archives |

**Lifecycle management policies** automatically transition blobs between tiers or delete them based on age. Archive tier requires rehydration (hours) before data can be read.

```bash
# Create a storage account and upload a blob
az storage account create \
  --name stwebappprod --resource-group rg-webapp-prod \
  --location eastus --sku Standard_LRS

az storage blob upload \
  --account-name stwebappprod --container-name uploads \
  --name report.pdf --file ./report.pdf
```

### Azure Files

Azure Files provides fully managed file shares accessible via **SMB** and **NFS** protocols. Use for shared application settings, lift-and-shift of on-premises file shares, and persistent storage for containers.

### Disk Storage

Azure Managed Disks are block-level storage volumes for Azure VMs. Azure handles replication and availability.

| Disk Type | Max IOPS | Max Throughput | Use Case |
|-----------|----------|----------------|----------|
| **Premium SSD v2** | 80,000 | 1,200 MB/s | Production databases, latency-sensitive |
| **Premium SSD** | 20,000 | 900 MB/s | Most production workloads |
| **Standard SSD** | 6,000 | 750 MB/s | Web servers, dev/test |
| **Standard HDD** | 2,000 | 500 MB/s | Backup, infrequent access |

### Table Storage

Azure Table Storage is a NoSQL key-value store for semi-structured datasets. It offers massive scale at low cost with a schema-less design. Each entity has a **PartitionKey** and **RowKey** for fast lookups. For more advanced querying, consider Cosmos DB Table API as an upgrade path.

---

## Networking

Azure networking is built around Virtual Networks, providing isolated, configurable network environments for cloud resources.

### Virtual Networks (VNets)

A **Virtual Network (VNet)** is the fundamental building block for private networking in Azure. When creating a VNet, you specify an address space using CIDR notation (e.g., `10.0.0.0/16`). VNets are scoped to a single region but can be connected to other VNets via peering.

### Subnets

A **subnet** segments a VNet into smaller address ranges with distinct access controls:

- **Public-facing subnets** — resources with public IPs or behind load balancers
- **Application subnets** — app servers, containers, internal services
- **Data subnets** — databases and storage with restricted access

### Network Security Groups (NSGs)

NSGs contain inbound and outbound **security rules** that filter traffic by source, destination, port, and protocol. NSGs can be attached to subnets or individual network interfaces.

| Property | Description |
|----------|-------------|
| **Priority** | Rules evaluated from lowest number (highest priority) to highest |
| **Action** | Allow or Deny |
| **Direction** | Inbound or Outbound |
| **Source / Destination** | IP address, CIDR, service tag, or application security group |
| **Protocol** | TCP, UDP, ICMP, or Any |

### Azure Load Balancer

Azure Load Balancer operates at **Layer 4** (TCP/UDP) and distributes inbound traffic across backend pool instances.

- **Public Load Balancer** — maps public IP addresses to private IPs of backend VMs
- **Internal Load Balancer** — distributes traffic within a VNet (for multi-tier applications)
- **Standard SKU** — zone-redundant, supports availability zones, required for production

### Application Gateway

Azure Application Gateway is a **Layer 7** (HTTP/HTTPS) load balancer with URL-based routing, SSL/TLS termination, Web Application Firewall (WAF) for OWASP top 10 protection, and autoscaling.

### Azure DNS

Azure DNS hosts DNS zones and provides name resolution using Microsoft's global network of name servers. Supports both **public DNS zones** (internet-facing) and **private DNS zones** (name resolution within VNets).

### VNet Peering

VNet peering connects two VNets so resources communicate using private IP addresses as if on the same network. Traffic stays on the Microsoft backbone — it never traverses the public internet.

- **Regional peering** — connects VNets in the same region
- **Global peering** — connects VNets across different regions

### VNet Architecture Diagram

```
Azure VNet: 10.0.0.0/16
┌──────────────────────────────────────────────────────┐
│                     ┌──────────┐                     │
│                     │ Internet │                     │
│                     └────┬─────┘                     │
│                  ┌───────┴────────┐                  │
│                  │  App Gateway   │                  │
│                  │  (WAF + L7 LB) │                  │
│                  └───────┬────────┘                  │
│  ┌─ Zone 1 ─────────────┼─────────────────────────┐ │
│  │  Subnet: 10.0.1.0/24 (Web Tier)    [NSG: 443] │ │
│  │  ┌──────────┐  ┌──────────┐                    │ │
│  │  │ App Svc  │  │ App Svc  │                    │ │
│  │  └────┬─────┘  └────┬─────┘                    │ │
│  │  Subnet: 10.0.2.0/24 (App Tier)    [NSG: app] │ │
│  │  ┌──────────┐  ┌──────────┐                    │ │
│  │  │ AKS Pods │  │ AKS Pods │                    │ │
│  │  └────┬─────┘  └────┬─────┘                    │ │
│  │  Subnet: 10.0.3.0/24 (Data Tier)   [NSG: DB]  │ │
│  │  ┌──────────┐  ┌──────────┐                    │ │
│  │  │Azure SQL │  │  Redis   │                    │ │
│  │  └──────────┘  └──────────┘                    │ │
│  └────────────────────────────────────────────────┘ │
│  ┌─ Zone 2 (mirrors Zone 1 for HA) ──────────────┐ │
│  │  Identical subnets with standby replicas       │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## Database Services

Azure offers purpose-built database services for relational, NoSQL, and in-memory workloads.

### Azure SQL Database

Azure SQL Database is a fully managed relational database engine based on the latest stable SQL Server. It handles patching, backups, and high availability automatically.

- **Purchasing models** — DTU-based (bundled) or vCore-based (independent compute/storage)
- **Service tiers** — General Purpose, Business Critical (low-latency, in-memory OLTP), Hyperscale (up to 100 TB)
- **Elastic Pools** — share resources across multiple databases for cost efficiency

### Cosmos DB

Azure Cosmos DB is a globally distributed, multi-model NoSQL database for single-digit millisecond latency at any scale.

**APIs** — Cosmos DB supports multiple access patterns:

- **NoSQL** (native) — document model with SQL-like query syntax
- **MongoDB** — wire protocol compatible with MongoDB
- **Cassandra** — column-family model
- **Gremlin** — graph database
- **Table** — key-value model compatible with Azure Table Storage

**Consistency Levels** — Cosmos DB offers five tunable consistency levels, trading off between consistency and performance:

| Level | Guarantee | Latency | Use Case |
|-------|-----------|---------|----------|
| **Strong** | Linearizable reads always see latest write | Highest | Financial transactions, inventory |
| **Bounded Staleness** | Reads lag by at most K versions or T time | High | Leaderboards, order tracking |
| **Session** | Reads within a session see own writes | Medium | Shopping carts, user profiles (default) |
| **Consistent Prefix** | Reads never see out-of-order writes | Low | Social feeds, activity logs |
| **Eventual** | No ordering guarantee, lowest latency | Lowest | Non-critical counters, analytics |

Session consistency is the default and most widely used — it ensures a user always sees their own writes while providing good performance.

### Azure Database for PostgreSQL / MySQL

Azure provides fully managed services for open-source relational databases. **Azure Database for PostgreSQL — Flexible Server** and **Azure Database for MySQL — Flexible Server** both support community versions, burstable/general-purpose/memory-optimized tiers, zone-redundant HA, and automated backups with up to 35-day retention.

### Azure Cache for Redis

Azure Cache for Redis is a fully managed in-memory data store for sub-millisecond caching, session management, and real-time analytics.

| Tier | Description |
|------|-------------|
| **Basic** | Single node, no SLA (dev/test) |
| **Standard** | Two-node replicated, 99.9% SLA |
| **Premium** | Clustering, persistence, VNet, geo-replication |
| **Enterprise** | Redis Enterprise modules (RediSearch, RedisJSON) |

### Database Service Comparison

| Criteria | Azure SQL | Cosmos DB | PostgreSQL/MySQL | Cache for Redis |
|----------|-----------|-----------|------------------|-----------------|
| **Data model** | Relational (SQL) | Multi-model (NoSQL) | Relational (SQL) | Key-value (in-memory) |
| **Scaling** | Vertical + read replicas | Horizontal (global) | Vertical + read replicas | Cluster sharding |
| **Latency** | Low ms | Single-digit ms | Low ms | Sub-millisecond |
| **Best for** | .NET/SQL Server workloads | Global apps, multi-model | Open-source, PostgreSQL/MySQL apps | Caching, sessions, real-time |

---

## Serverless and Integration

Azure provides a comprehensive set of serverless and integration services for building event-driven and workflow-based architectures.

### Azure Functions

Azure Functions executes event-driven code without managing infrastructure. Functions are organized within a **Function App** and activated by **triggers**.

- **Triggers** — the event that starts execution (HTTP request, timer, queue message, blob upload, Cosmos DB change feed, Event Grid event)
- **Bindings** — declarative input/output connections that eliminate boilerplate for reading from or writing to storage, queues, and databases

```json
{
  "bindings": [
    { "type": "httpTrigger", "direction": "in", "name": "req", "methods": ["get", "post"] },
    { "type": "http", "direction": "out", "name": "res" },
    { "type": "queue", "direction": "out", "name": "outputQueue", "queueName": "order-processing", "connection": "AzureWebJobsStorage" }
  ]
}
```

### Logic Apps

Azure Logic Apps is a low-code workflow automation platform with a visual designer and 400+ connectors. Use for B2B integration, approval workflows, and orchestrating processes across systems without writing code.

### Event Grid

Azure Event Grid is a fully managed event routing service using publish-subscribe. Publishers send events to **topics**, **event subscriptions** define filtering and routing, and **event handlers** (Functions, Logic Apps, webhooks, Service Bus) process the events. Supports both Azure-native events and custom events with at-least-once delivery.

### Service Bus

Azure Service Bus is an enterprise message broker with queues (point-to-point, FIFO, dead-lettering) and topics/subscriptions (publish-subscribe with rule-based filtering). Use when you need guaranteed delivery, transactions, message ordering, or large messages (up to 100 MB Premium).

### API Management

Azure API Management (APIM) is a gateway for publishing, securing, and monitoring APIs. It provides a gateway for applying policies (rate limiting, caching, transformation), an auto-generated developer portal, and XML-based policy rules for authentication, throttling, and header manipulation.

### When to Use Each Integration Service

| Service | Pattern | Best For |
|---------|---------|----------|
| **Functions** | Event-driven compute | Processing events, running scheduled tasks, building APIs |
| **Logic Apps** | Workflow automation | Multi-step integration, approval flows, no-code connectors |
| **Event Grid** | Event routing | Reactive programming, resource events, fan-out |
| **Service Bus** | Enterprise messaging | Decoupling services, ordered delivery, transactions |
| **API Management** | API gateway | Centralized API governance, rate limiting, analytics |

---

## Infrastructure as Code

Managing Azure resources through code ensures repeatability, version control, and consistent deployments across environments.

### ARM Templates

Azure Resource Manager (ARM) templates are JSON files that define infrastructure declaratively. ARM is the native deployment engine — every Azure resource operation goes through the ARM API. Templates are idempotent, handle dependency resolution, and support what-if previews before deploying.

### Bicep

**Bicep** is a domain-specific language (DSL) that compiles to ARM templates. It provides a cleaner, more concise syntax while retaining full ARM capability. Bicep is the recommended approach for new Azure IaC projects.

```bicep
// Deploy a storage account with blob service
param location string = resourceGroup().location
param storageAccountName string = 'stwebapp${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
  }
}

output storageAccountId string = storageAccount.id
```

```bash
# Deploy a Bicep file
az deployment group create \
  --resource-group rg-webapp-prod \
  --template-file main.bicep \
  --parameters storageAccountName=stwebappprod
```

### Terraform Support

HashiCorp **Terraform** is a widely used multi-cloud IaC tool with a mature Azure provider (`azurerm`). Choose Terraform when managing infrastructure across multiple cloud providers or when the team has existing Terraform expertise.

### IaC Best Practices

- **Use modules** for reusable infrastructure components
- **Store templates in version control** alongside application code
- **Parameterize** environment-specific values instead of hardcoding
- **Use what-if / plan** to preview changes before every deployment

---

## Monitoring and Governance

Azure provides a unified monitoring platform and governance tools to maintain visibility, compliance, and control across your cloud environment.

### Azure Monitor

Azure Monitor is the centralized monitoring service for collecting, analyzing, and acting on telemetry.

- **Metrics** — numeric time-series data with 1-minute granularity and 93-day retention
- **Logs** — structured event data sent to Log Analytics for querying
- **Alerts** — trigger notifications or actions when conditions are met
- **Autoscale** — adjust resources based on metric thresholds

### Log Analytics

**Log Analytics workspaces** are the central repository for log data in Azure Monitor. Use **Kusto Query Language (KQL)** to query, analyze, and visualize data across all sources.

```bash
# Example KQL query: find HTTP 500 errors in the last 24 hours
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AppRequests | where ResultCode == 500 | summarize count() by bin(TimeGenerated, 1h)" \
  --timespan PT24H \
  --output table
```

### Application Insights

Application Insights is an APM feature of Azure Monitor for live web applications. It provides request tracking, distributed tracing across microservices, live metrics, application dependency maps, and AI-powered anomaly detection.

### Azure Policy

Azure Policy enforces organizational standards and assesses compliance at scale. Policies are JSON rules evaluated against resource properties during creation and updates. Group related policies into **initiatives** (e.g., CIS Benchmark) and assign them at management group, subscription, or resource group scope. Effects include Audit, Deny, Modify, and DeployIfNotExists.

### Azure Blueprints

Azure Blueprints packages role assignments, policy assignments, ARM templates, and resource groups into a single deployable definition. Use for repeatable, governed environment provisioning — such as setting up new subscriptions with required policies and security controls.

### Governance Best Practices

- **Tag all resources** with owner, environment, and cost center for tracking
- **Apply policies at the management group level** so they cascade to all subscriptions
- **Use resource locks** (ReadOnly, Delete) on critical resources to prevent accidental changes
- **Centralize logs** in a single Log Analytics workspace for cross-resource querying

---

## Next Steps

Continue your cloud computing learning journey:

| File | Topic | Description |
|------|-------|-------------|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Cloud Computing Fundamentals | Core cloud concepts, service models, deployment models, provider comparison |
| [01-AWS.md](01-AWS.md) | Amazon Web Services | AWS global infrastructure, IAM, compute, storage, networking, databases |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Azure documentation |
