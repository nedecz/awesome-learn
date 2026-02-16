# Kubernetes Learning Path

A structured, self-paced training guide to mastering Kubernetes — from local development to production-grade deployments. Each phase builds on the previous one, progressing from core concepts to advanced operations.

> **Time Estimate:** 10–12 weeks at ~5 hours/week. Adjust pace to your experience level.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds on prior knowledge
2. **Read the linked documents** — they contain the detailed content
3. **Complete the exercises** — hands-on practice solidifies understanding
4. **Check yourself** — use the knowledge checks before moving on
5. **Build the capstone project** — ties everything together

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand Kubernetes architecture (control plane, nodes, kubelet, etcd)
- Learn core primitives: Pods, ReplicaSets, Deployments, Namespaces
- Grasp the declarative model and reconciliation loop
- Understand Service types and how cluster networking enables service discovery

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Architecture, control plane components, core objects, kubectl basics |
| 2 | [01-SERVICES](01-SERVICES.md) | ClusterIP, NodePort, LoadBalancer, ExternalName, Headless Services |

### Hands-On Exercise

**Install prerequisites:**

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**Create your first Pod:**

```bash
# Run an nginx Pod
kubectl run my-nginx --image=nginx:1.27 --port=80

# Inspect the Pod
kubectl get pods -o wide
kubectl describe pod my-nginx
kubectl logs my-nginx

# Open a shell inside the Pod
kubectl exec -it my-nginx -- /bin/bash
```

**Expose the Pod with a Service:**

```bash
# Create a ClusterIP service
kubectl expose pod my-nginx --port=80 --target-port=80 --name=nginx-svc

# Inspect the Service and its Endpoints
kubectl get svc nginx-svc
kubectl get endpoints nginx-svc
kubectl describe svc nginx-svc

# Test DNS resolution from another Pod
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://nginx-svc.default.svc.cluster.local
```

**Explore namespaces and resource listing:**

```bash
# List all namespaces
kubectl get namespaces

# List all resources in the kube-system namespace
kubectl get all -n kube-system

# List Pods with labels
kubectl get pods --show-labels
kubectl get pods -l app=my-nginx
```

### Knowledge Check

- [ ] What are the responsibilities of the API server, scheduler, and controller manager?
- [ ] How does a Deployment differ from a ReplicaSet, and why do you rarely create ReplicaSets directly?
- [ ] Explain the difference between ClusterIP and NodePort Services
- [ ] What happens when you run `kubectl apply -f manifest.yaml` — which components are involved?

---

## Phase 2: Local Development (Week 3)

### Learning Objectives

- Set up local Kubernetes clusters with kind and Minikube
- Write YAML manifests for Pods, Deployments, Services, and ConfigMaps
- Understand workload types: Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- Use Kustomize for manifest management

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-LOCAL-SETUP-KIND-MINIKUBE](05-LOCAL-SETUP-KIND-MINIKUBE.md) | kind vs. Minikube, multi-node clusters, local registries |
| 2 | [06-MANIFESTS](06-MANIFESTS.md) | YAML structure, Pods, Deployments, StatefulSets, DaemonSets, Jobs |

### Hands-On Exercises

**1. Create a multi-node kind cluster:**

```bash
# Install kind
go install sigs.k8s.io/kind@latest

# Create a cluster config
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# Create the cluster
kind create cluster --name dev --config kind-config.yaml
kubectl cluster-info --context kind-dev
kubectl get nodes
```

**2. Deploy a multi-tier application from scratch:**

Write the following manifests by hand (don't copy-paste — practice makes permanent):

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
        - name: frontend
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          env:
            - name: BACKEND_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: backend-url
```

```yaml
# app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  backend-url: "http://backend-svc:8080"
  log-level: "info"
```

```bash
# Apply the manifests
kubectl apply -f app-configmap.yaml
kubectl apply -f frontend-deployment.yaml

# Verify rollout
kubectl rollout status deployment/frontend
kubectl get pods -l tier=frontend

# Perform a rolling update
kubectl set image deployment/frontend frontend=nginx:1.27-alpine
kubectl rollout status deployment/frontend

# Roll back if needed
kubectl rollout undo deployment/frontend
kubectl rollout history deployment/frontend
```

**3. Explore workload types:**

```bash
# Create a Job that runs to completion
kubectl create job pi-calc --image=perl:5.34 \
  -- perl -Mbignum=bpi -wle "print bpi(2000)"
kubectl get jobs
kubectl logs job/pi-calc

# Create a CronJob
kubectl create cronjob hello --image=busybox:1.36 \
  --schedule="*/5 * * * *" -- echo "Hello from CronJob"
kubectl get cronjobs
```

**4. Use Kustomize for environment overlays:**

```bash
# Create a kustomization.yaml
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - frontend-deployment.yaml
  - app-configmap.yaml
namePrefix: dev-
commonLabels:
  environment: development
EOF

# Preview the output
kubectl kustomize .

# Apply with Kustomize
kubectl apply -k .
```

### Knowledge Check

- [ ] When would you choose kind over Minikube for local development?
- [ ] What is the difference between `kubectl apply` and `kubectl create`?
- [ ] How does a StatefulSet differ from a Deployment, and when would you use each?
- [ ] What problem does Kustomize solve compared to plain YAML manifests?

---

## Phase 3: Networking Deep Dive (Week 4–5)

### Learning Objectives

- Understand the Kubernetes networking model and CNI plugins
- Implement NetworkPolicies for traffic segmentation
- Configure Ingress controllers and the Gateway API
- Understand DNS resolution and Service Mesh concepts

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-NETWORKING](09-NETWORKING.md) | CNI, NetworkPolicies, Ingress, Gateway API, Service Mesh |
| 2 | [01-SERVICES](01-SERVICES.md) | Revisit — focus on headless Services, endpoint slices, DNS |

### Hands-On Exercises

**1. Install an Ingress controller:**

```bash
# Install NGINX Ingress Controller on kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

**2. Create an Ingress resource with host-based routing:**

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 8080
  tls:
    - hosts:
        - app.local
      secretName: app-tls-secret
```

```bash
# Generate a self-signed TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=app.local"

# Create the TLS Secret
kubectl create secret tls app-tls-secret --cert=tls.crt --key=tls.key

# Apply Ingress
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress app-ingress
```

**3. Write NetworkPolicies for segmentation:**

```yaml
# deny-all.yaml — Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress

---
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
          protocol: TCP
```

```bash
kubectl apply -f deny-all.yaml
kubectl apply -f allow-frontend-to-backend.yaml
kubectl get networkpolicies
```

**4. Explore DNS resolution:**

```bash
# Launch a debug Pod
kubectl run dns-debug --image=busybox:1.36 --rm -it --restart=Never -- sh

# Inside the Pod
nslookup kubernetes.default.svc.cluster.local
nslookup frontend.default.svc.cluster.local
cat /etc/resolv.conf
```

### Knowledge Check

- [ ] What are the three fundamental requirements of the Kubernetes networking model?
- [ ] How does a NetworkPolicy's empty `podSelector: {}` differ from a specific label selector?
- [ ] What is the difference between Ingress and the Gateway API?
- [ ] How does CoreDNS resolve Service names inside a cluster?

---

## Phase 4: Cloud Providers (Week 6–7)

### Learning Objectives

- Deploy managed Kubernetes clusters on AWS EKS, Oracle OKE, and Azure AKS
- Configure cloud IAM integration with Kubernetes RBAC
- Understand cloud-specific networking (VPCs, load balancers, CNI plugins)
- Compare managed Kubernetes offerings across providers

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-CLOUD-SERVICES-AWS-EKS](02-CLOUD-SERVICES-AWS-EKS.md) | EKS setup, IAM roles for service accounts, VPC CNI |
| 2 | [03-CLOUD-SERVICES-OKE](03-CLOUD-SERVICES-OKE.md) | OKE provisioning, OCI networking, node pools |
| 3 | [04-CLOUD-SERVICES-AKS](04-CLOUD-SERVICES-AKS.md) | AKS setup, Azure AD integration, Azure CNI |

### Hands-On Exercises

**1. Deploy a cluster on your preferred cloud provider:**

```bash
# --- Option A: AWS EKS with eksctl ---
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --managed

# Verify
kubectl get nodes
kubectl cluster-info

# --- Option B: Azure AKS with az CLI ---
az group create --name k8s-rg --location eastus
az aks create \
  --resource-group k8s-rg \
  --name my-cluster \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys

az aks get-credentials --resource-group k8s-rg --name my-cluster
kubectl get nodes
```

**2. Configure RBAC with cloud IAM:**

```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create namespace dev
kubectl apply -f developer-role.yaml

# Verify permissions
kubectl auth can-i create deployments --namespace dev --as developer@example.com
kubectl auth can-i delete namespaces --as developer@example.com
```

**3. Compare managed Kubernetes offerings:**

Create a comparison table covering:

| Feature | EKS | OKE | AKS |
|---------|-----|-----|-----|
| Control plane cost | | | |
| Default CNI plugin | | | |
| IAM integration | | | |
| Max nodes per cluster | | | |
| GPU support | | | |
| Serverless option | | | |

Fill this in from the three cloud documents — this exercise builds breadth across providers.

### Knowledge Check

- [ ] What is the difference between IAM roles for service accounts (IRSA) on EKS and Workload Identity on AKS?
- [ ] Why do cloud providers charge for the Kubernetes control plane separately from worker nodes?
- [ ] How does the VPC CNI plugin on EKS differ from overlay-based CNI plugins?
- [ ] What are the trade-offs between managed node groups and self-managed nodes?

---

## Phase 5: Advanced Kubernetes (Week 8–9)

### Learning Objectives

- Define Custom Resource Definitions (CRDs) to extend the Kubernetes API
- Understand the operator pattern and controller reconciliation loop
- Build a simple controller that watches and reacts to custom resources
- Deploy and use community operators (cert-manager, Prometheus Operator)

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-CRDS-AND-OPERATORS](07-CRDS-AND-OPERATORS.md) | CRD schema, operator pattern, Operator SDK, OLM |
| 2 | [08-CONTROLLERS](08-CONTROLLERS.md) | Built-in controllers, control loops, custom controller development |

### Hands-On Exercises

**1. Create a Custom Resource Definition:**

```yaml
# website-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.apps.example.com
spec:
  group: apps.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["image", "replicas"]
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
            status:
              type: object
              properties:
                availableReplicas:
                  type: integer
      subresources:
        status: {}
  scope: Namespaced
  names:
    plural: websites
    singular: website
    kind: Website
    shortNames:
      - ws
```

```bash
kubectl apply -f website-crd.yaml
kubectl get crds | grep website

# Create a custom resource instance
cat <<EOF | kubectl apply -f -
apiVersion: apps.example.com/v1
kind: Website
metadata:
  name: my-blog
spec:
  image: nginx:1.27
  replicas: 3
EOF

kubectl get websites
kubectl describe website my-blog
```

**2. Deploy cert-manager as a real-world operator example:**

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for cert-manager Pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager \
  -n cert-manager --timeout=120s

# Verify CRDs installed by the operator
kubectl get crds | grep cert-manager

# Create a self-signed ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

kubectl get clusterissuers
```

**3. Explore the controller reconciliation model:**

```bash
# Watch the Deployment controller in action
kubectl create deployment test-controller --image=nginx:1.27 --replicas=3

# Delete a Pod and observe the controller recreate it
kubectl get pods -l app=test-controller -w &
kubectl delete pod $(kubectl get pods -l app=test-controller -o jsonpath='{.items[0].metadata.name}')

# The controller detects desired vs. actual state mismatch and creates a new Pod
```

### Knowledge Check

- [ ] What is the difference between a CRD and a built-in Kubernetes resource?
- [ ] How does the reconciliation loop work in a Kubernetes controller?
- [ ] When should you build a custom operator versus using Helm charts?
- [ ] What role does the Operator Lifecycle Manager (OLM) play?

---

## Phase 6: Production Readiness (Week 10–12)

### Learning Objectives

- Apply Kubernetes best practices for resource management, security, and reliability
- Identify and fix common anti-patterns in cluster configuration
- Implement health checks, Pod Disruption Budgets, and autoscaling
- Set up monitoring, resource quotas, and security policies

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [10-BEST-PRACTICES-AND-PATTERNS](10-BEST-PRACTICES-AND-PATTERNS.md) | Resource limits, health probes, security contexts, labeling |
| 2 | [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common mistakes, over-provisioning, missing probes, running as root |

### Hands-On Exercises

**1. Security audit a namespace:**

```bash
# Check for Pods running as root
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].securityContext}{"\n"}{end}'

# Check for missing resource limits
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources}{"\n"}{end}'

# Check for missing health probes
kubectl get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].livenessProbe}{"\n"}{end}'
```

**2. Implement resource quotas and limit ranges:**

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"

---
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "1"
        memory: 1Gi
```

```bash
kubectl apply -f resource-quota.yaml
kubectl apply -f limit-range.yaml
kubectl describe quota team-quota -n dev
kubectl describe limitrange default-limits -n dev
```

**3. Configure health probes and Pod Disruption Budgets:**

```yaml
# production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

---
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

**4. Set up Horizontal Pod Autoscaling:**

```bash
# Install metrics-server (required for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Create an HPA
kubectl autoscale deployment web-app --cpu-percent=70 --min=2 --max=10

# Check HPA status
kubectl get hpa web-app
kubectl describe hpa web-app

# Generate load to trigger scaling
kubectl run load-gen --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://web-app; done"

# Watch scaling events
kubectl get hpa web-app -w
```

**5. Review and fix anti-patterns:**

Audit the following manifest and identify at least five anti-patterns:

```yaml
# broken-deployment.yaml — How many problems can you find?
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080
```

<details>
<summary>Anti-patterns in this manifest (click to reveal)</summary>

1. **`latest` tag** — Not pinning image versions makes rollbacks impossible
2. **Single replica** — No redundancy; a single Pod failure causes downtime
3. **No resource requests/limits** — Risks noisy-neighbor issues and OOMKill
4. **No health probes** — Kubernetes cannot detect if the app is stuck
5. **No security context** — Container may run as root
6. **No PDB** — Node drain will take the only replica offline
7. **No labels beyond `app`** — Missing `version`, `team`, or `environment` labels

</details>

### Knowledge Check

- [ ] Why should you always set both resource requests and limits on containers?
- [ ] What is the difference between liveness, readiness, and startup probes?
- [ ] How does a PodDisruptionBudget protect your application during cluster maintenance?
- [ ] Why is using `latest` as an image tag considered an anti-pattern?
- [ ] What is the purpose of `runAsNonRoot` in a security context?

---

## Capstone Project

Design and deploy a **production-grade multi-tier application** that demonstrates everything you've learned across all six phases.

### Architecture

```
                          ┌──────────────┐
                          │   Ingress    │
                          │  (TLS/HTTPS) │
                          └──────┬───────┘
                     ┌───────────┴───────────┐
                     ▼                       ▼
              ┌─────────────┐         ┌─────────────┐
              │  Frontend   │         │  API Server  │
              │  (nginx)    │         │  (app:8080)  │
              │  2 replicas │         │  3 replicas  │
              └─────────────┘         └──────┬───────┘
                                             │
                                      ┌──────┴───────┐
                                      ▼              ▼
                               ┌───────────┐  ┌───────────┐
                               │  Redis     │  │ PostgreSQL│
                               │ (cache)    │  │ (primary) │
                               └───────────┘  └───────────┘
```

### Requirements

Your deployment must include the following:

| Requirement | What to Implement |
|-------------|-------------------|
| Multi-container Pods | API server with sidecar logging container and init container for DB migration |
| Services & Ingress | ClusterIP for backend, Ingress with TLS termination and path-based routing |
| NetworkPolicies | Default deny, allow frontend→API, allow API→database, deny database→frontend |
| Resource management | Requests and limits on all containers, ResourceQuota for the namespace |
| Autoscaling | HPA on the API Deployment (target 70% CPU, min 2, max 10) |
| Health checks | Liveness, readiness, and startup probes on all application containers |
| Disruption budgets | PDB with `minAvailable: 2` on API and frontend Deployments |
| RBAC | Separate ServiceAccounts per workload, least-privilege Roles |
| Configuration | ConfigMaps for app config, Secrets for database credentials |
| GitOps readiness | All manifests in a Kustomize structure with base and overlay directories |

### Project Structure

```
capstone/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── pdb.yaml
│   ├── api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── serviceaccount.yaml
│   ├── database/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   └── secret.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── ingress.yaml
│   ├── network-policies.yaml
│   ├── resource-quota.yaml
│   └── rbac.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── reduce-replicas.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches/
            └── increase-resources.yaml
```

### Deployment Commands

```bash
# Create a kind cluster for the capstone
kind create cluster --name capstone --config kind-config.yaml

# Deploy with the dev overlay
kubectl apply -k capstone/overlays/dev/

# Verify all resources
kubectl get all -n capstone
kubectl get ingress -n capstone
kubectl get networkpolicies -n capstone
kubectl get hpa -n capstone
kubectl get pdb -n capstone

# Run a connectivity test
kubectl run smoke-test -n capstone --image=curlimages/curl --rm -it --restart=Never \
  -- curl -s http://api-svc:8080/health

# Verify NetworkPolicy enforcement
kubectl run blocked-test -n capstone --image=curlimages/curl --rm -it --restart=Never \
  -l tier=unauthorized \
  -- curl -s --connect-timeout 5 http://database-svc:5432 || echo "Blocked as expected"

# Test autoscaling
kubectl run load-gen -n capstone --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://api-svc:8080; done"
kubectl get hpa -n capstone -w
```

### Evaluation Criteria

| Area | What to Verify |
|------|---------------|
| Architecture | Clear separation of frontend, API, cache, and database tiers |
| Security | Non-root containers, least-privilege RBAC, NetworkPolicies, Secrets for credentials |
| Reliability | Health probes on all apps, PDBs, multiple replicas, HPA configured |
| Resource management | Requests and limits set, ResourceQuota enforced, LimitRange defaults |
| Networking | Ingress with TLS, path-based routing, DNS resolution working |
| Configuration | ConfigMaps and Secrets used properly, Kustomize overlays for environments |
| Observability | Structured logging sidecar, health endpoints, metrics endpoint for HPA |

---

## Recommended Tools

| Tool | Purpose |
|------|---------|
| [kubectl](https://kubernetes.io/docs/reference/kubectl/) | Primary CLI for Kubernetes |
| [kind](https://kind.sigs.k8s.io/) | Local multi-node clusters in Docker |
| [Minikube](https://minikube.sigs.k8s.io/) | Local single-node clusters with add-ons |
| [k9s](https://k9scli.io/) | Terminal-based Kubernetes dashboard |
| [Lens](https://k8slens.dev/) | GUI-based Kubernetes IDE |
| [Kustomize](https://kustomize.io/) | Template-free manifest customization |
| [Helm](https://helm.sh/) | Package manager for Kubernetes |
| [stern](https://github.com/stern/stern) | Multi-pod log tailing |
| [kubectx/kubens](https://github.com/ahmetb/kubectx) | Fast context and namespace switching |
| [Popeye](https://github.com/derailed/popeye) | Cluster sanitizer and best-practice scanner |

---

## Quick Reference: Document Map

| # | Document | Phase |
|---|----------|-------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 |
| 01 | [SERVICES](01-SERVICES.md) | 1, 3 |
| 02 | [CLOUD-SERVICES-AWS-EKS](02-CLOUD-SERVICES-AWS-EKS.md) | 4 |
| 03 | [CLOUD-SERVICES-OKE](03-CLOUD-SERVICES-OKE.md) | 4 |
| 04 | [CLOUD-SERVICES-AKS](04-CLOUD-SERVICES-AKS.md) | 4 |
| 05 | [LOCAL-SETUP-KIND-MINIKUBE](05-LOCAL-SETUP-KIND-MINIKUBE.md) | 2 |
| 06 | [MANIFESTS](06-MANIFESTS.md) | 2 |
| 07 | [CRDS-AND-OPERATORS](07-CRDS-AND-OPERATORS.md) | 5 |
| 08 | [CONTROLLERS](08-CONTROLLERS.md) | 5 |
| 09 | [NETWORKING](09-NETWORKING.md) | 3 |
| 10 | [BEST-PRACTICES-AND-PATTERNS](10-BEST-PRACTICES-AND-PATTERNS.md) | 6 |
| 11 | [ANTI-PATTERNS](11-ANTI-PATTERNS.md) | 6 |
