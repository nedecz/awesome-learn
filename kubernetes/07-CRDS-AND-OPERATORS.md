# CRDs and Operators

## Table of Contents

1. [Overview](#overview)
2. [Custom Resource Definitions (CRDs)](#custom-resource-definitions-crds)
   - [What Are CRDs](#what-are-crds)
   - [CRD Manifest Structure](#crd-manifest-structure)
   - [Creating and Registering CRDs](#creating-and-registering-crds)
   - [Custom Resource Instances](#custom-resource-instances)
   - [CRD Versioning](#crd-versioning)
   - [CRD Validation](#crd-validation)
   - [Subresources](#subresources)
3. [Controllers](#controllers)
   - [Controller Concept and Reconciliation Loop](#controller-concept-and-reconciliation-loop)
   - [Watch, Informers, Work Queues](#watch-informers-work-queues)
   - [Building a Custom Controller](#building-a-custom-controller)
   - [Controller-Runtime Example](#controller-runtime-example)
   - [Error Handling and Requeueing](#error-handling-and-requeueing)
4. [Operators](#operators)
   - [What Is an Operator](#what-is-an-operator)
   - [Operator Maturity Model](#operator-maturity-model)
   - [Building Operators with Frameworks](#building-operators-with-frameworks)
   - [Popular Operators](#popular-operators)
5. [Webhooks](#webhooks)
   - [Validating Admission Webhooks](#validating-admission-webhooks)
   - [Mutating Admission Webhooks](#mutating-admission-webhooks)
   - [Conversion Webhooks](#conversion-webhooks)
6. [Testing Operators](#testing-operators)
   - [envtest](#envtest)
   - [Integration Testing](#integration-testing)
7. [Operator Lifecycle Manager (OLM)](#operator-lifecycle-manager-olm)
8. [Best Practices](#best-practices)
9. [Anti-Patterns](#anti-patterns)
10. [Next Steps](#next-steps)

## Overview

This document covers **Custom Resource Definitions (CRDs)** and the **Operator pattern** — the primary mechanisms for extending Kubernetes beyond its built-in resource types. Together they allow platform teams to encode complex, domain-specific operational knowledge directly into the cluster control plane.

### Target Audience

- Platform engineers extending Kubernetes with custom APIs
- Developers building operators to automate application lifecycle management
- SREs managing operators in production clusters
- DevOps engineers evaluating operator frameworks and tooling

### Scope

- CRD authoring, versioning, validation, and subresources
- Controller architecture — informers, work queues, reconciliation
- Operator frameworks — Kubebuilder, Operator SDK, Metacontroller, KUDO
- Admission and conversion webhooks
- Testing strategies, OLM, best practices, and anti-patterns

## Custom Resource Definitions (CRDs)

### What Are CRDs

Kubernetes ships with built-in resources such as Pods, Services, and Deployments. **CRDs let you define entirely new resource types** that the API server manages just like native ones. Once a CRD is registered, users can create, read, update, and delete instances of that custom resource through `kubectl` and the standard Kubernetes API.

```
Built-in Resources                    Custom Resources (via CRDs)
┌────────────────┐                    ┌────────────────────┐
│  Pod           │                    │  Database          │
│  Service       │                    │  KafkaCluster      │
│  Deployment    │   + CRDs ──────►   │  Certificate       │
│  ConfigMap     │                    │  BackupPolicy      │
│  Secret        │                    │  MyApp             │
└────────────────┘                    └────────────────────┘
        ▲                                      ▲
        │          Same API server,             │
        └──────── same RBAC, same kubectl ──────┘
```

**Why CRDs exist:**

- **Extensibility** — add domain-specific resources without modifying Kubernetes source code
- **Declarative API** — users describe desired state; controllers reconcile it
- **Ecosystem integration** — RBAC, audit logging, `kubectl`, and client libraries work automatically
- **No separate API server** — unlike Aggregated APIs, CRDs require no additional infrastructure

### CRD Manifest Structure

A CRD manifest tells the API server about the new resource type, its schema, and how to display it.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com            # must match <plural>.<group>
spec:
  group: example.com                     # API group
  names:
    plural: databases                    # URL path: /apis/example.com/v1/databases
    singular: database                   # used in kubectl output
    kind: Database                       # PascalCase resource kind
    shortNames:
      - db                              # kubectl get db
    categories:
      - all                             # included in kubectl get all
  scope: Namespaced                      # or Cluster
  versions:
    - name: v1alpha1
      served: true                       # version is available via the API
      storage: true                      # version used for etcd storage
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - engine
                - version
              properties:
                engine:
                  type: string
                  enum:
                    - postgres
                    - mysql
                    - redis
                  description: Database engine type.
                version:
                  type: string
                  pattern: '^\d+\.\d+$'
                  description: Engine version (e.g. "15.4").
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 9
                  default: 1
                  description: Number of database replicas.
                storage:
                  type: object
                  properties:
                    size:
                      type: string
                      pattern: '^\d+(Gi|Mi)$'
                      default: "10Gi"
                    storageClass:
                      type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                readyReplicas:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
                      message:
                        type: string
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Version
          type: string
          jsonPath: .spec.version
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}                       # enables /status subresource
        scale:                           # enables /scale subresource
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
```

**Key fields explained:**

| Field | Purpose |
|---|---|
| `group` | API group — forms the URL path `/apis/<group>/<version>` |
| `names` | Plural, singular, kind, and shortNames for the resource |
| `scope` | `Namespaced` or `Cluster` — determines if the resource lives in a namespace |
| `versions` | One or more API versions with independent schemas |
| `schema.openAPIV3Schema` | Structural schema that validates every CR instance |
| `additionalPrinterColumns` | Extra columns shown in `kubectl get` output |
| `subresources` | Enables `/status` and `/scale` endpoints |

### Creating and Registering CRDs

```bash
# Apply the CRD to the cluster
kubectl apply -f database-crd.yaml

# Verify the CRD is established
kubectl get crd databases.example.com

# Check the CRD status conditions
kubectl get crd databases.example.com -o jsonpath='{.status.conditions[*].type}'
# Expected: NamesAccepted Established

# The new resource is now available in the API
kubectl api-resources | grep databases
# databases   db   example.com/v1alpha1   true   Database
```

### Custom Resource Instances

Once the CRD is registered, users create instances (Custom Resources, or CRs) using standard manifests:

```yaml
apiVersion: example.com/v1alpha1
kind: Database
metadata:
  name: orders-db
  namespace: production
spec:
  engine: postgres
  version: "15.4"
  replicas: 3
  storage:
    size: 50Gi
    storageClass: fast-ssd
```

```bash
# Create the custom resource
kubectl apply -f orders-db.yaml

# List custom resources — note the additional printer columns
kubectl get databases -n production
# NAME        ENGINE     VERSION   REPLICAS   PHASE   AGE
# orders-db   postgres   15.4      3                  5s

# Inspect the resource
kubectl describe database orders-db -n production

# Delete the resource
kubectl delete database orders-db -n production
```

### CRD Versioning

Production CRDs evolve over time. Kubernetes supports **multiple served versions** with a conversion strategy.

```
Version Lifecycle
┌──────────┐     ┌──────────┐     ┌──────────┐
│ v1alpha1 │────►│ v1beta1  │────►│    v1    │
│ (early)  │     │ (stable) │     │ (GA)     │
└──────────┘     └──────────┘     └──────────┘
     │                │                │
     │    Conversion Webhook           │
     └────────────────┼────────────────┘
                      ▼
              ┌──────────────┐
              │    etcd       │
              │ (one storage  │
              │  version)     │
              └──────────────┘
```

**Multi-version CRD snippet:**

```yaml
spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions:
        - v1
      clientConfig:
        service:
          name: database-webhook
          namespace: database-system
          path: /convert
        caBundle: <base64-encoded-CA>
  versions:
    - name: v1alpha1
      served: true
      storage: false                  # no longer the storage version
      schema: { ... }
    - name: v1beta1
      served: true
      storage: true                   # current storage version
      schema: { ... }
```

**Versioning rules:**

- Exactly **one version** must be the `storage` version at any time
- Old versions remain `served` until all clients migrate
- A **conversion webhook** translates between versions at request time
- Use `None` conversion strategy only when schemas are identical across versions

### CRD Validation

#### OpenAPI v3 Schema

Every CRD **must** include a structural OpenAPI v3 schema (required since Kubernetes 1.16). The schema validates every CR on create and update.

Common validation keywords:

| Keyword | Example | Purpose |
|---|---|---|
| `type` | `string`, `integer`, `object` | Field type |
| `required` | `[engine, version]` | Mandatory fields |
| `enum` | `[postgres, mysql]` | Allowed values |
| `pattern` | `^\d+\.\d+$` | Regex validation |
| `minimum` / `maximum` | `1` / `9` | Numeric bounds |
| `default` | `1` | Default value applied server-side |
| `format` | `date-time` | Well-known string formats |
| `maxLength` / `minLength` | `63` / `1` | String length constraints |

#### CEL Validation Rules

Kubernetes 1.25+ supports **Common Expression Language (CEL)** rules for cross-field validation that OpenAPI alone cannot express:

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        x-kubernetes-validations:
          - rule: "self.replicas <= self.maxReplicas"
            message: "replicas must not exceed maxReplicas"
          - rule: "self.minReplicas <= self.replicas"
            message: "replicas must be at least minReplicas"
          - rule: "has(self.backup) || self.replicas >= 3"
            message: "single/dual replica setups require a backup configuration"
        properties:
          replicas:
            type: integer
          minReplicas:
            type: integer
          maxReplicas:
            type: integer
          backup:
            type: object
            properties:
              schedule:
                type: string
```

### Subresources

#### Status Subresource

When enabled, the `/status` subresource is updated **separately** from the main resource. This separation ensures that:

- Users update `.spec` (desired state) via the main endpoint
- Controllers update `.status` (observed state) via the `/status` endpoint
- RBAC can be scoped independently for spec vs. status

```bash
# Update only the status (used by controllers)
kubectl proxy &
curl -X PUT http://localhost:8001/apis/example.com/v1alpha1/namespaces/production/databases/orders-db/status \
  -H "Content-Type: application/json" \
  -d '{"apiVersion":"example.com/v1alpha1","kind":"Database","metadata":{"name":"orders-db","namespace":"production"},"status":{"phase":"Running","readyReplicas":3}}'
```

#### Scale Subresource

Enables `kubectl scale` and HPA integration for custom resources:

```bash
# Scale the custom resource
kubectl scale database orders-db --replicas=5 -n production

# HPA can target the custom resource
kubectl autoscale database orders-db --min=2 --max=10 --cpu-percent=80 -n production
```

## Controllers

### Controller Concept and Reconciliation Loop

A **controller** is a control loop that watches the state of your cluster and makes changes to move the **current state** toward the **desired state**. Every built-in Kubernetes resource has a controller. Custom resources need custom controllers.

```
                    Reconciliation Loop
                    ───────────────────
          ┌──────────────────────────────────┐
          │                                  │
          ▼                                  │
   ┌─────────────┐    ┌──────────────┐    ┌──┴──────────┐
   │  Observe     │───►│   Diff       │───►│   Act       │
   │  current     │    │   desired vs │    │   create,   │
   │  state       │    │   current    │    │   update,   │
   └─────────────┘    └──────────────┘    │   delete    │
                                           └─────────────┘
          │                                       │
          │         ┌───────────────┐              │
          └─────────│  API Server   │◄─────────────┘
                    └───────────────┘
```

**Key principles:**

- **Level-triggered, not edge-triggered** — the controller reconciles based on current state, not events
- **Idempotent** — running the same reconciliation twice produces the same result
- **Eventual consistency** — the system converges over time, not instantly

### Watch, Informers, Work Queues

Controllers use the **Informer pattern** to efficiently track resource changes without polling the API server.

```
┌───────────────────────────────────────────────────────┐
│                      Controller                        │
│                                                        │
│  ┌───────────┐    ┌──────────┐    ┌────────────────┐  │
│  │  Informer  │───►│  Local    │    │  Work Queue    │  │
│  │            │    │  Cache    │    │                │  │
│  │  - List    │    │  (Store)  │    │  - Dedup       │  │
│  │  - Watch   │    │          │    │  - Rate limit  │  │
│  │            │    └──────────┘    │  - Retry       │  │
│  │            │         │         │                │  │
│  │            │    ┌────▼─────┐   └───────┬────────┘  │
│  │            │───►│ Event     │           │           │
│  │            │    │ Handlers  │──enqueue──┘           │
│  └───────────┘    └──────────┘                        │
│                                        │              │
│                                 ┌──────▼───────┐      │
│                                 │  Reconciler   │      │
│                                 │  (your logic) │      │
│                                 └──────────────┘      │
└───────────────────────────────────────────────────────┘
         ▲                                │
         │           API Server           │
         └────────────────────────────────┘
```

| Component | Role |
|---|---|
| **Informer** | Opens a long-lived watch stream against the API server and maintains a local cache |
| **Local Cache (Store)** | In-memory copy of all watched resources — reads are served from cache, not the API |
| **Event Handlers** | Callbacks (`OnAdd`, `OnUpdate`, `OnDelete`) that enqueue object keys |
| **Work Queue** | Deduplicated, rate-limited queue — ensures one reconciliation per object at a time |
| **Reconciler** | Your business logic — reads desired state, computes diff, takes action |

### Building a Custom Controller

The reconciliation pattern follows this structure:

1. **Fetch** the primary resource (the CR) from the cache
2. **Check** if it is being deleted (handle finalizers)
3. **Read** the current state of owned/dependent resources
4. **Compare** desired vs. current state
5. **Act** — create, update, or delete dependent resources
6. **Update status** on the primary resource
7. **Return** — requeue if needed, or signal success

### Controller-Runtime Example

The [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) library provides the standard building blocks. Below is a minimal Go reconciler for the `Database` CRD:

```go
package controller

import (
	"context"
	"fmt"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	examplev1 "example.com/database-operator/api/v1alpha1"
)

// DatabaseReconciler reconciles a Database object.
type DatabaseReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 1. Fetch the Database CR
	var db examplev1.Database
	if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
		if errors.IsNotFound(err) {
			logger.Info("Database resource deleted, skipping")
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	// 2. Build the desired StatefulSet
	desired := r.statefulSetForDatabase(&db)

	// 3. Check if StatefulSet already exists
	var existing appsv1.StatefulSet
	err := r.Get(ctx, client.ObjectKeyFromObject(desired), &existing)
	if err != nil && errors.IsNotFound(err) {
		logger.Info("Creating StatefulSet", "name", desired.Name)
		if err := r.Create(ctx, desired); err != nil {
			return ctrl.Result{}, err
		}
	} else if err != nil {
		return ctrl.Result{}, err
	} else {
		// 4. Update if spec has drifted
		if *existing.Spec.Replicas != int32(db.Spec.Replicas) {
			existing.Spec.Replicas = int32Ptr(int32(db.Spec.Replicas))
			if err := r.Update(ctx, &existing); err != nil {
				return ctrl.Result{}, err
			}
		}
	}

	// 5. Update Database status
	db.Status.Phase = "Running"
	db.Status.ReadyReplicas = existing.Status.ReadyReplicas
	if err := r.Status().Update(ctx, &db); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&examplev1.Database{}).           // watch Database CRs
		Owns(&appsv1.StatefulSet{}).          // watch owned StatefulSets
		Complete(r)
}

func (r *DatabaseReconciler) statefulSetForDatabase(db *examplev1.Database) *appsv1.StatefulSet {
	labels := map[string]string{"app": db.Name, "managed-by": "database-operator"}
	replicas := int32(db.Spec.Replicas)
	return &appsv1.StatefulSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%s-db", db.Name),
			Namespace: db.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(db, examplev1.GroupVersion.WithKind("Database")),
			},
		},
		Spec: appsv1.StatefulSetSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{MatchLabels: labels},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{
						Name:  "database",
						Image: fmt.Sprintf("%s:%s", db.Spec.Engine, db.Spec.Version),
					}},
				},
			},
		},
	}
}

func int32Ptr(i int32) *int32 { return &i }
```

### Error Handling and Requeueing

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// Returning an error triggers exponential backoff requeue
	if err := r.doSomething(); err != nil {
		return ctrl.Result{}, err              // requeue with backoff
	}

	// Explicit requeue after a delay (e.g., waiting for external resource)
	if !ready {
		return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
	}

	// Immediate requeue (use sparingly)
	if needsAnotherPass {
		return ctrl.Result{Requeue: true}, nil
	}

	// Success — no requeue
	return ctrl.Result{}, nil
}
```

| Return Value | Behavior |
|---|---|
| `Result{}, nil` | Success — no requeue unless a new event arrives |
| `Result{}, err` | Error — requeue with exponential backoff |
| `Result{Requeue: true}, nil` | Immediate requeue |
| `Result{RequeueAfter: d}, nil` | Requeue after duration `d` |

## Operators

### What Is an Operator

An **Operator** is a Kubernetes-native application that combines a **CRD** with a **custom controller** to manage a complex workload. The term comes from the idea of encoding the knowledge of a **human operator** — someone who knows how to deploy, configure, scale, back up, and recover an application — into software that runs inside the cluster.

```
┌─────────────────────────────────────────┐
│               Operator                   │
│                                          │
│   ┌────────────┐    ┌────────────────┐  │
│   │    CRD      │    │   Controller    │  │
│   │ (the API)   │    │ (the brains)   │  │
│   │             │    │                │  │
│   │ Database    │◄──►│ Reconciles     │  │
│   │ spec/status │    │ desired state  │  │
│   └────────────┘    └────────┬───────┘  │
│                              │           │
│                    ┌─────────▼────────┐  │
│                    │  Managed          │  │
│                    │  Resources        │  │
│                    │  - StatefulSet    │  │
│                    │  - Service        │  │
│                    │  - ConfigMap      │  │
│                    │  - PVC            │  │
│                    │  - Secret         │  │
│                    └──────────────────┘  │
└─────────────────────────────────────────┘
```

**What Operators handle that Helm cannot:**

- **Day-2 operations** — backups, failover, upgrades, scaling decisions
- **Self-healing** — detecting and recovering from failures automatically
- **Application-aware scaling** — scaling based on domain-specific metrics
- **Ordered lifecycle management** — complex multi-step provisioning sequences

### Operator Maturity Model

The [Operator Framework](https://operatorframework.io/) defines five capability levels:

| Level | Name | Capabilities |
|---|---|---|
| **1** | Basic Install | Automated provisioning via a CR |
| **2** | Seamless Upgrades | Patch and minor version upgrades managed by the operator |
| **3** | Full Lifecycle | Backup, restore, failure recovery |
| **4** | Deep Insights | Metrics, alerts, log processing, workload analysis |
| **5** | Auto Pilot | Horizontal/vertical scaling, auto-tuning, anomaly detection |

### Building Operators with Frameworks

#### Kubebuilder (Go)

[Kubebuilder](https://book.kubebuilder.io/) is the upstream project for scaffolding Go-based operators.

```bash
# Initialize a new operator project
kubebuilder init --domain example.com --repo example.com/database-operator

# Create a new API (CRD + Controller)
kubebuilder create api --group apps --version v1alpha1 --kind Database

# Generate CRD manifests, RBAC, and webhook configurations
make manifests

# Run locally against a cluster
make run

# Build and push the operator image
make docker-build docker-push IMG=registry.example.com/database-operator:v0.1.0

# Deploy to cluster
make deploy IMG=registry.example.com/database-operator:v0.1.0
```

**Project layout:**

```
database-operator/
├── api/
│   └── v1alpha1/
│       ├── database_types.go         # CRD Go types
│       ├── groupversion_info.go      # GVK registration
│       └── zz_generated.deepcopy.go  # generated DeepCopy methods
├── config/
│   ├── crd/                          # generated CRD manifests
│   ├── manager/                      # operator Deployment manifest
│   ├── rbac/                         # RBAC rules
│   └── samples/                      # example CRs
├── internal/
│   └── controller/
│       ├── database_controller.go    # reconciliation logic
│       └── database_controller_test.go
├── cmd/
│   └── main.go                       # entrypoint
├── Dockerfile
├── Makefile
└── PROJECT
```

#### Operator SDK (Go, Ansible, Helm)

The [Operator SDK](https://sdk.operatorframework.io/) builds on Kubebuilder and adds support for **Ansible** and **Helm** based operators — useful when teams do not write Go.

```bash
# Helm-based operator (no Go code needed)
operator-sdk init --plugins helm --domain example.com
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --helm-chart memcached

# Ansible-based operator
operator-sdk init --plugins ansible --domain example.com
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --generate-role

# Go-based operator (same as Kubebuilder)
operator-sdk init --domain example.com --repo example.com/memcached-operator
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
```

| Approach | Best For | Trade-offs |
|---|---|---|
| **Go** | Full control, complex Day-2 logic | Requires Go expertise |
| **Ansible** | Teams with Ansible experience, config-management tasks | Limited performance at scale |
| **Helm** | Wrapping existing Helm charts with an operator lifecycle | Limited to install/upgrade/delete |

#### Metacontroller

[Metacontroller](https://metacontroller.github.io/metacontroller/) lets you write controllers as **simple webhooks** in any language. You provide a function that receives the parent resource and returns the desired child resources.

```yaml
apiVersion: metacontroller.k8s.io/v1alpha1
kind: CompositeController
metadata:
  name: database-controller
spec:
  parentResource:
    apiVersion: example.com/v1alpha1
    resource: databases
  childResources:
    - apiVersion: apps/v1
      resource: statefulsets
      updateStrategy:
        method: InPlace
  hooks:
    sync:
      webhook:
        url: http://database-hook.default:8080/sync
```

#### KUDO

[KUDO](https://kudo.dev/) (Kubernetes Universal Declarative Operator) allows building operators **purely with YAML** — no programming required. Operational steps are defined as plans with phases and steps.

```yaml
apiVersion: kudo.dev/v1beta1
kind: OperatorVersion
metadata:
  name: database-0.1.0
spec:
  plans:
    deploy:
      phases:
        - name: main
          strategy: serial
          steps:
            - name: statefulset
              tasks:
                - statefulset
            - name: service
              tasks:
                - service
    backup:
      phases:
        - name: snapshot
          strategy: serial
          steps:
            - name: create-snapshot
              tasks:
                - backup-job
```

### Popular Operators

| Operator | Manages | Maturity |
|---|---|---|
| **Prometheus Operator** | Prometheus, Alertmanager, ServiceMonitors | Level 5 |
| **cert-manager** | TLS certificates, ACME issuers, certificate renewal | Level 3 |
| **Strimzi** | Apache Kafka clusters, topics, users, connectors | Level 4 |
| **CloudNativePG** | PostgreSQL clusters, backups, failover, WAL archiving | Level 4 |
| **Rook-Ceph** | Ceph storage clusters, pools, filesystems | Level 5 |
| **Crossplane** | Cloud infrastructure (AWS, GCP, Azure) as CRDs | Level 3 |

## Webhooks

Webhooks extend the API server's request processing pipeline.

```
API Request Flow with Webhooks
──────────────────────────────

  kubectl apply ──► API Server
                      │
                      ▼
               ┌──────────────┐
               │ Authentication│
               │ Authorization │
               └──────┬───────┘
                      ▼
               ┌──────────────┐
               │   Mutating    │  ◄── modify the request
               │   Admission   │      (inject sidecars, set defaults)
               └──────┬───────┘
                      ▼
               ┌──────────────┐
               │  Object       │
               │  Schema       │
               │  Validation   │
               └──────┬───────┘
                      ▼
               ┌──────────────┐
               │  Validating   │  ◄── accept or reject
               │  Admission    │      (policy enforcement)
               └──────┬───────┘
                      ▼
                   etcd
```

### Validating Admission Webhooks

Validating webhooks **accept or reject** requests. They cannot modify the object.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: database-validating
webhooks:
  - name: validate.database.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: database-webhook
        namespace: database-system
        path: /validate-database
      caBundle: <base64-encoded-CA>
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["example.com"]
        apiVersions: ["v1alpha1"]
        resources: ["databases"]
    failurePolicy: Fail                  # Fail closed — reject if webhook is unavailable
    matchPolicy: Equivalent
```

### Mutating Admission Webhooks

Mutating webhooks **modify** the request object before validation. Common uses include injecting sidecar containers, setting default values, and adding labels.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: database-mutating
webhooks:
  - name: mutate.database.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: database-webhook
        namespace: database-system
        path: /mutate-database
      caBundle: <base64-encoded-CA>
    rules:
      - operations: ["CREATE"]
        apiGroups: ["example.com"]
        apiVersions: ["v1alpha1"]
        resources: ["databases"]
    failurePolicy: Fail
    reinvocationPolicy: IfNeeded        # re-run if another mutating webhook changes the object
```

### Conversion Webhooks

Conversion webhooks translate custom resources **between API versions** at request time. They are configured in the CRD spec (see [CRD Versioning](#crd-versioning)) and are called whenever the API server needs to serve or store a version different from what the client sent.

**Conversion flow:**

```
Client sends v1alpha1 ──► API Server ──► Conversion Webhook
                                              │
                                    converts to v1beta1
                                    (storage version)
                                              │
                                              ▼
                                           etcd
```

## Testing Operators

### envtest

The `envtest` package from controller-runtime spins up a **real API server and etcd** (no kubelet, no scheduler) for fast integration tests without a full cluster.

```go
package controller_test

import (
	"context"
	"testing"
	"time"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	"sigs.k8s.io/controller-runtime/pkg/envtest"
	"sigs.k8s.io/controller-runtime/pkg/client"

	examplev1 "example.com/database-operator/api/v1alpha1"
)

var testEnv *envtest.Environment
var k8sClient client.Client

func TestControllers(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
	testEnv = &envtest.Environment{
		CRDDirectoryPaths: []string{"../config/crd/bases"},
	}
	cfg, err := testEnv.Start()
	Expect(err).NotTo(HaveOccurred())

	k8sClient, err = client.New(cfg, client.Options{})
	Expect(err).NotTo(HaveOccurred())
})

var _ = AfterSuite(func() {
	Expect(testEnv.Stop()).To(Succeed())
})

var _ = Describe("Database Controller", func() {
	It("should create a StatefulSet for a new Database", func() {
		ctx := context.Background()
		db := &examplev1.Database{
			ObjectMeta: metav1.ObjectMeta{
				Name:      "test-db",
				Namespace: "default",
			},
			Spec: examplev1.DatabaseSpec{
				Engine:   "postgres",
				Version:  "15.4",
				Replicas: 1,
			},
		}
		Expect(k8sClient.Create(ctx, db)).To(Succeed())

		// Eventually the StatefulSet should appear
		Eventually(func() error {
			var sts appsv1.StatefulSet
			return k8sClient.Get(ctx, client.ObjectKey{
				Name:      "test-db-db",
				Namespace: "default",
			}, &sts)
		}, 10*time.Second, 250*time.Millisecond).Should(Succeed())
	})
})
```

### Integration Testing

For full end-to-end testing, use a real cluster (Kind, Minikube, or a CI ephemeral cluster):

```bash
# Create a Kind cluster for testing
kind create cluster --name operator-test

# Install CRDs
make install

# Run the operator locally
make run &

# Apply test resources
kubectl apply -f config/samples/

# Verify
kubectl wait --for=condition=Ready database/test-db --timeout=60s

# Clean up
kind delete cluster --name operator-test
```

**Testing strategy summary:**

| Layer | Tool | Scope | Speed |
|---|---|---|---|
| Unit | Go `testing` | Pure functions, utilities | Milliseconds |
| Integration | `envtest` | Controller + API server | Seconds |
| E2E | Kind / Minikube | Full cluster behavior | Minutes |

## Operator Lifecycle Manager (OLM)

OLM manages the **lifecycle of operators themselves** — install, upgrade, and manage dependencies between operators in a cluster.

```
OLM Architecture
─────────────────
┌─────────────────────────────────────────────┐
│                 OLM                          │
│                                              │
│  ┌──────────────┐    ┌───────────────────┐  │
│  │  Catalog      │    │  Subscription      │  │
│  │  Source        │    │                   │  │
│  │ (available    │    │  (desired operator │  │
│  │  operators)   │    │   + update channel)│  │
│  └──────┬───────┘    └───────┬───────────┘  │
│         │                    │               │
│         ▼                    ▼               │
│  ┌──────────────────────────────────────┐   │
│  │        InstallPlan                    │   │
│  │  (resolved dependencies + approval)   │   │
│  └──────────────┬───────────────────────┘   │
│                 ▼                            │
│  ┌──────────────────────────────────────┐   │
│  │     ClusterServiceVersion (CSV)       │   │
│  │  (operator metadata, RBAC, CRDs)     │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Key OLM resources:**

| Resource | Purpose |
|---|---|
| `CatalogSource` | Points to a repository of operator bundles |
| `Subscription` | Declares which operator to install and which update channel to follow |
| `InstallPlan` | The resolved set of resources to install, subject to approval |
| `ClusterServiceVersion` | The deployable unit — contains operator metadata, owned CRDs, and RBAC |
| `OperatorGroup` | Scopes the operator's watch to specific namespaces |

## Best Practices

**CRD Design:**

- Use **clear, domain-specific naming** — the CRD API should read like a product interface, not infrastructure
- **Version from day one** — start with `v1alpha1` and plan for promotion
- Provide **sensible defaults** using OpenAPI `default` or mutating webhooks
- Keep the **spec/status separation** strict — users own spec, controllers own status
- Use **printer columns** so `kubectl get` is immediately useful

**Controller Design:**

- Make reconciliation **idempotent** — safe to run at any time, in any order
- Use **owner references** so garbage collection cleans up dependent resources
- **Do not use finalizers unless necessary** — they block deletion and complicate debugging
- When finalizers are needed, always handle the deletion path first in your reconciler
- Use **rate-limited requeue** instead of tight loops when waiting for external state
- **One controller per CRD** — avoid mixing concerns in a single reconciler

**Operational:**

- Run operators with **least-privilege RBAC** — only the permissions the controller actually needs
- Set **resource requests and limits** on the operator Deployment
- Expose **metrics** (Prometheus) and **structured logs** for observability
- Use **leader election** for HA operator deployments
- Ship **end-to-end tests** that run in CI against a Kind cluster

## Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|---|---|---|
| **God Operator** | Single operator managing dozens of unrelated CRDs | One operator per domain boundary |
| **Polling the API server** | Direct `List` calls in a loop instead of using informers | Use informers and work queues |
| **Non-idempotent reconciliation** | Creates duplicate resources on every reconciliation | Check before creating; use server-side apply |
| **Ignoring status** | No status subresource; users cannot observe progress | Always implement a status subresource with conditions |
| **Hard-coded namespace** | Operator only works in one namespace | Use the CR namespace or make it configurable |
| **Missing owner references** | Orphaned resources after CR deletion | Set owner references on all created resources |
| **Blocking reconciliation** | Synchronous HTTP calls or long computations in the reconciler | Use async patterns with requeue; offload heavy work |
| **Unversioned CRDs** | Breaking changes to the schema without a version bump | Use versioned APIs with conversion webhooks |
| **Skipping validation** | No schema or weak schema allows invalid CRs | Use OpenAPI validation + CEL rules + validating webhooks |
| **No tests** | Manual testing only; regressions go undetected | Unit + envtest + E2E tests in CI |

## Next Steps

- Review [01 — Services](01-SERVICES.md) for Kubernetes networking fundamentals and service discovery
- Explore the [Kubebuilder Book](https://book.kubebuilder.io/) for a full walkthrough of building an operator
- Try the [Operator SDK tutorials](https://sdk.operatorframework.io/docs/) for Ansible and Helm-based operators
- Review the [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) for designing production-grade CRDs
