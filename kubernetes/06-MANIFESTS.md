# Kubernetes Manifests

## Table of Contents

1. [Overview](#overview)
2. [Declarative vs Imperative](#declarative-vs-imperative)
3. [Manifest Structure](#manifest-structure)
   - [apiVersion](#apiversion)
   - [kind](#kind)
   - [metadata](#metadata)
   - [spec](#spec)
4. [Pod Manifests](#pod-manifests)
   - [Single-Container Pod](#single-container-pod)
   - [Multi-Container Pods](#multi-container-pods)
   - [Resource Requests and Limits](#resource-requests-and-limits)
5. [Deployments](#deployments)
   - [Rolling Updates](#rolling-updates)
   - [Rollbacks](#rollbacks)
   - [Update Strategy](#update-strategy)
   - [Scaling](#scaling)
6. [ReplicaSets](#replicasets)
7. [StatefulSets](#statefulsets)
8. [DaemonSets](#daemonsets)
9. [Jobs and CronJobs](#jobs-and-cronjobs)
   - [Jobs](#jobs)
   - [CronJobs](#cronjobs)
10. [ConfigMaps](#configmaps)
11. [Secrets](#secrets)
12. [Persistent Volumes and Claims](#persistent-volumes-and-claims)
    - [PersistentVolume](#persistentvolume)
    - [PersistentVolumeClaim](#persistentvolumeclaim)
    - [StorageClasses and Dynamic Provisioning](#storageclasses-and-dynamic-provisioning)
13. [Namespaces](#namespaces)
    - [Resource Quotas](#resource-quotas)
    - [Limit Ranges](#limit-ranges)
14. [Labels, Selectors, and Annotations](#labels-selectors-and-annotations)
15. [Kustomize](#kustomize)
16. [Helm Charts](#helm-charts)
17. [Manifest Validation and Linting](#manifest-validation-and-linting)
18. [Best Practices](#best-practices)
19. [Next Steps](#next-steps)

## Overview

This document provides a comprehensive guide to Kubernetes manifests — the YAML (or JSON) files that declare the desired state of every resource in a cluster. It covers the anatomy of a manifest, every major resource type, templating and overlay tools, and the validation workflow that keeps manifests correct before they reach the API server.

### Target Audience

- Developers writing and maintaining Kubernetes resource definitions
- DevOps engineers building deployment pipelines around manifests
- SREs reviewing manifests for production readiness
- Platform engineers designing manifest standards for their organizations

### Scope

- Manifest structure and the four top-level fields
- Workload resources: Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, CronJobs
- Configuration resources: ConfigMaps, Secrets
- Storage resources: PersistentVolumes, PersistentVolumeClaims, StorageClasses
- Organizational resources: Namespaces, Labels, Annotations
- Tooling: Kustomize, Helm, kubeval, kubeconform, kube-score

## Declarative vs Imperative

Kubernetes supports two approaches for managing resources:

| Aspect | Imperative | Declarative |
|---|---|---|
| **Command style** | `kubectl create deployment nginx --image=nginx` | `kubectl apply -f deployment.yaml` |
| **State tracking** | Cluster only | Manifest files (source of truth) |
| **Reproducibility** | Low — commands are not easily repeatable | High — YAML is version-controlled |
| **Auditability** | Requires command history | Git history provides full audit trail |
| **Collaboration** | Difficult across teams | Pull request workflows with peer review |
| **Recommended for** | Quick prototyping, one-off tasks | Production workloads, GitOps pipelines |

> **Note:** Declarative management with `kubectl apply` is the recommended approach. It enables drift detection, three-way merges, and integrates naturally with CI/CD pipelines.

```
Imperative Flow                    Declarative Flow
─────────────────                  ──────────────────
                                   ┌──────────────┐
┌────────────┐                     │  YAML Files  │
│  kubectl   │──► API Server       │  (Git repo)  │
│  create /  │                     └──────┬───────┘
│  run / set │                            │
└────────────┘                     kubectl apply -f
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │  API Server   │
                                   └──────┬───────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │  Desired =   │
                                   │  Actual?     │
                                   │  Controller  │
                                   │  reconciles  │
                                   └──────────────┘
```

## Manifest Structure

Every Kubernetes manifest shares four top-level fields:

```yaml
apiVersion: <group/version>   # API group and version
kind: <ResourceType>          # Type of resource
metadata:                     # Identity and organizational data
  name: my-resource
  namespace: default
  labels:
    app: my-app
spec:                         # Desired state (resource-specific)
  ...
```

### apiVersion

The `apiVersion` field tells the API server which schema to validate the manifest against.

| Resource | apiVersion |
|---|---|
| Pod | `v1` |
| Service | `v1` |
| ConfigMap | `v1` |
| Secret | `v1` |
| Namespace | `v1` |
| PersistentVolume | `v1` |
| PersistentVolumeClaim | `v1` |
| Deployment | `apps/v1` |
| ReplicaSet | `apps/v1` |
| StatefulSet | `apps/v1` |
| DaemonSet | `apps/v1` |
| Job | `batch/v1` |
| CronJob | `batch/v1` |
| Ingress | `networking.k8s.io/v1` |

Use `kubectl api-resources` to list all available resource types and their API versions:

```bash
kubectl api-resources --sort-by=name
```

### kind

The `kind` field identifies the resource type — `Pod`, `Deployment`, `Service`, etc. It must match the resource type registered under the specified `apiVersion`.

### metadata

Common metadata fields:

| Field | Description |
|---|---|
| `name` | Unique name within the namespace |
| `namespace` | Namespace scope (defaults to `default`) |
| `labels` | Key-value pairs for selection and grouping |
| `annotations` | Key-value pairs for non-identifying metadata |
| `generateName` | Prefix for auto-generated names (used by controllers) |

### spec

The `spec` field is resource-specific and describes the desired state. For a Pod it defines containers; for a Deployment it defines replicas, strategy, and a pod template.

## Pod Manifests

A Pod is the smallest deployable unit in Kubernetes. In most production scenarios Pods are managed by higher-level controllers (Deployments, StatefulSets), but understanding Pod manifests is foundational.

### Single-Container Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
```

```bash
# Apply and verify
kubectl apply -f nginx-pod.yaml
kubectl get pod nginx-pod
kubectl describe pod nginx-pod
```

### Multi-Container Pods

Multi-container pods share the same network namespace and storage volumes. Common patterns include sidecar containers and init containers.

#### Sidecar Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
  labels:
    app: web
spec:
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: log-shipper
      image: fluentbit:2.1
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
          readOnly: true
  volumes:
    - name: shared-logs
      emptyDir: {}
```

#### Init Containers

Init containers run to completion before any app containers start. They are useful for setup tasks like database migrations or downloading configuration.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - "until nc -z postgres-svc 5432; do echo waiting; sleep 2; done"
    - name: run-migrations
      image: my-app:1.0
      command: ["./migrate", "--up"]
  containers:
    - name: app
      image: my-app:1.0
      ports:
        - containerPort: 8080
```

### Resource Requests and Limits

Resource requests guarantee a minimum allocation; limits cap the maximum. The scheduler uses requests to place pods on nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: my-app:1.0
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

| Field | Purpose |
|---|---|
| `requests.cpu` | Minimum CPU guaranteed (250 millicores = 0.25 vCPU) |
| `requests.memory` | Minimum memory guaranteed |
| `limits.cpu` | Maximum CPU — pod is throttled beyond this |
| `limits.memory` | Maximum memory — pod is OOMKilled beyond this |

> **Note:** Always set both requests and limits in production. Pods without requests may be scheduled on overcommitted nodes and evicted under memory pressure.

## Deployments

A Deployment manages a ReplicaSet and provides declarative updates for Pods. It is the most common way to run stateless workloads.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: my-web-app:2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -l app=web-app
```

### Rolling Updates

By default, Deployments perform rolling updates — gradually replacing old pods with new ones.

```bash
# Update the image
kubectl set image deployment/web-app web=my-web-app:3.0

# Watch the rollout
kubectl rollout status deployment/web-app
```

### Rollbacks

```bash
# View rollout history
kubectl rollout history deployment/web-app

# Rollback to the previous revision
kubectl rollout undo deployment/web-app

# Rollback to a specific revision
kubectl rollout undo deployment/web-app --to-revision=2
```

### Update Strategy

Kubernetes supports two update strategies:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max pods above desired count during update
      maxUnavailable: 0     # Zero downtime — always keep all replicas available
```

| Strategy | Behavior | Use Case |
|---|---|---|
| `RollingUpdate` | Gradually replaces old pods with new ones | Default — zero/minimal downtime |
| `Recreate` | Terminates all old pods before creating new ones | When old and new versions cannot coexist |

```yaml
# Recreate strategy
spec:
  strategy:
    type: Recreate
```

### Scaling

```bash
# Scale imperatively
kubectl scale deployment/web-app --replicas=5

# Or update the manifest and reapply
# spec.replicas: 5
kubectl apply -f deployment.yaml

# Autoscale based on CPU utilization
kubectl autoscale deployment/web-app --min=2 --max=10 --cpu-percent=70
```

## ReplicaSets

A ReplicaSet ensures a specified number of pod replicas are running at any given time. Deployments manage ReplicaSets automatically — you rarely create them directly.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-app-rs
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: my-web-app:2.0
          ports:
            - containerPort: 8080
```

```
Deployment ──manages──► ReplicaSet ──manages──► Pod, Pod, Pod
                           │
                   (tracks revision history)
```

> **Note:** Use a Deployment instead of a ReplicaSet directly. Deployments provide rollout history, rollback support, and update strategies that ReplicaSets lack. The only reason to use a bare ReplicaSet is if you need custom update orchestration that the Deployment controller does not support.

## StatefulSets

StatefulSets manage stateful applications that require stable network identities, ordered deployment, and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

Key StatefulSet behaviors:

| Feature | Description |
|---|---|
| **Stable network identity** | Pods get predictable names: `postgres-0`, `postgres-1`, `postgres-2` |
| **Ordered creation** | Pods are created sequentially (0 → 1 → 2) |
| **Ordered termination** | Pods are terminated in reverse order (2 → 1 → 0) |
| **Stable storage** | Each pod gets its own PersistentVolumeClaim that survives rescheduling |
| **Headless Service** | Required for DNS-based pod discovery (`postgres-0.postgres-headless.default.svc.cluster.local`) |

```bash
kubectl apply -f statefulset.yaml
kubectl get statefulset postgres
kubectl get pods -l app=postgres
```

## DaemonSets

A DaemonSet ensures that a copy of a pod runs on every (or a selected set of) node(s). Common use cases include log collectors, monitoring agents, and network plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
  namespace: logging
  labels:
    app: fluentbit
spec:
  selector:
    matchLabels:
      app: fluentbit
  template:
    metadata:
      labels:
        app: fluentbit
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentbit
          image: fluent/fluent-bit:3.0
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset -n logging
kubectl get pods -n logging -l app=fluentbit -o wide
```

> **Note:** DaemonSets ignore most scheduling constraints by default. Use `nodeSelector` or `affinity` rules to target specific nodes. Add `tolerations` to run on control-plane nodes.

## Jobs and CronJobs

### Jobs

A Job creates one or more pods and ensures they run to completion. Jobs are used for batch processing, data migrations, and one-off tasks.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: my-app:1.0
          command: ["./migrate", "--up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
```

Parallel Jobs process multiple work items concurrently:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10
  parallelism: 3
  backoffLimit: 5
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: batch-worker:1.0
```

| Field | Description |
|---|---|
| `completions` | Total number of pod completions required |
| `parallelism` | Maximum pods running concurrently |
| `backoffLimit` | Number of retries before marking the Job as failed |
| `activeDeadlineSeconds` | Maximum duration for the entire Job |
| `restartPolicy` | Must be `Never` or `OnFailure` (not `Always`) |

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/db-migration
```

### CronJobs

CronJobs run Jobs on a schedule using standard cron syntax.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["./backup.sh"]
              env:
                - name: S3_BUCKET
                  value: "my-backups"
```

| Field | Description |
|---|---|
| `schedule` | Cron expression (minute hour day-of-month month day-of-week) |
| `concurrencyPolicy` | `Allow`, `Forbid`, or `Replace` — controls overlapping runs |
| `successfulJobsHistoryLimit` | Number of completed Jobs to retain |
| `failedJobsHistoryLimit` | Number of failed Jobs to retain |
| `startingDeadlineSeconds` | Deadline for starting the Job if it misses its schedule |

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl get jobs --watch
```

## ConfigMaps

ConfigMaps store non-confidential configuration data as key-value pairs. They can be consumed as environment variables, command-line arguments, or mounted as files.

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# From a file
kubectl create configmap nginx-config \
  --from-file=nginx.conf

# From an env file
kubectl create configmap env-config \
  --from-env-file=app.env
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10
```

### Using ConfigMaps as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config
      env:
        - name: SPECIFIC_KEY
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

### Mounting ConfigMaps as Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config-volume
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        items:
          - key: config.yaml
            path: config.yaml
```

## Secrets

Secrets store sensitive data such as passwords, tokens, and TLS certificates. Values are base64-encoded (not encrypted) at rest unless you enable encryption at the API server level.

### Secret Types

| Type | Description |
|---|---|
| `Opaque` | Default type — arbitrary key-value pairs |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and private key |
| `kubernetes.io/basic-auth` | Username and password |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/service-account-token` | Auto-generated for ServiceAccounts |

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cureP@ss!'

# Docker registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# TLS secret
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: "S3cureP@ss!"
```

> **Note:** Use `stringData` for plaintext values (Kubernetes encodes them automatically). Use `data` with base64-encoded values for pre-encoded content.

### Using Secrets in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true
  volumes:
    - name: tls-certs
      secret:
        secretName: my-tls
```

> **Caveat:** Secrets are base64-encoded, not encrypted. For production, enable encryption at rest with `EncryptionConfiguration` or use an external secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault).

## Persistent Volumes and Claims

Kubernetes decouples storage provisioning from consumption using PersistentVolumes (PV) and PersistentVolumeClaims (PVC).

```
┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
│   StorageClass     │     │  PersistentVolume   │     │       Pod          │
│  (provisioner,     │◄────│  (capacity, access  │◄────│  (volumeMounts)    │
│   parameters)      │     │   mode, reclaim)    │     │                    │
└────────────────────┘     └────────┬───────────┘     └────────────────────┘
                                    │ bound to                  ▲
                           ┌────────▼───────────┐              │
                           │ PersistentVolume    │──────────────┘
                           │ Claim (PVC)         │  referenced by pod
                           └────────────────────┘
```

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

| Access Mode | Abbreviation | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted read-write by a single node |
| `ReadOnlyMany` | ROX | Mounted read-only by many nodes |
| `ReadWriteMany` | RWX | Mounted read-write by many nodes |

| Reclaim Policy | Behavior |
|---|---|
| `Retain` | PV is kept after PVC is deleted (manual cleanup) |
| `Delete` | PV and underlying storage are deleted |
| `Recycle` | Deprecated — data is scrubbed and PV is reused |

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 10Gi
```

Using a PVC in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
```

### StorageClasses and Dynamic Provisioning

StorageClasses enable dynamic provisioning — PersistentVolumes are created automatically when a PVC is submitted.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "50"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```yaml
# PVC using dynamic provisioning
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl get pv,pvc
```

## Namespaces

Namespaces provide logical isolation within a cluster. They scope resource names and can be used to apply resource quotas, limit ranges, and RBAC policies.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
```

```bash
kubectl create namespace staging
kubectl get namespaces
kubectl get pods --namespace=staging

# Set a default namespace for your context
kubectl config set-context --current --namespace=staging
```

### Resource Quotas

Resource quotas limit the total resource consumption within a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

### Limit Ranges

LimitRanges set default and maximum resource constraints for individual containers within a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: staging
spec:
  limits:
    - type: Container
      default:
        cpu: "250m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

```bash
kubectl apply -f resourcequota.yaml
kubectl apply -f limitrange.yaml
kubectl describe namespace staging
```

## Labels, Selectors, and Annotations

### Labels

Labels are key-value pairs attached to resources for identification and selection. They drive how Services route traffic, how Deployments find their Pods, and how operators query resources.

```yaml
metadata:
  labels:
    app: web-app
    version: v2
    environment: production
    team: platform
```

Recommended label conventions from the Kubernetes documentation:

| Label | Example | Purpose |
|---|---|---|
| `app.kubernetes.io/name` | `web-app` | Application name |
| `app.kubernetes.io/version` | `2.0.0` | Application version |
| `app.kubernetes.io/component` | `frontend` | Component within the architecture |
| `app.kubernetes.io/part-of` | `ecommerce` | Higher-level application this belongs to |
| `app.kubernetes.io/managed-by` | `helm` | Tool managing the resource |

### Selectors

```bash
# Equality-based selectors
kubectl get pods -l app=web-app
kubectl get pods -l environment=production,team=platform

# Set-based selectors
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'app notin (test-app)'
kubectl get pods -l 'version'          # Has the label
kubectl get pods -l '!canary'          # Does not have the label
```

### Annotations

Annotations store non-identifying metadata — build info, URLs, tool configuration, or human-readable notes. Unlike labels, annotations cannot be used in selectors.

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Updated image to v2.0"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    buildUrl: "https://ci.example.com/builds/1234"
```

## Kustomize

Kustomize lets you customize Kubernetes manifests without modifying the originals. It uses overlays to patch base manifests for different environments.

### Directory Structure

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── resource-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── resource-patch.yaml
```

### Base kustomization.yaml

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: web-app
```

### Overlay kustomization.yaml

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: production
namePrefix: prod-
patches:
  - path: resource-patch.yaml
    target:
      kind: Deployment
      name: web-app
```

```yaml
# overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: web
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
```

```bash
# Preview the generated manifests
kubectl kustomize overlays/production

# Apply directly
kubectl apply -k overlays/production
```

## Helm Charts

Helm is a package manager for Kubernetes. Charts are collections of templated manifests, a `values.yaml` file, and metadata. While a full Helm guide is beyond this document's scope, understanding the chart structure helps when working with manifests.

### Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
└── templates/
    ├── _helpers.tpl    # Template helper functions
    ├── deployment.yaml # Templated Deployment manifest
    ├── service.yaml    # Templated Service manifest
    ├── configmap.yaml  # Templated ConfigMap
    ├── ingress.yaml    # Templated Ingress
    └── NOTES.txt       # Post-install usage notes
```

### Templated Manifest Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

```bash
# Install a chart
helm install my-release ./my-chart --values custom-values.yaml

# Upgrade a release
helm upgrade my-release ./my-chart --values custom-values.yaml

# Preview rendered templates without installing
helm template my-release ./my-chart --values custom-values.yaml
```

## Manifest Validation and Linting

Validating manifests before applying them catches errors early and prevents misconfigurations from reaching the cluster.

### kubeval

kubeval validates manifests against the official Kubernetes JSON schemas.

```bash
# Validate a single file
kubeval deployment.yaml

# Validate all YAML files in a directory
kubeval -d k8s/

# Validate against a specific Kubernetes version
kubeval --kubernetes-version 1.29.0 deployment.yaml
```

### kubeconform

kubeconform is a faster, actively maintained alternative to kubeval with support for custom resource definitions (CRDs).

```bash
# Validate manifests
kubeconform -summary deployment.yaml

# Validate with strict mode and specific Kubernetes version
kubeconform -strict -kubernetes-version 1.29.0 k8s/

# Validate including CRDs from a schema registry
kubeconform -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  k8s/
```

### kube-score

kube-score performs static analysis and scores manifests against best practices.

```bash
# Score a manifest
kube-score score deployment.yaml

# Score all manifests in a directory
kube-score score k8s/*.yaml
```

Common checks performed by kube-score:

| Check | Description |
|---|---|
| Container resources | Verifies requests and limits are set |
| Pod probes | Checks for readiness and liveness probes |
| Image tag | Warns against using `latest` tag |
| Network policy | Checks if a NetworkPolicy exists |
| Pod disruption budget | Checks if a PDB is configured |

### CI Pipeline Integration

```bash
# Example: validate manifests in a CI pipeline
kubeconform -strict -summary -output json k8s/ || exit 1
kube-score score k8s/*.yaml || echo "Warnings detected"
```

## Best Practices

### Manifest Organization

✅ **Do:**

- Store all manifests in version control alongside application code
- Use a consistent directory structure (`base/`, `overlays/`, or per-environment directories)
- Separate resources into individual files (`deployment.yaml`, `service.yaml`, `configmap.yaml`)
- Use Kustomize or Helm for environment-specific customization
- Include resource requests and limits on every container
- Set readiness and liveness probes on all long-running containers
- Use namespaces to isolate workloads by team or environment
- Pin container images to specific tags or digests — never use `latest`
- Apply the standard `app.kubernetes.io/*` labels
- Validate manifests in CI before they reach the cluster

❌ **Don't:**

- Commit Secrets with plaintext values to Git — use sealed-secrets or external secret managers
- Put multiple unrelated resources in the same YAML file
- Use imperative commands for production deployments
- Omit resource limits — unbounded containers can starve co-located workloads
- Rely on the `default` namespace for production workloads
- Use `kubectl edit` in production — changes are not tracked in version control

### Recommended Directory Layout

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── hpa.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── README.md
```

## Next Steps

Continue exploring Kubernetes topics with the other documents in this series:

- [Kubernetes Overview](00-OVERVIEW.md) — Architecture and core concepts
- [Kubernetes Services](01-SERVICES.md) — Service types, discovery, and networking
- [Cloud Services — AWS EKS](02-CLOUD-SERVICES-AWS-EKS.md) — Deploying to managed Kubernetes on AWS
- [Cloud Services — OKE](03-CLOUD-SERVICES-OKE.md) — Deploying to Oracle Kubernetes Engine
- [Cloud Services — AKS](04-CLOUD-SERVICES-AKS.md) — Deploying to Azure Kubernetes Service
- [Local Setup — kind and Minikube](05-LOCAL-SETUP-KIND-MINIKUBE.md) — Local development clusters

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Kubernetes manifests documentation |
