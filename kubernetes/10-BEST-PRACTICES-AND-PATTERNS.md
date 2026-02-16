# Best Practices and Patterns

## Table of Contents

1. [Overview](#overview)
2. [Resource Management](#resource-management)
   - [Requests and Limits](#requests-and-limits)
   - [Quality of Service Classes](#quality-of-service-classes)
   - [ResourceQuotas and LimitRanges](#resourcequotas-and-limitranges)
   - [Autoscaling](#autoscaling)
   - [Right-Sizing with Metrics](#right-sizing-with-metrics)
3. [Pod Design Patterns](#pod-design-patterns)
   - [Sidecar Pattern](#sidecar-pattern)
   - [Ambassador Pattern](#ambassador-pattern)
   - [Adapter Pattern](#adapter-pattern)
   - [Init Container Pattern](#init-container-pattern)
4. [Deployment Patterns](#deployment-patterns)
   - [Rolling Updates](#rolling-updates)
   - [Blue-Green Deployments](#blue-green-deployments)
   - [Canary Deployments](#canary-deployments)
   - [GitOps](#gitops)
5. [Configuration Best Practices](#configuration-best-practices)
   - [Externalizing Configuration](#externalizing-configuration)
   - [Secret Management](#secret-management)
   - [Kustomize Overlays](#kustomize-overlays)
   - [Immutable ConfigMaps and Secrets](#immutable-configmaps-and-secrets)
6. [Security Best Practices](#security-best-practices)
   - [Pod Security Standards](#pod-security-standards)
   - [Pod Security Admission](#pod-security-admission)
   - [SecurityContext](#securitycontext)
   - [Network Policies](#network-policies)
   - [RBAC Least Privilege](#rbac-least-privilege)
   - [Image Security](#image-security)
   - [Service Account Hardening](#service-account-hardening)
7. [Reliability Patterns](#reliability-patterns)
   - [Health Checks](#health-checks)
   - [Pod Disruption Budgets](#pod-disruption-budgets)
   - [Topology Spread Constraints](#topology-spread-constraints)
   - [Pod Anti-Affinity](#pod-anti-affinity)
   - [Graceful Shutdown](#graceful-shutdown)
8. [Observability](#observability)
   - [Structured Logging](#structured-logging)
   - [Prometheus Metrics](#prometheus-metrics)
   - [Label Strategy for Monitoring](#label-strategy-for-monitoring)
   - [Distributed Tracing](#distributed-tracing)
9. [Namespace Strategy](#namespace-strategy)
   - [Organizational Patterns](#organizational-patterns)
   - [Resource Isolation and RBAC](#resource-isolation-and-rbac)
   - [Cross-Namespace Communication](#cross-namespace-communication)
10. [Helm Best Practices](#helm-best-practices)
    - [Chart Structure](#chart-structure)
    - [Values File Management](#values-file-management)
    - [Chart Testing](#chart-testing)
11. [CI/CD Best Practices](#cicd-best-practices)
    - [Image Tagging](#image-tagging)
    - [Manifest Validation](#manifest-validation)
    - [Progressive Rollouts](#progressive-rollouts)
12. [Cost Optimization](#cost-optimization)
    - [Spot and Preemptible Nodes](#spot-and-preemptible-nodes)
    - [Cluster Autoscaling](#cluster-autoscaling)
    - [Namespace Budgets](#namespace-budgets)
13. [Summary](#summary)
14. [Next Steps](#next-steps)

---

## Overview

Running workloads on Kubernetes is straightforward — running them **well** is not. This document captures battle-tested best practices and design patterns that help teams build workloads that are reliable, secure, cost-efficient, and operationally mature.

### Target Audience

- Developers deploying applications to Kubernetes
- Platform engineers building internal developer platforms
- SREs responsible for production reliability and cost management

### Scope

- Resource management, autoscaling, and right-sizing
- Multi-container Pod patterns (sidecar, ambassador, adapter, init)
- Deployment strategies (rolling, blue-green, canary, GitOps)
- Security hardening, RBAC, and network policies
- Reliability patterns including health checks and disruption budgets
- Observability, CI/CD, Helm, namespaces, and cost optimization

### Why Best Practices Matter

```
  Without Best Practices              With Best Practices
  ┌─────────────────────┐            ┌─────────────────────┐
  │  No resource limits │            │  Requests + Limits  │
  │  No health checks   │            │  Liveness/Readiness │
  │  :latest tags       │            │  Immutable tags     │
  │  Root containers    │            │  Non-root, read-only│
  │  No network policy  │            │  Default deny       │
  │  Manual deploys     │            │  GitOps pipelines   │
  └────────┬────────────┘            └────────┬────────────┘
           │                                  │
           ▼                                  ▼
  ╔═══════════════════╗              ╔═══════════════════╗
  ║  Outages, drift,  ║              ║  Stable, secure,  ║
  ║  security holes,  ║              ║  auditable, cost-  ║
  ║  runaway costs    ║              ║  efficient cluster ║
  ╚═══════════════════╝              ╚═══════════════════╝
```

---

## Resource Management

### Requests and Limits

Every container should declare **resource requests** (guaranteed minimum) and **resource limits** (enforced maximum). Without them, the scheduler cannot make informed placement decisions and Pods risk being evicted or starving neighbors.

- ✅ Always set both `requests` and `limits` for CPU and memory
- ✅ Set requests based on observed usage under normal load
- ✅ Set limits to handle reasonable spikes without allowing runaway consumption
- ❌ Do not set limits without requests — this creates unpredictable scheduling
- ❌ Do not omit resources entirely — Pods become BestEffort and are evicted first

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
          resources:
            requests:
              cpu: 250m        # 0.25 CPU cores — guaranteed minimum
              memory: 256Mi    # 256 MiB — guaranteed minimum
            limits:
              cpu: "1"         # 1 CPU core — hard ceiling
              memory: 512Mi    # 512 MiB — OOMKilled if exceeded
          ports:
            - containerPort: 8080
```

### Quality of Service Classes

Kubernetes assigns a **QoS class** to every Pod based on how requests and limits are configured. The QoS class determines eviction priority when a node is under memory pressure.

| QoS Class    | Condition                                           | Eviction Priority | Best For                  |
|--------------|-----------------------------------------------------|-------------------|---------------------------|
| Guaranteed   | Every container has equal `requests` and `limits`   | Last (lowest)     | Critical production Pods  |
| Burstable    | At least one container has `requests` < `limits`    | Middle            | General workloads         |
| BestEffort   | No container specifies `requests` or `limits`       | First (highest)   | Batch jobs, dev/test only |

```
Memory Pressure Eviction Order:

  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ BestEffort  │ ──►│  Burstable  │ ──►│ Guaranteed  │
  │ (evict 1st) │    │ (evict 2nd) │    │ (evict last)│
  └─────────────┘    └─────────────┘    └─────────────┘
```

**Guaranteed QoS example** — requests equal limits:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### ResourceQuotas and LimitRanges

**ResourceQuotas** cap total resource consumption per namespace, preventing any single team from monopolizing cluster capacity.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"         # Total CPU requests across all Pods
    requests.memory: 20Gi     # Total memory requests
    limits.cpu: "20"           # Total CPU limits
    limits.memory: 40Gi       # Total memory limits
    pods: "50"                 # Maximum number of Pods
    services: "20"             # Maximum number of Services
    persistentvolumeclaims: "10"
```

**LimitRanges** set default and maximum resource values for individual containers, ensuring every Pod has reasonable resource boundaries even when developers forget to set them.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-alpha
spec:
  limits:
    - type: Container
      default:               # Applied if no limits are specified
        cpu: 500m
        memory: 256Mi
      defaultRequest:        # Applied if no requests are specified
        cpu: 100m
        memory: 128Mi
      max:                   # Maximum allowed per container
        cpu: "2"
        memory: 2Gi
      min:                   # Minimum allowed per container
        cpu: 50m
        memory: 64Mi
```

### Autoscaling

**Horizontal Pod Autoscaler (HPA)** adjusts the number of replicas based on observed metrics. This is the primary autoscaling mechanism for stateless workloads.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10                      # Scale down 10% at a time
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately
      policies:
        - type: Percent
          value: 100                     # Double capacity if needed
          periodSeconds: 60
        - type: Pods
          value: 4                       # Or add 4 Pods at a time
          periodSeconds: 60
      selectPolicy: Max                  # Use whichever policy adds more
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70         # Target 70% CPU utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**Vertical Pod Autoscaler (VPA)** adjusts CPU and memory requests on individual Pods. VPA is useful for workloads where horizontal scaling is impractical (databases, stateful services). VPA and HPA should not target the same metric on the same workload.

```
Autoscaling Decision Tree:

  Is the workload stateless?
  ├── Yes → Use HPA (scale replicas)
  │         Target CPU or custom metrics
  └── No  → Can it be sharded?
            ├── Yes → Use HPA with custom metrics
            └── No  → Use VPA (resize Pods)
                      Set updateMode: "Auto" or "Off" (recommendation only)
```

### Right-Sizing with Metrics

Install **metrics-server** to enable `kubectl top` and HPA:

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Observe actual usage
kubectl top pods -n production --sort-by=cpu
kubectl top nodes

# Compare requests vs actual — look for over-provisioning
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
```

- ✅ Review metrics weekly and adjust requests to match P95 usage
- ✅ Use VPA in recommendation mode to get sizing suggestions
- ❌ Do not set requests far above actual usage — this wastes cluster capacity

---

## Pod Design Patterns

### Sidecar Pattern

A **sidecar** container runs alongside the main application container in the same Pod, extending functionality without modifying the application. Common use cases include logging agents, service mesh proxies, and monitoring collectors.

```
┌─────────────────────────────────┐
│              Pod                │
│  ┌──────────┐  ┌─────────────┐ │
│  │   App    │  │   Sidecar   │ │
│  │ Container│  │  (logging   │ │
│  │          │──│   agent)    │ │
│  │  :8080   │  │             │ │
│  └──────────┘  └─────────────┘ │
│       │  shared volume  │      │
│       └────────┬────────┘      │
│          /var/log/app           │
└─────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging-sidecar
  labels:
    app: web
spec:
  containers:
    - name: app
      image: myregistry/web-app:v2.1.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
    - name: log-agent
      image: fluent/fluent-bit:2.2
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
          readOnly: true
      env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
  volumes:
    - name: log-volume
      emptyDir: {}
```

### Ambassador Pattern

An **ambassador** container proxies network traffic from the application to external services, abstracting connection details like credentials, endpoints, and protocols.

```
┌──────────────────────────────────┐
│               Pod                │
│  ┌──────────┐  ┌──────────────┐ │        ┌──────────────┐
│  │   App    │  │  Ambassador  │ │        │   External   │
│  │          │─►│   (proxy)    │─┼───────►│   Database   │
│  │ localhost│  │  localhost   │ │        │  (cloud SQL) │
│  │  :5432   │  │   :5432     │ │        └──────────────┘
│  └──────────┘  └──────────────┘ │
└──────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
  labels:
    app: backend
spec:
  containers:
    - name: app
      image: myregistry/backend:v3.0.1
      env:
        - name: DB_HOST
          value: "localhost"       # App connects to localhost
        - name: DB_PORT
          value: "5432"
    - name: cloudsql-proxy
      image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
      args:
        - "--port=5432"
        - "my-project:us-central1:my-database"
      securityContext:
        runAsNonRoot: true
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
```

### Adapter Pattern

An **adapter** container transforms or standardizes the output of the main container. This is useful when aggregation systems expect a uniform format but applications produce different output formats.

```
┌───────────────────────────────────┐
│                Pod                │
│  ┌──────────┐  ┌───────────────┐ │     ┌────────────────┐
│  │   App    │  │   Adapter     │ │     │   Monitoring   │
│  │  (custom │─►│  (transforms  │─┼────►│   System       │
│  │  metrics)│  │  to Prometheus│ │     │  (Prometheus)  │
│  │  :9999   │  │  format)      │ │     └────────────────┘
│  └──────────┘  │  :9090        │ │
│                └───────────────┘ │
└───────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
  labels:
    app: legacy-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
spec:
  containers:
    - name: legacy-app
      image: myregistry/legacy-service:v1.8.0
      ports:
        - containerPort: 9999
    - name: prometheus-adapter
      image: myregistry/metrics-adapter:v1.0.0
      ports:
        - containerPort: 9090
      env:
        - name: SOURCE_URL
          value: "http://localhost:9999/stats"
        - name: OUTPUT_FORMAT
          value: "prometheus"
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
```

### Init Container Pattern

**Init containers** run to completion before the main containers start. They are ideal for pre-flight tasks such as database migrations, configuration fetching, or waiting for dependencies.

```
  Pod Startup Sequence:

  ┌────────────┐   ┌────────────┐   ┌────────────────────────┐
  │  Init #1   │──►│  Init #2   │──►│  Main Containers       │
  │ (wait for  │   │ (run DB    │   │  (app + sidecars)      │
  │  database) │   │  migration)│   │  start in parallel     │
  └────────────┘   └────────────┘   └────────────────────────┘
     sequential       sequential          parallel
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  labels:
    app: api-server
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          until nc -z postgres.database.svc.cluster.local 5432; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready"
    - name: run-migrations
      image: myregistry/api-server:v1.4.2
      command: ["./migrate", "--direction", "up"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
  containers:
    - name: api
      image: myregistry/api-server:v1.4.2
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 512Mi
```

---

## Deployment Patterns

### Rolling Updates

The default Kubernetes deployment strategy. Pods are replaced incrementally, ensuring zero downtime when configured correctly.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2            # At most 2 extra Pods during update (6+2=8)
      maxUnavailable: 1      # At most 1 Pod unavailable during update
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: v2.0.0
    spec:
      containers:
        - name: web
          image: myregistry/web-app:v2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

```
Rolling Update Sequence (replicas=4, maxSurge=1, maxUnavailable=1):

  Step 1: v1 v1 v1 v1        ← all running v1
  Step 2: v1 v1 v1 -- v2     ← terminate 1 old, start 1 new
  Step 3: v1 v1 -- v2 v2     ← new Pod ready, continue
  Step 4: v1 -- v2 v2 v2     ← repeat
  Step 5: v2 v2 v2 v2        ← all running v2
```

- ✅ Always define **readinessProbes** — rolling updates rely on readiness to proceed
- ✅ Set `maxUnavailable: 0` for zero-downtime updates (requires `maxSurge >= 1`)
- ✅ Use `kubectl rollout undo` to revert a bad release quickly

### Blue-Green Deployments

Run two identical environments (blue and green) simultaneously. Traffic is switched atomically by updating the Service selector. This provides instant rollback — just switch the selector back.

```
  Blue-Green Deployment:

  ┌──────────────┐
  │   Service    │
  │  selector:   │──── Points to ONE deployment at a time
  │  version: v2 │
  └──────┬───────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼ (idle)
  ┌──────┐  ┌──────┐
  │Green │  │ Blue │
  │(v2)  │  │ (v1) │
  │active│  │stand-│
  │      │  │ by   │
  └──────┘  └──────┘
```

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-app
      version: v1
  template:
    metadata:
      labels:
        app: web-app
        version: v1
    spec:
      containers:
        - name: web
          image: myregistry/web-app:v1.9.0
          ports:
            - containerPort: 8080
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
  namespace: production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-app
      version: v2
  template:
    metadata:
      labels:
        app: web-app
        version: v2
    spec:
      containers:
        - name: web
          image: myregistry/web-app:v2.0.0
          ports:
            - containerPort: 8080
---
# Service — switch traffic by changing the version selector
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
spec:
  selector:
    app: web-app
    version: v2          # ← Change to "v1" to roll back
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Switch from blue (v1) to green (v2)
kubectl patch service web-app -n production \
  -p '{"spec":{"selector":{"version":"v2"}}}'

# Roll back to blue (v1)
kubectl patch service web-app -n production \
  -p '{"spec":{"selector":{"version":"v1"}}}'
```

### Canary Deployments

Route a small percentage of traffic to the new version to validate it in production before full rollout. This can be achieved with native Kubernetes using replica ratios or with a service mesh for fine-grained traffic splitting.

```
  Canary Deployment (10% traffic):

  ┌──────────────────┐
  │     Service      │
  │  selector:       │
  │    app: web-app  │
  └───────┬──────────┘
          │
    ┌─────┴──────┐
    │            │
    ▼            ▼
  ┌─────────┐  ┌──────┐
  │ Stable  │  │Canary│
  │ (v1)    │  │ (v2) │
  │ 9 pods  │  │1 pod │
  │ 90%     │  │ 10%  │
  └─────────┘  └──────┘
```

With **native Kubernetes**, traffic is split proportionally by replica count — 9 stable + 1 canary = ~10% canary traffic. For precise traffic splitting, use **Istio VirtualService** or **Argo Rollouts**.

### GitOps

**GitOps** treats Git as the single source of truth for cluster state. Changes are applied through pull requests, and a reconciliation controller ensures the cluster matches the desired state in Git.

```
  GitOps Workflow:

  Developer ──► Git Repo ──► ArgoCD / Flux ──► Kubernetes Cluster
                  │               │                    │
              PR + Review    Detect drift         Apply manifests
              Audit trail    Auto-sync             Reconcile
```

**Benefits of GitOps:**

- ✅ Full audit trail — every change is a Git commit
- ✅ Declarative and reproducible — rebuild from Git at any time
- ✅ Pull-based delivery — no direct `kubectl apply` in production
- ✅ Drift detection — controller alerts or auto-corrects configuration drift
- ✅ Multi-cluster consistency — same repo deploys to staging and production

Popular tools: **ArgoCD**, **Flux**, **Rancher Fleet**

---

## Configuration Best Practices

### Externalizing Configuration

Never bake environment-specific configuration into container images. Use **ConfigMaps** to inject configuration as environment variables or mounted files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAGS: |
    {
      "new-checkout": true,
      "dark-mode": false
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
          envFrom:
            - configMapRef:
                name: app-config       # Inject all keys as env vars
          volumeMounts:
            - name: feature-flags
              mountPath: /etc/config
              readOnly: true
      volumes:
        - name: feature-flags
          configMap:
            name: app-config
            items:
              - key: FEATURE_FLAGS
                path: features.json    # Mount specific key as file
```

### Secret Management

Kubernetes Secrets are base64-encoded, **not encrypted** at rest by default. For production, use an external secrets solution.

| Solution                     | How It Works                                     | Best For                     |
|------------------------------|--------------------------------------------------|------------------------------|
| External Secrets Operator    | Syncs secrets from AWS SM, GCP SM, Azure KV      | Cloud-native environments    |
| Sealed Secrets               | Encrypts secrets client-side for safe Git storage | GitOps workflows             |
| HashiCorp Vault              | Centralized secrets engine with dynamic secrets   | Multi-cloud, strict policies |
| SOPS + Age/KMS               | Encrypts YAML values in-place                     | Simple GitOps setups         |

- ✅ Enable **etcd encryption at rest** for Kubernetes Secrets
- ✅ Use an external secrets operator to avoid storing secrets in Git
- ✅ Rotate secrets regularly and use short-lived credentials
- ❌ Never commit plain-text secrets to version control

### Kustomize Overlays

**Kustomize** enables environment-specific configuration without duplicating manifests. Use a base with overlays for each environment.

```
Directory structure:

  k8s/
  ├── base/
  │   ├── kustomization.yaml
  │   ├── deployment.yaml
  │   └── service.yaml
  ├── overlays/
  │   ├── dev/
  │   │   ├── kustomization.yaml
  │   │   └── replica-patch.yaml
  │   ├── staging/
  │   │   ├── kustomization.yaml
  │   │   └── resource-patch.yaml
  │   └── production/
  │       ├── kustomization.yaml
  │       ├── resource-patch.yaml
  │       └── hpa.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - hpa.yaml
namespace: production
patches:
  - path: resource-patch.yaml
    target:
      kind: Deployment
      name: api-server
---
# overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: api
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 1Gi
```

```bash
# Preview the generated manifests
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/production
```

### Immutable ConfigMaps and Secrets

Mark ConfigMaps and Secrets as **immutable** once finalized. Immutable resources cannot be updated (only deleted and recreated), which prevents accidental changes and improves cluster performance by eliminating the watch overhead.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v3
  namespace: production
data:
  LOG_LEVEL: "warn"
  MAX_CONNECTIONS: "200"
immutable: true     # Cannot be modified after creation
```

- ✅ Use versioned names (`app-config-v3`) for immutable ConfigMaps
- ✅ Update the Deployment to reference the new ConfigMap name to trigger a rollout

---

## Security Best Practices

### Pod Security Standards

Kubernetes defines three **Pod Security Standards** that represent progressively restrictive security profiles:

| Standard     | Description                                          | When to Use                         |
|--------------|------------------------------------------------------|-------------------------------------|
| Privileged   | No restrictions — allows everything                  | System-level workloads (CNI, CSI)   |
| Baseline     | Prevents known privilege escalations                 | Non-critical workloads              |
| Restricted   | Heavily restricted — follows hardening best practices| Production application Pods         |

### Pod Security Admission

**Pod Security Admission** (PSA) enforces Pod Security Standards at the namespace level using labels. It replaced the deprecated PodSecurityPolicy.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted — reject Pods that violate
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Warn on baseline violations (for migration)
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    # Audit log restricted violations
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### SecurityContext

Apply a restrictive **SecurityContext** to every Pod and container. This limits the blast radius of a container compromise.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true            # Pod-level: no container may run as root
    seccompProfile:
      type: RuntimeDefault        # Enable default seccomp profile
  containers:
    - name: app
      image: myregistry/app:v1.0.0@sha256:abc123...
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true     # Prevent writes to container filesystem
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        capabilities:
          drop:
            - ALL                        # Drop all Linux capabilities
      volumeMounts:
        - name: tmp
          mountPath: /tmp                # Writable temp directory
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
```

- ✅ Always set `runAsNonRoot: true` and `allowPrivilegeEscalation: false`
- ✅ Use `readOnlyRootFilesystem: true` and mount writable volumes explicitly
- ✅ Drop all capabilities with `capabilities.drop: [ALL]`
- ✅ Enable the `RuntimeDefault` seccomp profile

### Network Policies

Apply a **default deny** NetworkPolicy to every namespace, then allowlist only the traffic that is required.

```yaml
# Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}         # Applies to ALL Pods in namespace
  policyTypes:
    - Ingress
    - Egress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: web-app
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: database
      ports:
        - protocol: TCP
          port: 5432
    - to:                         # Allow DNS resolution
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### RBAC Least Privilege

Grant the **minimum permissions** necessary. Prefer namespace-scoped **Roles** over cluster-wide **ClusterRoles**.

```yaml
# Role — namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: team-alpha
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
# RoleBinding — bind Role to a specific user or group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-team-alpha
  namespace: team-alpha
subjects:
  - kind: Group
    name: team-alpha-devs
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

- ✅ Use namespace-scoped Roles instead of ClusterRoles when possible
- ✅ Bind to Groups rather than individual Users for easier management
- ✅ Audit RBAC regularly with `kubectl auth can-i --list --as=<user> -n <ns>`
- ❌ Never grant `cluster-admin` to application workloads
- ❌ Avoid wildcard verbs (`*`) or resources (`*`) in production

### Image Security

- ✅ Scan images in CI with tools like **Trivy**, **Grype**, or **Snyk**
- ✅ Reference images by **digest** (`@sha256:...`) instead of mutable tags
- ✅ Use an **image allowlist** with admission controllers (OPA Gatekeeper, Kyverno)
- ✅ Keep base images minimal — prefer distroless or scratch
- ❌ Never use `:latest` in production
- ❌ Do not pull from untrusted public registries without scanning

### Service Account Hardening

Disable automatic mounting of the service account token unless the workload needs to call the Kubernetes API.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: false    # Disable for all Pods using this SA
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      serviceAccountName: app-sa
      automountServiceAccountToken: false  # Also disable at Pod level
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
```

---

## Reliability Patterns

### Health Checks

Kubernetes uses three types of probes to determine Pod health. Every production workload should define at least **readiness** and **liveness** probes.

| Probe     | Purpose                                        | Failure Action                    |
|-----------|-------------------------------------------------|-----------------------------------|
| Startup   | Wait for slow-starting apps before other probes | Restart container                 |
| Liveness  | Detect deadlocks and unrecoverable states       | Restart container                 |
| Readiness | Determine if the Pod can accept traffic         | Remove from Service endpoints     |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
          ports:
            - containerPort: 8080
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30       # 30 * 10s = 5 min to start
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 0     # Startup probe handles delay
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
---
# Example using tcpSocket probe (for non-HTTP services)
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
    - name: redis
      image: redis:7-alpine
      ports:
        - containerPort: 6379
      livenessProbe:
        tcpSocket:
          port: 6379
        periodSeconds: 10
      readinessProbe:
        exec:
          command:
            - redis-cli
            - ping
        periodSeconds: 5
```

- ✅ Use `startupProbe` for applications with variable initialization times
- ✅ Keep liveness checks lightweight — a simple `/healthz` endpoint
- ✅ Make readiness checks reflect actual ability to serve traffic
- ❌ Do not use liveness probes that depend on external services — this causes cascading restarts

### Pod Disruption Budgets

**PodDisruptionBudgets (PDBs)** tell Kubernetes how many Pods can be voluntarily disrupted (node drain, cluster upgrade) at the same time. Without PDBs, a node drain can terminate all replicas simultaneously.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  minAvailable: 2             # At least 2 Pods must remain running
  # OR use: maxUnavailable: 1 # At most 1 Pod can be down at a time
  selector:
    matchLabels:
      app: api-server
```

- ✅ Define a PDB for every production Deployment with 2+ replicas
- ✅ Use `maxUnavailable: 1` for most workloads
- ❌ Do not set `minAvailable` equal to `replicas` — this blocks node drains entirely

### Topology Spread Constraints

Spread Pods across failure domains (zones, nodes) to survive infrastructure failures.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-server
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway    # Soft constraint for nodes
          labelSelector:
            matchLabels:
              app: api-server
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
```

### Pod Anti-Affinity

Ensure replicas of the same application do not land on the same node. This provides high availability if a single node fails.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - api-server
              topologyKey: kubernetes.io/hostname
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
```

- ✅ Use `requiredDuringScheduling` for critical services (hard anti-affinity)
- ✅ Use `preferredDuringScheduling` for non-critical services (soft anti-affinity)
- ❌ Do not use hard anti-affinity when you have more replicas than nodes

### Graceful Shutdown

When Kubernetes terminates a Pod, it sends a `SIGTERM` signal and waits for `terminationGracePeriodSeconds` (default: 30s) before sending `SIGKILL`. Use `preStop` hooks to drain connections before the process exits.

```
  Pod Termination Sequence:

  1. Pod marked for deletion
  2. Pod removed from Service endpoints (async)
  3. preStop hook executes
  4. SIGTERM sent to PID 1
  5. Wait terminationGracePeriodSeconds
  6. SIGKILL sent (forced termination)
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      terminationGracePeriodSeconds: 60    # Allow up to 60s for graceful shutdown
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
          ports:
            - containerPort: 8080
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - |
                    # Give kube-proxy/endpoints time to update
                    sleep 5
                    # Signal the app to stop accepting new requests
                    kill -SIGTERM 1
                    # Wait for in-flight requests to complete
                    sleep 10
```

- ✅ Set `terminationGracePeriodSeconds` longer than your longest expected request
- ✅ Add a short `sleep` in `preStop` to allow endpoint propagation
- ✅ Ensure your application handles `SIGTERM` and drains in-flight requests

---

## Observability

### Structured Logging

Use **structured JSON logging** so log aggregation systems (Elasticsearch, Loki, CloudWatch) can parse, search, and filter effectively.

```
  ❌ Unstructured:
  2024-01-15 10:23:45 ERROR Failed to process order 12345

  ✅ Structured JSON:
  {
    "timestamp": "2024-01-15T10:23:45.123Z",
    "level": "error",
    "message": "Failed to process order",
    "orderId": "12345",
    "userId": "user-789",
    "traceId": "abc-def-123",
    "service": "order-processor"
  }
```

- ✅ Log to `stdout` and `stderr` — let the platform handle collection
- ✅ Include correlation IDs (`traceId`, `requestId`) in every log line
- ✅ Use consistent field names across all services
- ❌ Do not log sensitive data (passwords, tokens, PII)

### Prometheus Metrics

Expose Prometheus metrics using **annotations** to enable auto-discovery by Prometheus scrape configs.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  template:
    metadata:
      labels:
        app: api-server
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: api
          image: myregistry/api-server:v1.4.2
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
```

Key metrics every service should expose:

| Metric Type  | Example                              | Purpose                    |
|--------------|--------------------------------------|----------------------------|
| Counter      | `http_requests_total`                | Request rate and error rate|
| Histogram    | `http_request_duration_seconds`      | Latency percentiles       |
| Gauge        | `active_connections`                 | Current state              |
| Counter      | `errors_total{type="timeout"}`       | Error classification       |

### Label Strategy for Monitoring

Apply a **consistent labeling strategy** across all resources to enable effective monitoring, alerting, and cost attribution.

```yaml
metadata:
  labels:
    # Recommended Kubernetes labels
    app.kubernetes.io/name: api-server
    app.kubernetes.io/instance: api-server-production
    app.kubernetes.io/version: v1.4.2
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: payment-platform
    app.kubernetes.io/managed-by: helm
    # Custom labels for operations
    team: platform
    environment: production
    cost-center: engineering
```

### Distributed Tracing

Implement distributed tracing with **OpenTelemetry** to follow requests across service boundaries. Traces complement logs and metrics by showing the full request lifecycle.

```
  Trace: POST /checkout
  ┌───────────────────────────────────────────────────────┐
  │ api-gateway           [45ms]                          │
  │  ├── auth-service     [8ms]                           │
  │  ├── order-service    [25ms]                          │
  │  │    ├── inventory   [10ms]                          │
  │  │    └── database    [12ms]                          │
  │  └── payment-service  [15ms]                          │
  └───────────────────────────────────────────────────────┘
```

- ✅ Use **OpenTelemetry SDK** for vendor-neutral instrumentation
- ✅ Propagate trace context headers (`traceparent`, `tracestate`) across services
- ✅ Sample traces in production (e.g., 1-5%) to control storage costs

---

## Namespace Strategy

### Organizational Patterns

Choose a namespace strategy based on team structure and operational requirements.

| Pattern           | Example Namespaces                                   | Best For                         |
|-------------------|------------------------------------------------------|----------------------------------|
| Per-team          | `team-alpha`, `team-beta`, `platform`                | Small-medium organizations       |
| Per-environment   | `dev`, `staging`, `production`                       | Single-team clusters             |
| Per-application   | `payment-api`, `user-service`, `analytics`           | Microservice architectures       |
| Hybrid            | `team-alpha-prod`, `team-alpha-dev`                  | Large organizations              |

### Resource Isolation and RBAC

Combine namespaces with ResourceQuotas, LimitRanges, NetworkPolicies, and RBAC to create isolated tenant boundaries.

```
  Namespace Isolation Layers:

  ┌──────────────────────────────────────┐
  │            Namespace                 │
  │  ┌────────────┐  ┌───────────────┐  │
  │  │   RBAC     │  │  ResourceQuota│  │
  │  │ (who can   │  │ (how much     │  │
  │  │  access)   │  │  capacity)    │  │
  │  └────────────┘  └───────────────┘  │
  │  ┌────────────┐  ┌───────────────┐  │
  │  │ Network    │  │  LimitRange   │  │
  │  │ Policies   │  │ (per-Pod      │  │
  │  │ (traffic)  │  │  defaults)    │  │
  │  └────────────┘  └───────────────┘  │
  └──────────────────────────────────────┘
```

### Cross-Namespace Communication

Services in different namespaces communicate using fully qualified DNS names:

```
<service-name>.<namespace>.svc.cluster.local
```

```bash
# From namespace "frontend", call service "api" in namespace "backend"
curl http://api.backend.svc.cluster.local:8080/health
```

- ✅ Use NetworkPolicies with `namespaceSelector` to control cross-namespace access
- ❌ Do not rely on namespace boundaries alone for security — combine with NetworkPolicies and RBAC

---

## Helm Best Practices

### Chart Structure

Follow the standard Helm chart directory layout:

```
my-chart/
├── Chart.yaml              # Chart metadata (name, version, dependencies)
├── values.yaml             # Default configuration values
├── values-production.yaml  # Environment-specific overrides
├── templates/
│   ├── _helpers.tpl        # Template helper functions
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt           # Post-install instructions
└── tests/
    └── test-connection.yaml
```

### Values File Management

- ✅ Provide sensible defaults in `values.yaml` that work for development
- ✅ Use separate values files for each environment (`values-staging.yaml`, `values-production.yaml`)
- ✅ Document every value with comments in `values.yaml`
- ❌ Do not put secrets in values files — use External Secrets or `--set` from CI

```bash
# Install with environment-specific values
helm install api-server ./my-chart \
  -f values.yaml \
  -f values-production.yaml \
  -n production

# Override individual values from CI
helm upgrade api-server ./my-chart \
  --set image.tag=v1.4.2 \
  --set replicaCount=5
```

### Chart Testing

Validate charts before deployment to catch errors early.

```bash
# Lint the chart for issues
helm lint ./my-chart
helm lint ./my-chart -f values-production.yaml

# Render templates locally without deploying
helm template api-server ./my-chart -f values-production.yaml

# Dry-run against the cluster API server
helm install api-server ./my-chart --dry-run --debug

# Run chart tests after deployment
helm test api-server -n production
```

---

## CI/CD Best Practices

### Image Tagging

- ✅ Tag images with the **Git SHA** or **semantic version** (`v1.4.2`, `abc1234`)
- ✅ Use immutable tags — once pushed, a tag should never be overwritten
- ✅ Reference images by **digest** in production manifests for maximum reproducibility
- ❌ Never use `:latest` — it is mutable and makes rollbacks impossible

```yaml
# ❌ Bad — mutable, ambiguous
image: myregistry/api-server:latest

# ✅ Better — semantic version
image: myregistry/api-server:v1.4.2

# ✅ Best — immutable digest
image: myregistry/api-server@sha256:a1b2c3d4e5f6...
```

### Manifest Validation

Validate Kubernetes manifests in CI before they reach the cluster.

```bash
# Validate manifest structure against Kubernetes schemas
kubeconform -strict -summary deployment.yaml

# Validate with specific Kubernetes version
kubeconform -kubernetes-version 1.29.0 -strict manifests/

# Policy checks with kyverno CLI or OPA/conftest
kyverno apply policies/ --resource manifests/

# Helm chart validation
helm template my-release ./chart | kubeconform -strict -summary
```

### Progressive Rollouts

Use **Argo Rollouts** for advanced deployment strategies beyond native Kubernetes capabilities.

```
  Argo Rollouts — Canary with Analysis:

  Step 1: Deploy canary (10% traffic)
  Step 2: Run AnalysisTemplate (check error rate, latency)
  Step 3: If analysis passes → increase to 30%
  Step 4: Run analysis again
  Step 5: If analysis passes → promote to 100%
  Step 6: If analysis fails at any step → automatic rollback
```

- ✅ Automate rollback decisions with metric-based analysis
- ✅ Use traffic management (Istio, Nginx) for precise canary traffic splitting
- ✅ Include smoke tests and synthetic checks in the rollout pipeline

---

## Cost Optimization

### Spot and Preemptible Nodes

Use **spot instances** (AWS), **preemptible VMs** (GCP), or **spot VMs** (Azure) for fault-tolerant workloads to reduce compute costs by 60-90%.

- ✅ Run stateless, fault-tolerant workloads (batch jobs, dev/test) on spot nodes
- ✅ Use **taints and tolerations** to schedule spot-eligible workloads
- ✅ Combine spot and on-demand nodes — critical workloads on on-demand, the rest on spot
- ❌ Do not run databases or stateful singletons on spot nodes

### Cluster Autoscaling

**Cluster Autoscaler** adds or removes nodes based on pending Pod scheduling and node utilization. **Karpenter** (AWS) offers faster, more flexible node provisioning.

```
  Autoscaling Layers:

  ┌──────────────────────────────────────┐
  │           HPA / VPA                  │  ← Pod-level scaling
  │  (scale Pods up/down or resize)      │
  ├──────────────────────────────────────┤
  │   Cluster Autoscaler / Karpenter     │  ← Node-level scaling
  │  (add/remove nodes as Pods need      │
  │   capacity)                          │
  └──────────────────────────────────────┘
```

- ✅ Set appropriate `--scale-down-utilization-threshold` (default: 0.5)
- ✅ Use Karpenter for heterogeneous instance types and faster provisioning
- ✅ Define `PodDisruptionBudgets` so autoscaler respects availability during scale-down

### Namespace Budgets

Combine ResourceQuotas with cost monitoring to enforce per-team budgets.

- ✅ Set ResourceQuotas per namespace to cap resource consumption
- ✅ Use cost monitoring tools (**Kubecost**, **OpenCost**) for visibility
- ✅ Review and right-size resources monthly
- ✅ Label resources with `cost-center` and `team` for cost attribution

---

## Summary

| Practice                          | Benefit                                  | Priority   |
|-----------------------------------|------------------------------------------|------------|
| Set resource requests and limits  | Predictable scheduling, prevent eviction | 🔴 Critical |
| Define readiness/liveness probes  | Zero-downtime deploys, auto-healing      | 🔴 Critical |
| Use non-root containers           | Reduce blast radius of compromises       | 🔴 Critical |
| Drop all capabilities             | Minimize kernel attack surface           | 🔴 Critical |
| Never use `:latest` tag           | Reproducible, auditable deployments      | 🔴 Critical |
| Default deny NetworkPolicies      | Prevent lateral movement                 | 🟠 High    |
| Enable Pod Security Admission     | Enforce security standards cluster-wide  | 🟠 High    |
| Define PodDisruptionBudgets       | Safe node drains and cluster upgrades    | 🟠 High    |
| Use GitOps for deployments        | Audit trail, drift detection             | 🟠 High    |
| Externalize config with ConfigMaps| Environment portability                  | 🟠 High    |
| Implement HPA                     | Handle load spikes automatically         | 🟡 Medium  |
| Use topology spread constraints   | Survive zone/node failures               | 🟡 Medium  |
| Structured JSON logging           | Effective log aggregation and search     | 🟡 Medium  |
| Scan images in CI                 | Catch vulnerabilities before deploy      | 🟡 Medium  |
| Use Kustomize/Helm for config     | Manage environment differences cleanly   | 🟡 Medium  |
| Right-size resources with metrics | Reduce waste and lower costs             | 🟢 Low     |
| Use spot/preemptible nodes        | Reduce compute costs 60-90%             | 🟢 Low     |
| Implement distributed tracing     | Debug cross-service latency issues       | 🟢 Low     |

---

## Next Steps

Continue building your Kubernetes knowledge with related topics:

- [Anti-Patterns](11-ANTI-PATTERNS.md) — Common mistakes and how to avoid them
- [Kubernetes Overview](00-OVERVIEW.md) — Review the architecture and core concepts
- [Services and Networking](01-SERVICES.md) — Service types, discovery, and traffic policies
- [Manifests](06-MANIFESTS.md) — Writing and managing Kubernetes manifests
- [CRDs and Operators](07-CRDS-AND-OPERATORS.md) — Extending Kubernetes with custom resources
- [Networking](09-NETWORKING.md) — Deep dive into cluster networking
