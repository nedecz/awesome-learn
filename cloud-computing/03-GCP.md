# Google Cloud Platform (GCP)

## Table of Contents

1. [Overview](#overview)
2. [GCP Global Infrastructure](#1-gcp-global-infrastructure)
3. [Identity and Access Management](#2-identity-and-access-management)
4. [Compute Services](#3-compute-services)
5. [Storage Services](#4-storage-services)
6. [Networking](#5-networking)
7. [Database Services](#6-database-services)
8. [Serverless and Application Services](#7-serverless-and-application-services)
9. [Infrastructure as Code](#8-infrastructure-as-code)
10. [Monitoring and Operations](#9-monitoring-and-operations)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Google Cloud Platform (GCP) is a suite of cloud computing services offered by Google that runs on the same infrastructure Google uses internally for its end-user products such as Search, Gmail, and YouTube. GCP provides a broad set of compute, storage, networking, data analytics, and machine learning services that enable organizations to build, deploy, and scale applications and workloads in the cloud.

GCP differentiates itself through its strengths in data analytics, machine learning, open-source technologies, and its global network backbone — one of the largest and most advanced in the world.

### Target Audience

This guide is intended for:

- **Developers** looking to build and deploy applications on GCP
- **System administrators** transitioning workloads to the cloud
- **Solutions architects** evaluating GCP services for infrastructure design
- **Students and newcomers** beginning their cloud computing journey

### Scope

This document covers the core GCP services across compute, storage, networking, databases, serverless, infrastructure as code, and monitoring. It provides a foundational understanding of each service category with practical examples and comparison tables to aid in service selection.

---

## 1. GCP Global Infrastructure

GCP's global infrastructure is built on Google's private network, one of the largest in the world, spanning undersea cables and points of presence across the globe.

### Regions and Zones

- **Region**: A specific geographical location where GCP resources are hosted (e.g., `us-central1`, `europe-west1`, `asia-east1`). Each region is an independent geographic area.
- **Zone**: An isolated deployment area within a region. Each region contains three or more zones (e.g., `us-central1-a`, `us-central1-b`, `us-central1-c`). Zones are designed to be isolated from failures in other zones.
- **Multi-region**: A large geographic area containing two or more regions. Multi-region locations such as `US`, `EU`, and `ASIA` are used for services like Cloud Storage to provide higher availability and geo-redundancy.

### Points of Presence

Google Cloud has over 180 network edge locations worldwide. These points of presence (PoPs) and edge caching nodes bring content closer to end users, reducing latency for services like Cloud CDN and media delivery.

### Region Selection Considerations

When choosing a region for your resources, consider:

| Factor | Description |
|---|---|
| **Latency** | Select a region close to your end users for lower latency |
| **Compliance** | Ensure data residency requirements are met for regulatory compliance |
| **Service availability** | Not all GCP services are available in every region |
| **Cost** | Pricing varies by region; some regions are more expensive than others |
| **Redundancy** | Use multiple zones or regions for high availability and disaster recovery |

### Listing Available Regions

```bash
# List all available regions
gcloud compute regions list

# List all available zones
gcloud compute zones list

# Describe a specific region
gcloud compute regions describe us-central1
```

---

## 2. Identity and Access Management

GCP Identity and Access Management (IAM) enables you to control who has access to what resources. IAM follows the principle of least privilege, granting only the permissions needed to perform a task.

### Resource Hierarchy

GCP organizes resources in a hierarchy that directly maps to how IAM policies are inherited:

```
Organization (example.com)
├── Folder (Engineering)
│   ├── Project (web-app-prod)
│   │   ├── Compute Engine VM
│   │   ├── Cloud Storage Bucket
│   │   └── Cloud SQL Instance
│   └── Project (web-app-dev)
│       ├── Compute Engine VM
│       └── Cloud Storage Bucket
├── Folder (Marketing)
│   └── Project (marketing-analytics)
│       └── BigQuery Dataset
└── Folder (Finance)
    └── Project (finance-reporting)
        └── Cloud SQL Instance
```

IAM policies set at a higher level in the hierarchy are inherited by all resources below. A policy set on the Organization applies to all Folders, Projects, and Resources within it.

### IAM Concepts

- **Principal (Member)**: The identity requesting access — a Google Account, service account, Google Group, or Cloud Identity domain.
- **Role**: A collection of permissions. Roles are granted to principals.
- **Policy**: A binding of one or more principals to a role, attached to a resource.

### Role Types

| Role Type | Description | Example |
|---|---|---|
| **Basic roles** | Broad roles that existed prior to IAM (Owner, Editor, Viewer). Use sparingly. | `roles/owner`, `roles/editor`, `roles/viewer` |
| **Predefined roles** | Fine-grained roles managed by Google for specific services. | `roles/compute.instanceAdmin`, `roles/storage.objectViewer` |
| **Custom roles** | User-defined roles with specific permissions tailored to your needs. | `roles/myCompany.appDeployer` |

### Service Accounts

Service accounts are special accounts used by applications and virtual machines, not people. They authenticate workloads and call GCP APIs.

```bash
# Create a service account
gcloud iam service-accounts create my-app-sa \
    --display-name="My Application Service Account" \
    --description="Used by the web application to access Cloud Storage"

# Grant a role to the service account on a project
gcloud projects add-iam-policy-binding my-project-id \
    --member="serviceAccount:my-app-sa@my-project-id.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# List all service accounts in a project
gcloud iam service-accounts list --project=my-project-id
```

### IAM Best Practices

- **Use predefined roles** instead of basic roles whenever possible
- **Apply the principle of least privilege** — grant only the minimum permissions required
- **Use service accounts** for application authentication instead of user credentials
- **Audit IAM policies** regularly with Cloud Audit Logs
- **Use IAM Conditions** for attribute-based access control (e.g., time-based or resource-based restrictions)

---

## 3. Compute Services

GCP offers a range of compute services from full virtual machines to fully managed serverless platforms.

### Compute Engine

Compute Engine provides virtual machines (VMs) running on Google's infrastructure. You can choose from predefined or custom machine types and a variety of operating systems.

```bash
# Create a VM instance
gcloud compute instances create my-vm \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --boot-disk-size=20GB \
    --tags=http-server

# List all VM instances
gcloud compute instances list

# SSH into a VM
gcloud compute ssh my-vm --zone=us-central1-a
```

### Google Kubernetes Engine (GKE)

GKE is a managed Kubernetes service for deploying, managing, and scaling containerized applications.

```bash
# Create a GKE cluster
gcloud container clusters create my-cluster \
    --zone=us-central1-a \
    --num-nodes=3 \
    --machine-type=e2-standard-4

# Get cluster credentials for kubectl
gcloud container clusters get-credentials my-cluster \
    --zone=us-central1-a
```

### Cloud Run

Cloud Run is a fully managed platform that automatically scales stateless containers. It abstracts away all infrastructure management.

```bash
# Deploy a container to Cloud Run
gcloud run deploy my-service \
    --image=gcr.io/my-project/my-app:latest \
    --platform=managed \
    --region=us-central1 \
    --allow-unauthenticated
```

### App Engine

App Engine is a fully managed platform for building web applications and APIs. It supports standard runtimes (Python, Java, Node.js, Go, PHP, Ruby) and custom runtimes.

### Cloud Functions

Cloud Functions is an event-driven, serverless compute service for lightweight, single-purpose functions that respond to cloud events.

### Compute Services Comparison

| Service | Use Case | Scaling | Management Level | Pricing Model |
|---|---|---|---|---|
| **Compute Engine** | Full control over VMs, lift-and-shift | Manual or autoscaler | IaaS | Per-second billing |
| **GKE** | Container orchestration at scale | Cluster autoscaler | Managed Kubernetes | Cluster + node costs |
| **Cloud Run** | Stateless containers, microservices | Automatic (to zero) | Fully managed | Pay per request |
| **App Engine** | Web apps and APIs | Automatic | PaaS | Instance hours |
| **Cloud Functions** | Event-driven, lightweight tasks | Automatic (to zero) | FaaS | Per invocation |

---

## 4. Storage Services

GCP provides a variety of storage services for different data types, access patterns, and performance requirements.

### Cloud Storage

Cloud Storage is an object storage service for unstructured data. It offers multiple storage classes optimized for different access frequencies and cost requirements.

#### Storage Classes

| Storage Class | Min Storage Duration | Use Case | Availability SLA |
|---|---|---|---|
| **Standard** | None | Frequently accessed data, websites, streaming | 99.95% (multi-region: 99.99%) |
| **Nearline** | 30 days | Data accessed less than once a month | 99.9% |
| **Coldline** | 90 days | Data accessed less than once a quarter | 99.9% |
| **Archive** | 365 days | Long-term archival, regulatory compliance | 99.9% |

All storage classes share the same APIs, latency (time to first byte in milliseconds), and throughput. The key differences are in storage cost and retrieval cost.

```bash
# Create a Cloud Storage bucket
gcloud storage buckets create gs://my-unique-bucket-name \
    --location=us-central1 \
    --default-storage-class=standard

# Upload a file
gcloud storage cp my-file.txt gs://my-unique-bucket-name/

# Set lifecycle rules to transition to Nearline after 30 days
cat > lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30}
    }
  ]
}
EOF
gcloud storage buckets update gs://my-unique-bucket-name \
    --lifecycle-file=lifecycle.json
```

### Persistent Disk

Persistent Disk provides durable block storage for Compute Engine VMs and GKE nodes. Disks are automatically encrypted and replicated within a zone.

- **Standard (pd-standard)**: Cost-effective HDD-backed storage for sequential read/write workloads
- **Balanced (pd-balanced)**: SSD-backed storage balancing price and performance
- **SSD (pd-ssd)**: High-performance SSD-backed storage for random IOPS workloads
- **Extreme (pd-extreme)**: Highest performance SSD option for demanding database workloads

### Filestore

Filestore is a managed NFS file storage service for applications that require a file system interface. It is ideal for shared storage across multiple Compute Engine VMs or GKE pods.

```bash
# Create a Filestore instance
gcloud filestore instances create my-filestore \
    --zone=us-central1-a \
    --tier=BASIC_HDD \
    --file-share=name=vol1,capacity=1TB \
    --network=name=default
```

---

## 5. Networking

GCP networking is built on Google's global software-defined network, providing high performance, reliability, and security.

### VPC Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        VPC Network (Global)                         │
│                                                                     │
│  ┌──────────────────────┐       ┌──────────────────────┐           │
│  │  Region: us-central1 │       │  Region: europe-west1│           │
│  │                      │       │                      │           │
│  │ ┌──────────────────┐ │       │ ┌──────────────────┐ │           │
│  │ │ Subnet: 10.0.1.0 │ │       │ │ Subnet: 10.0.3.0 │ │           │
│  │ │    /24            │ │       │ │    /24            │ │           │
│  │ │ ┌──┐ ┌──┐ ┌──┐   │ │       │ │ ┌──┐ ┌──┐       │ │           │
│  │ │ │VM│ │VM│ │VM│   │ │       │ │ │VM│ │VM│       │ │           │
│  │ │ └──┘ └──┘ └──┘   │ │       │ │ └──┘ └──┘       │ │           │
│  │ └──────────────────┘ │       │ └──────────────────┘ │           │
│  │                      │       │                      │           │
│  │ ┌──────────────────┐ │       │ ┌──────────────────┐ │           │
│  │ │ Subnet: 10.0.2.0 │ │       │ │ Subnet: 10.0.4.0 │ │           │
│  │ │    /24            │ │       │ │    /24            │ │           │
│  │ │ ┌──┐ ┌──┐        │ │       │ │ ┌──┐             │ │           │
│  │ │ │VM│ │VM│        │ │       │ │ │VM│             │ │           │
│  │ │ └──┘ └──┘        │ │       │ │ └──┘             │ │           │
│  │ └──────────────────┘ │       │ └──────────────────┘ │           │
│  └──────────────────────┘       └──────────────────────┘           │
│                                                                     │
│  ┌────────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │   Firewall Rules   │  │  Cloud Router     │  │  Cloud NAT     │  │
│  └────────────────────┘  └──────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐       ┌───────────────────┐
│  Cloud Load     │       │  VPC Peering /    │
│  Balancing      │       │  Cloud VPN /      │
│                 │       │  Interconnect     │
└─────────────────┘       └───────────────────┘
```

### Key Networking Components

**VPC (Virtual Private Cloud)**: VPC networks in GCP are global resources. Subnets are regional, but a single VPC can span all regions without any interconnection needed.

**Subnets**: Regional resources with a defined IP range. GCP supports auto-mode (automatic subnets in each region) and custom-mode (user-defined subnets) VPCs.

**Firewall Rules**: GCP firewall rules are stateful and applied at the VPC level. Rules can target instances by tags or service accounts.

```bash
# Create a custom VPC
gcloud compute networks create my-vpc \
    --subnet-mode=custom

# Create a subnet
gcloud compute networks subnets create my-subnet \
    --network=my-vpc \
    --region=us-central1 \
    --range=10.0.1.0/24

# Create a firewall rule to allow HTTP traffic
gcloud compute firewall-rules create allow-http \
    --network=my-vpc \
    --allow=tcp:80 \
    --target-tags=http-server \
    --source-ranges=0.0.0.0/0
```

### Cloud Load Balancing

GCP offers several load balancing options:

- **Global HTTP(S) Load Balancer**: Layer 7 load balancing across regions with URL-based routing
- **Regional TCP/UDP Load Balancer**: Layer 4 load balancing within a region
- **Internal Load Balancer**: Load balancing for internal services within a VPC
- **SSL Proxy / TCP Proxy**: Global load balancing for non-HTTP SSL and TCP traffic

### Cloud DNS

Cloud DNS is a scalable, reliable, and managed DNS service running on Google's infrastructure. It supports both public and private DNS zones.

### Cloud CDN

Cloud CDN caches HTTP(S) content at Google's globally distributed edge points of presence, reducing latency for users and offloading traffic from your backends.

### VPC Peering

VPC Network Peering allows private connectivity between two VPC networks, whether they are in the same project, different projects, or different organizations. Peered networks exchange routes for private IP communication.

---

## 6. Database Services

GCP provides managed database services for relational, NoSQL, analytical, and wide-column workloads.

### Cloud SQL

Cloud SQL is a fully managed relational database service supporting MySQL, PostgreSQL, and SQL Server. It handles replication, backups, and patching automatically.

```bash
# Create a Cloud SQL PostgreSQL instance
gcloud sql instances create my-postgres \
    --database-version=POSTGRES_15 \
    --tier=db-custom-2-8192 \
    --region=us-central1 \
    --availability-type=REGIONAL \
    --storage-size=50GB \
    --storage-auto-increase

# Create a database
gcloud sql databases create my-database --instance=my-postgres

# Create a user
gcloud sql users create my-user \
    --instance=my-postgres \
    --password=securepassword123
```

### Cloud Spanner

Cloud Spanner is a globally distributed, horizontally scalable relational database that provides ACID transactions and SQL semantics at virtually unlimited scale. It is designed for mission-critical applications requiring high availability (99.999% SLA).

### Firestore

Firestore is a serverless, NoSQL document database built for automatic scaling, high performance, and ease of application development. It supports real-time listeners, offline access, and ACID transactions.

### Cloud Bigtable

Cloud Bigtable is a fully managed, wide-column NoSQL database designed for large analytical and operational workloads. It is ideal for time-series data, IoT data, and high-throughput analytics.

### BigQuery

BigQuery is a serverless, highly scalable enterprise data warehouse designed for business agility. It supports standard SQL and can analyze petabytes of data with ease.

```sql
-- Example BigQuery query
SELECT
  DATE(created_at) AS order_date,
  COUNT(*) AS total_orders,
  SUM(amount) AS total_revenue
FROM
  `my-project.sales_dataset.orders`
WHERE
  created_at >= TIMESTAMP('2025-01-01')
GROUP BY
  order_date
ORDER BY
  order_date DESC
LIMIT 30;
```

### Database Services Comparison

| Service | Type | Use Case | Scaling | Max Availability SLA |
|---|---|---|---|---|
| **Cloud SQL** | Relational (MySQL, PostgreSQL, SQL Server) | Traditional web apps, OLTP | Vertical (read replicas) | 99.95% |
| **Cloud Spanner** | Relational (globally distributed) | Global transactions, financial systems | Horizontal | 99.999% |
| **Firestore** | Document (NoSQL) | Mobile/web apps, real-time sync | Automatic | 99.999% |
| **Bigtable** | Wide-column (NoSQL) | IoT, time-series, analytics | Horizontal (add nodes) | 99.99% |
| **BigQuery** | Data warehouse (analytical) | Analytics, BI, ML workloads | Serverless | 99.99% |

---

## 7. Serverless and Application Services

GCP provides a comprehensive set of serverless services for building event-driven, loosely coupled applications.

### Cloud Functions

Cloud Functions is a lightweight, event-driven compute platform for single-purpose functions. Functions are triggered by events from Cloud Storage, Pub/Sub, HTTP requests, Firestore, and more.

```python
# Example Cloud Function (Python) triggered by HTTP
import functions_framework

@functions_framework.http
def hello_world(request):
    """HTTP-triggered Cloud Function."""
    name = request.args.get("name", "World")
    return f"Hello, {name}!"
```

### Cloud Run

Cloud Run runs stateless containers that are invocable via HTTP requests or event triggers. It is suitable for microservices, APIs, and web applications.

### Pub/Sub

Cloud Pub/Sub is a fully managed, real-time messaging service for asynchronous communication between services. It decouples senders and receivers, enabling event-driven architectures.

```bash
# Create a topic
gcloud pubsub topics create my-topic

# Create a subscription
gcloud pubsub subscriptions create my-subscription \
    --topic=my-topic

# Publish a message
gcloud pubsub topics publish my-topic \
    --message="Hello from Pub/Sub"

# Pull messages
gcloud pubsub subscriptions pull my-subscription --auto-ack
```

### Eventarc

Eventarc enables you to build event-driven architectures by routing events from Google services, SaaS integrations, and your own applications to Cloud Run, Cloud Functions, or GKE targets.

### Workflows

Cloud Workflows is a serverless orchestration service that combines services (Cloud Functions, Cloud Run, APIs) into automated workflows defined in YAML or JSON.

### When to Use Each Service

| Scenario | Recommended Service |
|---|---|
| Short-lived, single-purpose event handlers | **Cloud Functions** |
| Containerized microservices and APIs | **Cloud Run** |
| Asynchronous message passing between services | **Pub/Sub** |
| Routing events from GCP services to targets | **Eventarc** |
| Orchestrating multi-step, long-running processes | **Workflows** |
| Real-time stream processing at scale | **Pub/Sub + Dataflow** |

---

## 8. Infrastructure as Code

Infrastructure as Code (IaC) allows you to define and manage GCP resources using declarative configuration files, enabling version control, repeatability, and automation.

### Cloud Deployment Manager

Deployment Manager is Google's native IaC tool. It uses YAML templates to define GCP resources.

```yaml
# deployment.yaml - Deployment Manager template
resources:
- name: my-vm
  type: compute.v1.instance
  properties:
    zone: us-central1-a
    machineType: zones/us-central1-a/machineTypes/e2-medium
    disks:
    - deviceName: boot
      type: PERSISTENT
      boot: true
      autoDelete: true
      initializeParams:
        sourceImage: projects/debian-cloud/global/images/family/debian-12
    networkInterfaces:
    - network: global/networks/default
      accessConfigs:
      - name: External NAT
        type: ONE_TO_ONE_NAT
```

```bash
# Deploy resources with Deployment Manager
gcloud deployment-manager deployments create my-deployment \
    --config=deployment.yaml
```

### Terraform on GCP

Terraform by HashiCorp is a widely adopted, multi-cloud IaC tool with strong GCP support through the Google provider.

```hcl
# main.tf - Terraform configuration for GCP
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project-id"
  region  = "us-central1"
}

# Create a VPC network
resource "google_compute_network" "vpc" {
  name                    = "my-vpc"
  auto_create_subnetworks = false
}

# Create a subnet
resource "google_compute_subnetwork" "subnet" {
  name          = "my-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
  network       = google_compute_network.vpc.id
}

# Create a Compute Engine instance
resource "google_compute_instance" "vm" {
  name         = "my-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    access_config {}
  }

  tags = ["http-server"]
}
```

```bash
# Initialize and apply Terraform
terraform init
terraform plan
terraform apply
```

### Choosing Between Deployment Manager and Terraform

| Consideration | Deployment Manager | Terraform |
|---|---|---|
| **Scope** | GCP-only | Multi-cloud |
| **Language** | YAML / Jinja2 / Python | HCL |
| **Community** | Smaller | Very large |
| **State management** | Managed by GCP | Local or remote backend |
| **Recommendation** | Legacy or GCP-only shops | Most teams and projects |

---

## 9. Monitoring and Operations

GCP's operations suite (formerly Stackdriver) provides integrated monitoring, logging, tracing, and error reporting for applications running on GCP and beyond.

### Cloud Monitoring

Cloud Monitoring collects metrics, events, and metadata from GCP services, hosted uptime probes, and application instrumentation. It provides dashboards, alerting, and uptime checks.

Key capabilities:

- **Metrics Explorer**: Query and visualize metrics from over 150 GCP services
- **Dashboards**: Build custom dashboards or use pre-built dashboards for GCP services
- **Alerting policies**: Configure alerts based on metric thresholds, absence conditions, or forecasted values
- **Uptime checks**: Monitor the availability of public URLs, TCP ports, or internal services

```bash
# Create an alerting policy (using gcloud)
gcloud alpha monitoring policies create \
    --display-name="High CPU Alert" \
    --condition-display-name="CPU over 80%" \
    --condition-filter='resource.type = "gce_instance" AND metric.type = "compute.googleapis.com/instance/cpu/utilization"' \
    --condition-threshold-value=0.8 \
    --condition-threshold-comparison=COMPARISON_GT \
    --condition-threshold-duration=300s \
    --notification-channels=projects/my-project/notificationChannels/12345
```

### Cloud Logging

Cloud Logging is a fully managed service for storing, searching, analyzing, and alerting on log data from GCP services and applications.

```bash
# Read recent logs for a Compute Engine instance
gcloud logging read 'resource.type="gce_instance" AND resource.labels.instance_id="1234567890"' \
    --limit=20 \
    --format=json

# Create a log-based metric
gcloud logging metrics create error-count \
    --description="Count of error log entries" \
    --log-filter='severity>=ERROR'
```

Key features:

- **Log Explorer**: Search, filter, and analyze logs in real time
- **Log Router**: Route logs to Cloud Storage, BigQuery, or Pub/Sub for further processing
- **Log-based metrics**: Create custom metrics from log entries for monitoring and alerting
- **Audit Logs**: Automatically captures Admin Activity, Data Access, and System Event logs

### Cloud Trace

Cloud Trace is a distributed tracing system that collects latency data from applications. It helps you understand how requests propagate through your services and identify performance bottlenecks.

- Automatic trace collection for App Engine, Cloud Run, and Cloud Functions
- Client libraries available for custom instrumentation (supports OpenTelemetry)
- Integration with Cloud Logging for correlated log and trace analysis

### Error Reporting

Error Reporting aggregates and displays errors produced by cloud services and applications. It groups errors by root cause, tracks error frequency over time, and sends notifications for new errors.

- Automatic error detection for App Engine, Cloud Functions, Cloud Run, GKE, and Compute Engine
- Stack trace analysis and deduplication
- Integration with issue trackers for error assignment

### Operations Best Practices

- **Set up dashboards** for each service and environment from day one
- **Configure alerting policies** with meaningful thresholds and appropriate notification channels
- **Export logs** to BigQuery for long-term analysis and compliance
- **Use structured logging** (JSON format) for easier searching and filtering
- **Implement distributed tracing** across microservices for end-to-end visibility
- **Review audit logs** regularly for security monitoring and compliance

---

## Next Steps

| Resource | Description | Link |
|---|---|---|
| **GCP Free Tier** | Explore GCP services with free tier resources | [cloud.google.com/free](https://cloud.google.com/free) |
| **Google Cloud Skills Boost** | Hands-on labs and learning paths | [cloudskillsboost.google](https://www.cloudskillsboost.google/) |
| **GCP Architecture Center** | Reference architectures and best practices | [cloud.google.com/architecture](https://cloud.google.com/architecture) |
| **Cloud Certification** | Professional and Associate certifications | [cloud.google.com/certification](https://cloud.google.com/certification) |
| **Terraform GCP Provider** | Terraform documentation for GCP resources | [registry.terraform.io/providers/hashicorp/google](https://registry.terraform.io/providers/hashicorp/google/latest/docs) |
| **GCP Pricing Calculator** | Estimate costs for your workloads | [cloud.google.com/products/calculator](https://cloud.google.com/products/calculator) |

---

## Version History

| Version | Date | Description |
|---|---|---|
| 1.0 | 2025 | Initial GCP documentation |
