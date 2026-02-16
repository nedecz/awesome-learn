# Anti-Patterns

## Table of Contents

1. [Overview](#overview)
2. [Resource Anti-Patterns](#resource-anti-patterns)
   - [Not Setting Resource Requests and Limits](#not-setting-resource-requests-and-limits)
   - [Over-Provisioning Resources](#over-provisioning-resources)
   - [Using BestEffort QoS in Production](#using-besteffort-qos-in-production)
3. [Image Anti-Patterns](#image-anti-patterns)
   - [Using the `:latest` Tag](#using-the-latest-tag)
   - [Running as Root](#running-as-root)
   - [Using Bloated Base Images](#using-bloated-base-images)
   - [Not Scanning Images for Vulnerabilities](#not-scanning-images-for-vulnerabilities)
4. [Configuration Anti-Patterns](#configuration-anti-patterns)
   - [Hardcoding Config in Container Images](#hardcoding-config-in-container-images)
   - [Storing Secrets in ConfigMaps or Env Vars](#storing-secrets-in-configmaps-or-env-vars)
   - [Not Using Namespaces](#not-using-namespaces)
   - [Mutable ConfigMaps Causing Drift](#mutable-configmaps-causing-drift)
5. [Deployment Anti-Patterns](#deployment-anti-patterns)
   - [Using Bare Pods](#using-bare-pods)
   - [Not Using Health Checks](#not-using-health-checks)
   - [Single Replica in Production](#single-replica-in-production)
   - [No Pod Disruption Budget](#no-pod-disruption-budget)
   - [Ignoring Graceful Shutdown](#ignoring-graceful-shutdown)
6. [Networking Anti-Patterns](#networking-anti-patterns)
   - [No NetworkPolicies](#no-networkpolicies)
   - [Using NodePort for Everything](#using-nodeport-for-everything)
   - [Hardcoding Pod IPs](#hardcoding-pod-ips)
7. [Security Anti-Patterns](#security-anti-patterns)
   - [Running Privileged Containers](#running-privileged-containers)
   - [Using the Default ServiceAccount](#using-the-default-serviceaccount)
   - [Cluster-Wide ClusterRoleBindings](#cluster-wide-clusterrolebindings)
   - [No Pod Security Standards](#no-pod-security-standards)
   - [Mounting Service Account Tokens Unnecessarily](#mounting-service-account-tokens-unnecessarily)
8. [Storage Anti-Patterns](#storage-anti-patterns)
   - [Using hostPath in Production](#using-hostpath-in-production)
   - [Not Setting Storage Limits](#not-setting-storage-limits)
   - [Using emptyDir for Persistent Data](#using-emptydir-for-persistent-data)
9. [Scaling Anti-Patterns](#scaling-anti-patterns)
   - [Manual Scaling Only](#manual-scaling-only)
   - [No Cluster Autoscaler](#no-cluster-autoscaler)
   - [Setting HPA min Equals max](#setting-hpa-min-equals-max)
10. [Observability Anti-Patterns](#observability-anti-patterns)
    - [No Centralized Logging](#no-centralized-logging)
    - [No Metrics or Alerts](#no-metrics-or-alerts)
    - [No Labels or Annotations Strategy](#no-labels-or-annotations-strategy)
11. [Organizational Anti-Patterns](#organizational-anti-patterns)
    - [YAML Sprawl Without Structure](#yaml-sprawl-without-structure)
    - [No CI/CD for Manifests](#no-cicd-for-manifests)
    - [Everyone is cluster-admin](#everyone-is-cluster-admin)
12. [Summary Table](#summary-table)
13. [Next Steps](#next-steps)

---

## Overview

An **anti-pattern** is a common practice that appears helpful at first glance but ultimately leads to problems — degraded reliability, security vulnerabilities, runaway costs, or operational nightmares. In Kubernetes, anti-patterns are especially dangerous because they tend to remain invisible until a production incident forces them into the open.

### Why Anti-Patterns Matter

Kubernetes gives teams enormous flexibility. That flexibility comes with risk: there are very few guardrails out of the box. A Pod with no resource limits can starve an entire node. A container running as root can compromise a cluster. A Deployment with a single replica guarantees downtime during rolling updates.

Learning to recognize anti-patterns is just as important as learning best practices — they are two sides of the same coin.

### How to Use This Document

Each section follows a consistent structure:

- ❌ **Anti-pattern** — What teams commonly do wrong and why it seems reasonable.
- ✅ **Better approach** — The recommended practice, with YAML examples where applicable.
- **Why it matters** — The concrete risks of the anti-pattern and the benefits of the fix.

### Target Audience

- Developers deploying applications to Kubernetes
- Platform engineers building internal developer platforms
- SREs responsible for production reliability and security

---

## Resource Anti-Patterns

### Not Setting Resource Requests and Limits

Without resource requests and limits, the scheduler has no information about what a Pod needs. Pods land on random nodes, and a single noisy container can consume all CPU or memory on a node, starving everything else.

❌ **Anti-pattern — no resource constraints:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
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
          image: myapp:1.2.0
          # No resources block — BestEffort QoS, no scheduling hints
```

✅ **Better approach — always set requests and limits:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
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
          image: myapp:1.2.0
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

**Why it matters:**

- **Requests** tell the scheduler how much capacity a Pod needs — without them, bin-packing is blind.
- **Limits** prevent a single container from consuming all node resources.
- Without limits, an OOM condition on a node can kill unrelated Pods.
- Use `LimitRange` objects to enforce defaults at the namespace level so nothing slips through.

---

### Over-Provisioning Resources

Teams often set requests and limits far higher than actual usage — "just to be safe." This wastes cluster capacity, inflates cloud bills, and prevents efficient bin-packing.

❌ **Anti-pattern — requesting 4 CPU and 8 Gi for a service that uses 200m and 256Mi:**

```yaml
resources:
  requests:
    cpu: "4"
    memory: 8Gi
  limits:
    cpu: "4"
    memory: 8Gi
```

✅ **Better approach — right-size using observed metrics:**

```yaml
resources:
  requests:
    cpu: 250m
    memory: 384Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**How to right-size:**

1. Deploy with conservative estimates.
2. Monitor actual usage with `kubectl top pods`, Prometheus, or Vertical Pod Autoscaler (VPA) in recommendation mode.
3. Adjust requests to the P95 of observed usage and set limits with reasonable headroom.
4. Re-evaluate after every major release or traffic change.

---

### Using BestEffort QoS in Production

When a Pod has no resource requests or limits, Kubernetes assigns it the `BestEffort` Quality of Service class. BestEffort Pods are the first to be evicted when a node runs low on memory.

| QoS Class   | Condition                                  | Eviction Priority |
|-------------|--------------------------------------------|--------------------|
| BestEffort  | No requests or limits set                  | First to be evicted |
| Burstable   | Requests set but lower than limits         | Evicted after BestEffort |
| Guaranteed  | Requests equal limits for all containers   | Last to be evicted |

❌ **Anti-pattern — BestEffort in production (no resources specified).**

✅ **Better approach — use Guaranteed or Burstable QoS:**

```yaml
# Guaranteed QoS — requests equal limits
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

For most production workloads, `Burstable` (requests < limits) provides a good balance between resource efficiency and eviction safety. Use `Guaranteed` for mission-critical services where predictability is paramount.

---

## Image Anti-Patterns

### Using the `:latest` Tag

The `:latest` tag is mutable — it points to whatever was pushed last. This makes deployments non-reproducible: the same manifest can produce different containers depending on when it runs. Rollbacks become impossible because there is no way to know which image was previously running.

❌ **Anti-pattern — using :latest:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:latest
```

✅ **Better approach — use specific version tags or image digests:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0

    # Even better — pin to an immutable digest
    - name: worker
      image: myapp@sha256:abc123def456...
```

**Why it matters:**

- `:latest` defeats `imagePullPolicy: IfNotPresent` — you never know if the cached image matches what is in the registry.
- Digest pinning guarantees byte-for-byte reproducibility.
- Version tags make rollbacks trivial: `kubectl rollout undo` returns to a known state.
- Always set `imagePullPolicy: Always` if you must use mutable tags (but prefer immutable tags instead).

---

### Running as Root

By default, many container images run as UID 0 (root). If an attacker escapes the container, they have root privileges on the host — particularly dangerous when combined with other misconfigurations like privileged mode or host mounts.

❌ **Anti-pattern — no securityContext, implicitly running as root:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      # No securityContext — runs as root by default
```

✅ **Better approach — run as a non-root user:**

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: api
      image: myapp:1.2.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

**Why it matters:**

- `runAsNonRoot: true` causes the kubelet to reject the Pod if the image tries to run as UID 0.
- `allowPrivilegeEscalation: false` prevents `setuid` binaries from gaining additional privileges.
- `readOnlyRootFilesystem: true` blocks writes to the container filesystem, limiting attacker options.

---

### Using Bloated Base Images

Full OS images like `ubuntu:22.04` or `debian:bookworm` contain hundreds of packages — shells, package managers, network tools — that most applications never need. Each unnecessary package is additional attack surface and increases image pull time.

❌ **Anti-pattern — large base images:**

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 python3-pip
COPY . /app
CMD ["python3", "/app/main.py"]
# Image size: ~450 MB
```

✅ **Better approach — use minimal base images:**

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM gcr.io/distroless/python3-debian12
COPY --from=builder /app /app
CMD ["main.py"]
# Image size: ~50 MB
```

**Recommended minimal base images:**

| Use Case      | Image                              | Size     |
|---------------|------------------------------------|----------|
| Go binaries   | `gcr.io/distroless/static-debian12`| ~2 MB    |
| General       | `alpine:3.19`                      | ~7 MB    |
| Python        | `python:3.12-slim`                 | ~50 MB   |
| Java          | `eclipse-temurin:21-jre-alpine`    | ~80 MB   |
| Scratch       | `scratch`                          | 0 MB     |

---

### Not Scanning Images for Vulnerabilities

Deploying unscanned images means you discover CVEs in production — via an incident, not a CI pipeline.

❌ **Anti-pattern — push and deploy with no scanning.**

✅ **Better approach — scan in CI before images reach any cluster:**

```yaml
# Example GitHub Actions step
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    exit-code: "1"
    severity: "CRITICAL,HIGH"
    format: "table"
```

Integrate scanning into admission control as well — use tools like Kyverno, OPA Gatekeeper, or cloud-native admission policies to reject unscanned or vulnerable images at deploy time.

---

## Configuration Anti-Patterns

### Hardcoding Config in Container Images

Baking environment-specific configuration (database URLs, API keys, feature flags) into the container image creates a tight coupling between the image and the environment. You end up building separate images for staging and production.

❌ **Anti-pattern — config baked into the image:**

```dockerfile
FROM node:20-slim
ENV DATABASE_URL=postgres://prod-db:5432/myapp
COPY . /app
CMD ["node", "/app/server.js"]
```

✅ **Better approach — externalize with ConfigMaps and environment variables:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  DATABASE_HOST: "db.prod.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
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
          image: myapp:1.2.0
          envFrom:
            - configMapRef:
                name: api-config
```

**Why it matters:**

- One image works across all environments (dev, staging, production).
- Configuration changes do not require a new image build.
- ConfigMaps can be managed independently by platform teams.

---

### Storing Secrets in ConfigMaps or Env Vars

ConfigMaps are stored unencrypted in etcd. Environment variables appear in `kubectl describe pod` output, process listings, and crash dumps. Neither is appropriate for passwords, API keys, or certificates.

❌ **Anti-pattern — secrets in a ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_PASSWORD: "supersecret123"  # Visible in plain text
```

✅ **Better approach — use Kubernetes Secrets with an external secret manager:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: c3VwZXJzZWNyZXQxMjM=  # base64-encoded, not encrypted

# Better still: use External Secrets Operator to sync from Vault / AWS Secrets Manager
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

**Why it matters:**

- Kubernetes Secrets are base64-encoded, not encrypted — enable encryption at rest in etcd.
- External secret managers (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager) provide audit logging, rotation, and access control.
- The External Secrets Operator bridges the gap, syncing external secrets into Kubernetes Secret objects automatically.

---

### Not Using Namespaces

Running everything in the `default` namespace is common in tutorials but dangerous in production. Without namespace boundaries, there is no resource isolation, no RBAC scoping, and no way to apply per-team quotas.

❌ **Anti-pattern — everything in `default`:**

```
$ kubectl get pods
NAME                           READY   STATUS
frontend-abc123                1/1     Running
backend-def456                 1/1     Running
monitoring-ghi789              1/1     Running
database-jkl012                1/1     Running
# All in the same namespace — no isolation
```

✅ **Better approach — namespace isolation strategy:**

```yaml
# Namespace per team or application boundary
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    team: payments
    env: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

**Namespace strategy guidelines:**

- Use namespaces to separate teams, environments, or application boundaries.
- Apply `ResourceQuota` and `LimitRange` per namespace.
- Scope RBAC `RoleBindings` to namespaces — avoid cluster-wide permissions.
- Use `NetworkPolicy` to restrict cross-namespace traffic.

---

### Mutable ConfigMaps Causing Drift

When you update a ConfigMap in place, existing Pods do not automatically restart. Some Pods pick up the new values (if using volume mounts with kubelet sync), while others keep the old values. This creates configuration drift that is extremely hard to debug.

❌ **Anti-pattern — mutating ConfigMaps in place:**

```bash
kubectl edit configmap api-config
# Some Pods see new values, some don't — silent drift
```

✅ **Better approach — immutable ConfigMaps with rollout:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config-v2  # Versioned name
immutable: true
data:
  LOG_LEVEL: "debug"
  FEATURE_FLAG_NEW_UI: "true"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  template:
    metadata:
      annotations:
        configmap-version: "v2"  # Change triggers rollout
    spec:
      containers:
        - name: api
          envFrom:
            - configMapRef:
                name: api-config-v2  # Point to new version
```

**Why it matters:**

- `immutable: true` prevents accidental edits and reduces etcd watch load.
- Changing the ConfigMap reference in the Deployment triggers a rolling update, so all Pods converge on the same configuration.
- Versioned names create an audit trail of configuration changes.

---

## Deployment Anti-Patterns

### Using Bare Pods

A bare Pod (created directly with `kind: Pod`) is not managed by any controller. If it crashes, no one restarts it. If the node fails, the Pod is lost. There is no rolling update, no scaling, no self-healing.

❌ **Anti-pattern — bare Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  containers:
    - name: api
      image: myapp:1.2.0
```

✅ **Better approach — use a Deployment (or StatefulSet for stateful workloads):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
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
          image: myapp:1.2.0
```

**Why it matters:**

- Deployments provide self-healing — crashed Pods are automatically replaced.
- Rolling updates enable zero-downtime deployments.
- Scaling is a single command: `kubectl scale deployment api-server --replicas=5`.
- Use `StatefulSet` when Pods need stable network identities or persistent storage (databases, message queues).
- Use `DaemonSet` when you need exactly one Pod per node (log collectors, monitoring agents).

---

### Not Using Health Checks

Without health checks, Kubernetes has no way to know whether your application is actually working. A Pod can be in `Running` state while the application inside is deadlocked, out of memory, or waiting on an unresponsive dependency.

❌ **Anti-pattern — no probes configured:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      # No probes — Kubernetes only knows if the process is alive
```

✅ **Better approach — configure liveness, readiness, and startup probes:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 2
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
        failureThreshold: 2
```

**Probe roles:**

| Probe     | Purpose                                          | Failure Action            |
|-----------|--------------------------------------------------|---------------------------|
| Startup   | Wait for slow-starting apps before other probes  | Restart container         |
| Liveness  | Detect deadlocks and unrecoverable states        | Restart container         |
| Readiness | Gate traffic until the app is ready to serve      | Remove from Service endpoints |

**Common mistake:** Using a liveness probe that checks dependencies (database, cache). If the dependency is down, Kubernetes restart-loops every Pod — making recovery impossible. Liveness probes should check the process itself; readiness probes should check dependencies.

---

### Single Replica in Production

Running a single replica means any disruption — a node drain, a rolling update, an OOM kill — causes downtime. There is no redundancy.

❌ **Anti-pattern — `replicas: 1` in production.**

✅ **Better approach — multiple replicas with topology spread:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
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
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-server
      containers:
        - name: api
          image: myapp:1.2.0
```

**Why it matters:**

- Multiple replicas survive node failures, zone outages, and rolling updates.
- `topologySpreadConstraints` distribute Pods across failure domains (zones, nodes).
- Combine with a `PodDisruptionBudget` (see next section) to guarantee minimum availability during voluntary disruptions.

---

### No Pod Disruption Budget

Without a PDB, cluster operations (node drains, upgrades, autoscaler scale-downs) can evict all replicas simultaneously. Kubernetes does not know how many Pods your service needs to stay healthy.

❌ **Anti-pattern — no PDB defined.**

✅ **Better approach — define a PodDisruptionBudget:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2          # At least 2 Pods must remain running
  # OR: maxUnavailable: 1  # At most 1 Pod can be down at a time
  selector:
    matchLabels:
      app: api-server
```

**Guidelines:**

- Use `minAvailable` when you have a known minimum for the service to function.
- Use `maxUnavailable` when you want to allow a percentage or fixed number to be disrupted.
- For a 3-replica Deployment, `maxUnavailable: 1` allows node drains to proceed one Pod at a time.
- Never set `minAvailable` equal to `replicas` — it blocks all voluntary disruptions, including cluster upgrades.

---

### Ignoring Graceful Shutdown

When Kubernetes terminates a Pod, it sends SIGTERM and waits for `terminationGracePeriodSeconds` (default: 30s) before sending SIGKILL. If your application does not handle SIGTERM, in-flight requests are dropped, database connections are left open, and data can be corrupted.

❌ **Anti-pattern — no shutdown handling:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      # No preStop hook, app ignores SIGTERM
      # terminationGracePeriodSeconds defaults to 30s
```

✅ **Better approach — handle SIGTERM and use preStop hooks:**

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      image: myapp:1.2.0
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
      # Application should handle SIGTERM:
      # - Stop accepting new connections
      # - Drain in-flight requests
      # - Close database connections
      # - Exit cleanly
```

**Why the `sleep 5` in preStop?**

The `preStop` hook runs before SIGTERM is sent. The 5-second sleep gives the Service endpoints time to update so that load balancers and other Pods stop sending traffic to the terminating Pod. Without this delay, requests arrive at a Pod that is already shutting down.

---

## Networking Anti-Patterns

### No NetworkPolicies

By default, every Pod in a Kubernetes cluster can talk to every other Pod — across namespaces, across teams, across trust boundaries. This flat network model means a compromised Pod can reach databases, internal APIs, and control-plane components.

❌ **Anti-pattern — no NetworkPolicies (implicit allow-all).**

✅ **Better approach — default deny with explicit allow rules:**

```yaml
# Default deny all ingress in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# Allow only frontend to reach the API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: team-payments
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

**Why it matters:**

- Default deny stops lateral movement — a compromised frontend Pod cannot reach the database.
- NetworkPolicies are namespaced, so each team can manage their own rules.
- Requires a CNI plugin that supports NetworkPolicies (Calico, Cilium, Antrea — the default kubenet does not).

---

### Using NodePort for Everything

`NodePort` opens a port (30000–32767) on every node in the cluster. This exposes services directly to the network, bypasses TLS termination, and does not support hostname-based routing.

❌ **Anti-pattern — NodePort for external traffic:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: api-server
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Exposed on every node
```

✅ **Better approach — use Ingress or LoadBalancer:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

**When NodePort is acceptable:**

- Local development clusters (Kind, Minikube).
- Environments where an external load balancer is not available.
- Never in production without a load balancer in front.

---

### Hardcoding Pod IPs

Pod IPs are ephemeral — they change on every restart, reschedule, or scale event. Hardcoding them in configuration creates brittle systems that break silently.

❌ **Anti-pattern — referencing Pod IPs directly:**

```yaml
env:
  - name: BACKEND_URL
    value: "http://10.244.1.5:8080"  # This IP will change
```

✅ **Better approach — use Kubernetes DNS and Services:**

```yaml
env:
  - name: BACKEND_URL
    value: "http://backend-service.team-payments.svc.cluster.local:8080"
```

**Kubernetes DNS patterns:**

| Pattern                                          | Resolves To                  |
|--------------------------------------------------|------------------------------|
| `<service>.<namespace>.svc.cluster.local`        | ClusterIP of the Service     |
| `<service>.<namespace>`                          | Same (short form)            |
| `<pod-ip-dashed>.<namespace>.pod.cluster.local`  | Individual Pod (rarely used) |

---

## Security Anti-Patterns

### Running Privileged Containers

A privileged container has almost unrestricted access to the host: all Linux capabilities, all devices, and the ability to modify the host kernel. It is effectively root on the node.

❌ **Anti-pattern — privileged container:**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.2.0
      securityContext:
        privileged: true  # Full host access
```

✅ **Better approach — drop all capabilities, use read-only filesystem:**

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.2.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
```

**Why it matters:**

- `capabilities.drop: [ALL]` removes every Linux capability (network raw sockets, mount, etc.).
- `readOnlyRootFilesystem` prevents attackers from writing malware or scripts to the container.
- `seccompProfile: RuntimeDefault` blocks dangerous syscalls like `unshare` and `mount`.
- If a workload needs specific capabilities, add only what is required (e.g., `NET_BIND_SERVICE` for ports < 1024).

---

### Using the Default ServiceAccount

Every namespace has a `default` ServiceAccount. If you do not specify a ServiceAccount for your Pod, it uses `default`. If anyone grants permissions to `default`, every Pod in the namespace inherits those permissions.

❌ **Anti-pattern — using the default ServiceAccount with broad permissions:**

```yaml
spec:
  # No serviceAccountName specified — uses "default"
  containers:
    - name: app
      image: myapp:1.2.0
```

✅ **Better approach — dedicated ServiceAccounts with minimal RBAC:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server-sa
  namespace: team-payments
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-server-role
  namespace: team-payments
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-binding
  namespace: team-payments
subjects:
  - kind: ServiceAccount
    name: api-server-sa
roleRef:
  kind: Role
  name: api-server-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: team-payments
spec:
  template:
    spec:
      serviceAccountName: api-server-sa
      containers:
        - name: api
          image: myapp:1.2.0
```

---

### Cluster-Wide ClusterRoleBindings

`ClusterRoleBinding` grants permissions across every namespace. Using it for application workloads violates the principle of least privilege and makes it impossible to scope access.

❌ **Anti-pattern — cluster-wide binding for an application:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-full-access
subjects:
  - kind: ServiceAccount
    name: api-server-sa
    namespace: team-payments
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Full access to everything
  apiGroup: rbac.authorization.k8s.io
```

✅ **Better approach — namespace-scoped RoleBindings:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-binding
  namespace: team-payments  # Scoped to one namespace
subjects:
  - kind: ServiceAccount
    name: api-server-sa
roleRef:
  kind: Role
  name: api-server-role  # Only the permissions this service needs
  apiGroup: rbac.authorization.k8s.io
```

**When ClusterRoleBinding is appropriate:**

- Cluster-scoped resources (nodes, namespaces, PersistentVolumes).
- Infrastructure components (ingress controllers, cert-manager, monitoring).
- Never for application workloads.

---

### No Pod Security Standards

Without Pod Security Standards (or their predecessor PodSecurityPolicies), there is nothing preventing users from deploying privileged containers, host-network Pods, or containers running as root.

❌ **Anti-pattern — no security enforcement at the namespace level.**

✅ **Better approach — apply the restricted Pod Security Standard:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Pod Security Standard levels:**

| Level       | Description                                                   |
|-------------|---------------------------------------------------------------|
| Privileged  | No restrictions (default)                                     |
| Baseline    | Blocks known privilege escalations (hostNetwork, privileged)  |
| Restricted  | Best practices — non-root, no capabilities, seccomp profile   |

Start with `warn` and `audit` in existing namespaces to identify violations, then move to `enforce` once all workloads comply.

---

### Mounting Service Account Tokens Unnecessarily

By default, Kubernetes mounts a service account token into every Pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Most application Pods never call the Kubernetes API, so this token is unnecessary attack surface.

❌ **Anti-pattern — token mounted by default:**

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.2.0
      # Token is auto-mounted at /var/run/secrets/kubernetes.io/serviceaccount/
      # If the container is compromised, the attacker can use it to call the K8s API
```

✅ **Better approach — disable auto-mounting:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server-sa
automountServiceAccountToken: false  # Disable at ServiceAccount level
---
spec:
  serviceAccountName: api-server-sa
  automountServiceAccountToken: false  # Also disable at Pod level
  containers:
    - name: api
      image: myapp:1.2.0
```

Only mount service account tokens for Pods that actually need to interact with the Kubernetes API (operators, controllers, CI runners). For everything else, disable it.

---

## Storage Anti-Patterns

### Using hostPath in Production

`hostPath` mounts a directory from the host node into the Pod. This creates a tight coupling to a specific node, breaks portability, and introduces security risks — a Pod can read arbitrary files from the host filesystem.

❌ **Anti-pattern — hostPath for application data:**

```yaml
spec:
  volumes:
    - name: data
      hostPath:
        path: /data/myapp  # Tied to this specific node
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /app/data
```

✅ **Better approach — PersistentVolumes with CSI drivers:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-encrypted
  resources:
    requests:
      storage: 10Gi
---
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /app/data
```

**Why it matters:**

- PVCs are portable — they work on any node and survive Pod rescheduling.
- CSI drivers provide encryption at rest, snapshots, and resizing.
- `hostPath` is acceptable for DaemonSets that need access to host-level data (log collectors, node monitors) — never for application data.

---

### Not Setting Storage Limits

Without storage quotas, a single team or application can consume all available storage in the cluster, starving other workloads.

❌ **Anti-pattern — no storage limits at the namespace level.**

✅ **Better approach — StorageClass with ResourceQuotas:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-payments
spec:
  hard:
    requests.storage: 100Gi
    persistentvolumeclaims: "10"
    gp3-encrypted.storageclass.storage.k8s.io/requests.storage: 100Gi
```

This ensures that the `team-payments` namespace can create at most 10 PVCs totaling no more than 100 Gi of storage.

---

### Using emptyDir for Persistent Data

`emptyDir` volumes are tied to the Pod lifecycle. When the Pod is deleted or rescheduled, the data is gone. Using `emptyDir` for data that needs to survive restarts is a recipe for data loss.

❌ **Anti-pattern — emptyDir for database storage:**

```yaml
spec:
  volumes:
    - name: db-data
      emptyDir: {}  # Data lost when Pod restarts
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
```

✅ **Better approach — PVC with appropriate StorageClass:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
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
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: db-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 50Gi
```

**When emptyDir is appropriate:**

- Temporary scratch space (image processing, file uploads in progress).
- Cache directories that can be regenerated.
- Sharing data between containers in the same Pod (sidecar pattern).
- Always set `sizeLimit` on `emptyDir` to prevent unbounded disk usage.

---

## Scaling Anti-Patterns

### Manual Scaling Only

Manually running `kubectl scale` does not respond to traffic spikes at 3 AM. By the time a human notices and reacts, your service has been down for minutes — or hours.

❌ **Anti-pattern — fixed replica count, manual intervention.**

✅ **Better approach — HorizontalPodAutoscaler (HPA):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

**Autoscaling options:**

| Tool  | Scales                          | Based On                          |
|-------|---------------------------------|-----------------------------------|
| HPA   | Pod replicas (horizontal)       | CPU, memory, custom metrics       |
| VPA   | Pod resource requests (vertical)| Historical usage                  |
| KEDA  | Pod replicas (event-driven)     | Queue depth, event count, custom  |

---

### No Cluster Autoscaler

Even with HPA, if the cluster runs out of node capacity, new Pods stay `Pending`. Without a cluster autoscaler, someone has to manually add nodes.

❌ **Anti-pattern — fixed-size node pool, no autoscaling.**

✅ **Better approach — enable cluster autoscaler or Karpenter:**

```yaml
# EKS managed node group with autoscaling
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production
  region: us-east-1
managedNodeGroups:
  - name: general
    instanceType: m6i.xlarge
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
```

**Karpenter** (AWS) takes a different approach — instead of scaling node groups, it provisions right-sized nodes on demand based on pending Pod requirements, selecting from a broad set of instance types automatically.

---

### Setting HPA min Equals max

Setting `minReplicas` equal to `maxReplicas` on an HPA defeats the purpose of autoscaling entirely. It is effectively a fixed replica count with extra complexity.

❌ **Anti-pattern — HPA that cannot scale:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 5
  maxReplicas: 5  # Cannot scale up or down
```

✅ **Better approach — meaningful min/max spread:**

```yaml
spec:
  minReplicas: 3   # Baseline for low traffic
  maxReplicas: 20  # Headroom for traffic spikes
```

**Guidelines for setting min and max:**

- `minReplicas` — the number of replicas needed for acceptable performance at baseline traffic.
- `maxReplicas` — the number of replicas needed for peak traffic, plus headroom.
- A good starting ratio is 3x–5x between min and max.
- Combine with cluster autoscaler so that nodes scale alongside Pods.

---

## Observability Anti-Patterns

### No Centralized Logging

Without centralized logging, debugging means `kubectl logs` on individual Pods — which disappears when the Pod is deleted. In a cluster with hundreds of Pods, this is unworkable.

❌ **Anti-pattern — relying on `kubectl logs` and local files.**

✅ **Better approach — structured logging with centralized aggregation:**

```yaml
# Application emits structured JSON logs to stdout
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "error",
  "message": "database connection failed",
  "service": "api-server",
  "trace_id": "abc123",
  "error": "connection refused",
  "host": "db.prod.svc.cluster.local:5432"
}
```

Deploy a log collector (Fluent Bit, Vector, or Fluentd) as a DaemonSet to ship logs to a central store (Elasticsearch, Loki, CloudWatch).

**Logging best practices:**

- Always log to stdout/stderr — never to files inside the container.
- Use structured JSON format for machine-parseable logs.
- Include correlation IDs (trace ID, request ID) for distributed tracing.
- Set appropriate log levels; avoid logging sensitive data.

---

### No Metrics or Alerts

Without metrics and alerts, you discover problems from user complaints, not dashboards. By the time a user reports an issue, it has likely been happening for a while.

❌ **Anti-pattern — no metrics, no dashboards, no alerts.**

✅ **Better approach — Prometheus metrics with Grafana dashboards and alerting:**

```yaml
# ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
  labels:
    team: payments
spec:
  selector:
    matchLabels:
      app: api-server
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
---
# PrometheusRule for alerting
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-server-alerts
spec:
  groups:
    - name: api-server
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5..",app="api-server"}[5m]))
            / sum(rate(http_requests_total{app="api-server"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "API server error rate above 5%"
```

**Key metrics to monitor:**

- **RED metrics**: Rate, Errors, Duration (for request-driven services).
- **USE metrics**: Utilization, Saturation, Errors (for resources like CPU, memory, disk).
- **Kubernetes-specific**: Pod restarts, OOMKills, pending Pods, node conditions.

---

### No Labels or Annotations Strategy

Without a consistent labeling strategy, you cannot filter, monitor, or manage resources effectively. Labels are the primary mechanism for selecting, grouping, and querying resources in Kubernetes.

❌ **Anti-pattern — inconsistent or missing labels:**

```yaml
metadata:
  name: my-app
  # No labels — invisible to selectors, monitoring, and cost allocation
```

✅ **Better approach — standard labels on every resource:**

```yaml
metadata:
  name: api-server
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: payments-platform
    app.kubernetes.io/managed-by: helm
    team: payments
    env: production
    cost-center: eng-payments
  annotations:
    owner: team-payments@company.com
    runbook: "https://wiki.internal/runbooks/api-server"
    on-call: "https://pagerduty.com/services/api-server"
```

**Recommended label taxonomy:**

| Label                              | Purpose                        | Example             |
|------------------------------------|--------------------------------|---------------------|
| `app.kubernetes.io/name`           | Application name               | `api-server`        |
| `app.kubernetes.io/version`        | Application version            | `1.2.0`             |
| `app.kubernetes.io/component`      | Component within the app       | `backend`           |
| `app.kubernetes.io/part-of`        | Higher-level application       | `payments-platform` |
| `team`                             | Owning team                    | `payments`          |
| `env`                              | Environment                    | `production`        |
| `cost-center`                      | Cost allocation                | `eng-payments`      |

---

## Organizational Anti-Patterns

### YAML Sprawl Without Structure

As clusters grow, teams accumulate hundreds of YAML files with no organization, no reuse, and no way to manage differences between environments. Copying and editing YAML files is error-prone and creates drift.

❌ **Anti-pattern — flat directory of duplicated YAML files:**

```
manifests/
├── api-deployment-dev.yaml
├── api-deployment-staging.yaml
├── api-deployment-prod.yaml
├── api-service-dev.yaml
├── api-service-staging.yaml
├── api-service-prod.yaml    # 90% identical to staging version
└── ... (hundreds more)
```

✅ **Better approach — Kustomize overlays or Helm charts with GitOps:**

```
manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml    # patches: replicas=1, resources=small
    ├── staging/
    │   └── kustomization.yaml    # patches: replicas=2
    └── prod/
        └── kustomization.yaml    # patches: replicas=3, resources=large
```

**Why it matters:**

- Base manifests define the common structure; overlays define per-environment differences.
- Changes to the base automatically propagate to all environments.
- GitOps tools (Argo CD, Flux) sync desired state from Git to the cluster, providing an audit trail and drift detection.

---

### No CI/CD for Manifests

Applying manifests manually with `kubectl apply` is error-prone, unauditable, and impossible to roll back cleanly.

❌ **Anti-pattern — manual `kubectl apply` from a laptop.**

✅ **Better approach — automated validation and deployment pipeline:**

```yaml
# Example CI pipeline steps
steps:
  - name: Lint manifests
    run: |
      kubeval --strict manifests/
      kube-linter lint manifests/

  - name: Validate with policy
    run: |
      conftest test manifests/ -p policy/

  - name: Dry-run apply
    run: |
      kubectl apply --dry-run=server -f manifests/

  - name: Deploy via Argo CD
    run: |
      argocd app sync api-server --prune
```

**Pipeline stages:**

| Stage        | Tool                      | Purpose                              |
|--------------|---------------------------|--------------------------------------|
| Lint         | kubeval, kube-linter      | Catch syntax and structural errors   |
| Policy       | Conftest, OPA, Kyverno    | Enforce organizational policies      |
| Dry-run      | kubectl --dry-run=server  | Validate against the live API server |
| Deploy       | Argo CD, Flux             | Git-driven deployment with rollback  |

---

### Everyone is cluster-admin

Giving every developer `cluster-admin` removes all guardrails. Anyone can delete namespaces, modify RBAC, access secrets, or disable security policies — accidentally or intentionally.

❌ **Anti-pattern — broad cluster-admin grants:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-developers-admin
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

✅ **Better approach — RBAC with least privilege:**

```yaml
# Developers get edit access to their team namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-payments
  namespace: team-payments
subjects:
  - kind: Group
    name: team-payments-devs
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit              # Built-in role: create/update/delete most resources
  apiGroup: rbac.authorization.k8s.io
---
# Read-only access to cluster-scoped resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-view
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view              # Built-in role: read-only access
  apiGroup: rbac.authorization.k8s.io
```

**RBAC guidelines:**

- Reserve `cluster-admin` for platform/SRE teams and break-glass scenarios.
- Use the built-in `view`, `edit`, and `admin` ClusterRoles as starting points.
- Scope access to specific namespaces with `RoleBindings`.
- Audit RBAC regularly — use `kubectl auth can-i --list` to verify effective permissions.

---

## Summary Table

| Anti-Pattern                          | Risk                                    | Solution                                   | Priority    |
|---------------------------------------|-----------------------------------------|--------------------------------------------|-------------|
| No resource requests/limits           | Noisy neighbors, unpredictable scheduling | Set requests and limits on all containers | 🔴 Critical |
| Using `:latest` tag                   | Non-reproducible deployments            | Pin to version tags or digests             | 🔴 Critical |
| Running as root                       | Container escape → host compromise      | `runAsNonRoot`, drop capabilities          | 🔴 Critical |
| No health checks                      | Silent failures, zombie Pods            | Liveness, readiness, startup probes        | 🔴 Critical |
| Running privileged containers         | Full host access from container         | Drop ALL capabilities, read-only FS       | 🔴 Critical |
| Secrets in ConfigMaps                 | Credentials exposed in plain text       | Use Secrets + external secret manager      | 🔴 Critical |
| Everyone is cluster-admin             | No access control, accidental deletion  | Namespace-scoped RBAC, least privilege     | 🔴 Critical |
| No NetworkPolicies                    | Lateral movement after compromise       | Default deny + explicit allow              | 🟠 High     |
| Using bare Pods                       | No self-healing, no scaling             | Deployments, StatefulSets, DaemonSets      | 🟠 High     |
| Single replica in production          | Downtime during any disruption          | Multiple replicas + PDB                    | 🟠 High     |
| No Pod Security Standards             | Privileged containers deployed freely   | Enforce restricted profile                 | 🟠 High     |
| Default ServiceAccount everywhere     | Over-permissioned Pods                  | Dedicated SAs with minimal RBAC            | 🟠 High     |
| Hardcoding config in images           | One image per environment               | ConfigMaps + environment variables         | 🟠 High     |
| No CI/CD for manifests                | Manual errors, no audit trail           | Automated lint, validate, deploy pipeline  | 🟠 High     |
| No PodDisruptionBudget                | All replicas evicted simultaneously     | Define PDB with minAvailable               | 🟡 Medium   |
| Over-provisioning resources           | Wasted capacity, inflated costs         | Right-size with metrics, use VPA           | 🟡 Medium   |
| Mutable ConfigMaps                    | Configuration drift across Pods         | Immutable ConfigMaps + rolling updates     | 🟡 Medium   |
| hostPath in production                | Node-tied, security risk                | PVCs with CSI drivers                      | 🟡 Medium   |
| Manual scaling only                   | Slow response to traffic spikes         | HPA, VPA, KEDA                             | 🟡 Medium   |
| No centralized logging                | Cannot debug across Pods                | Structured logs + aggregation              | 🟡 Medium   |
| No metrics or alerts                  | Problems discovered by users            | Prometheus + Grafana + alerting            | 🟡 Medium   |
| YAML sprawl                           | Duplication, drift between environments | Kustomize/Helm + GitOps                    | 🟡 Medium   |
| Using bloated base images             | Large attack surface, slow pulls        | Distroless, Alpine, multi-stage builds     | 🟡 Medium   |
| Not scanning images                   | CVEs deployed to production             | CI pipeline scanning + admission control   | 🟡 Medium   |
| NodePort for everything               | No TLS, no routing, exposed ports       | Ingress with TLS termination               | 🟡 Medium   |
| Hardcoding Pod IPs                    | Breakage on reschedule                  | DNS and Services                           | 🟡 Medium   |
| No labels strategy                    | Cannot filter, monitor, or allocate     | Standard label taxonomy                    | 🟢 Low      |
| emptyDir for persistent data          | Data loss on Pod restart                | PVCs with StatefulSets                     | 🟢 Low      |
| No storage limits                     | Unbounded storage consumption           | ResourceQuotas per namespace               | 🟢 Low      |
| No cluster autoscaler                 | Pods stuck in Pending                   | Cluster Autoscaler or Karpenter            | 🟢 Low      |
| HPA min equals max                    | Autoscaler does nothing                 | Meaningful min/max spread                  | 🟢 Low      |
| Ignoring graceful shutdown            | Dropped requests during deploys         | preStop hooks + SIGTERM handling           | 🟢 Low      |
| Mounting SA tokens unnecessarily      | Unnecessary API credentials in Pods     | `automountServiceAccountToken: false`      | 🟢 Low      |

---

## Next Steps

Continue building your Kubernetes knowledge with related topics:

- [Learning Path](LEARNING-PATH.md) — Guided progression through all Kubernetes topics
- [Best Practices and Patterns](10-BEST-PRACTICES-AND-PATTERNS.md) — The positive counterpart to this document
- [Kubernetes Overview](00-OVERVIEW.md) — Review the architecture and core concepts
- [Networking](09-NETWORKING.md) — Deep dive into cluster networking and policies
- [Manifests](06-MANIFESTS.md) — Writing and managing Kubernetes manifests
