# Kubernetes Deployment for Payment Systems

## Overview

Deploying payment services on Kubernetes requires special attention to security (PCI DSS), zero-downtime deployments, secret management, and resilience. This guide covers production-grade Kubernetes manifests, Helm charts, and operational patterns for .NET 8.0+ payment systems.

## Table of Contents

1. [Container Image Best Practices](#container-image-best-practices)
2. [Deployment Manifests](#deployment-manifests)
3. [Service and Ingress](#service-and-ingress)
4. [Secret Management](#secret-management)
5. [ConfigMaps and Environment Config](#configmaps-and-environment-config)
6. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
7. [Network Policies (PCI Compliance)](#network-policies-pci-compliance)
8. [Helm Chart Structure](#helm-chart-structure)
9. [CI/CD Pipeline](#cicd-pipeline)
10. [Operational Runbooks](#operational-runbooks)

---

## Container Image Best Practices

### Optimized Dockerfile

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app

# Security: run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1

COPY --from=build /app .

ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_EnableDiagnostics=0

EXPOSE 8080
ENTRYPOINT ["dotnet", "PaymentService.dll"]
```

### Image Security Scanning

```yaml
# GitHub Actions: scan image before push
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'payment-service:${{ github.sha }}'
    format: 'sarif'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

---

## Deployment Manifests

### Payment Service Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: payment-system
  labels:
    app: payment-service
    version: v1
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # Zero downtime
      maxSurge: 1
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: payment-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Anti-affinity: spread across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [payment-service]
                topologyKey: kubernetes.io/hostname

      containers:
        - name: payment-service
          image: registry.example.com/payment-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP

          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: connection-string
            - name: Stripe__SecretKey
              valueFrom:
                secretKeyRef:
                  name: stripe-credentials
                  key: secret-key
            - name: Stripe__WebhookSecret
              valueFrom:
                secretKeyRef:
                  name: stripe-credentials
                  key: webhook-secret

          envFrom:
            - configMapRef:
                name: payment-service-config

          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi

          # Readiness: only receive traffic when ready
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          # Liveness: restart if unhealthy
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3

          # Startup: give time for initial startup
          startupProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30  # 150s max startup time

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      # Graceful shutdown
      terminationGracePeriodSeconds: 60

      # Topology spread for multi-zone
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: payment-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-service
```

---

## Service and Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: payment-system
  labels:
    app: payment-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: payment-service

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-service
  namespace: payment-system
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit-connections: "10"
    nginx.ingress.kubernetes.io/rate-limit-rps: "50"
    nginx.ingress.kubernetes.io/proxy-body-size: "1m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - payments.example.com
      secretName: payment-tls
  rules:
    - host: payments.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

---

## Secret Management

### External Secrets Operator (Recommended)

```yaml
# Install External Secrets Operator
# helm install external-secrets external-secrets/external-secrets

# SecretStore (AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets

---
# ExternalSecret → pulls from AWS SM → creates K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: stripe-credentials
  namespace: payment-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: ClusterSecretStore
  target:
    name: stripe-credentials
    creationPolicy: Owner
  data:
    - secretKey: secret-key
      remoteRef:
        key: production/payment-service/stripe
        property: secret_key
    - secretKey: webhook-secret
      remoteRef:
        key: production/payment-service/stripe
        property: webhook_secret

---
# ExternalSecret for database credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-credentials
  namespace: payment-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: ClusterSecretStore
  target:
    name: payment-db-credentials
  data:
    - secretKey: connection-string
      remoteRef:
        key: production/payment-service/database
        property: connection_string
```

### Sealed Secrets (Alternative for GitOps)

```bash
# Encrypt secrets for Git storage
kubeseal --format yaml < stripe-secret.yaml > stripe-sealed-secret.yaml
```

---

## ConfigMaps and Environment Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: payment-system
data:
  ASPNETCORE_URLS: "http://+:8080"
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft: "Warning"
  PaymentProviders__DefaultProvider: "Stripe"
  PaymentProviders__FallbackProvider: "PayPal"
  CircuitBreaker__FailureThreshold: "5"
  CircuitBreaker__BreakDurationSeconds: "30"
  RateLimiting__PermitLimit: "100"
  RateLimiting__WindowSeconds: "60"
  OpenTelemetry__Endpoint: "http://otel-collector.observability:4317"
```

---

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: payment-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60             # Remove 1 pod per minute max
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60             # Add up to 4 pods per minute
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: payment_active           # Custom metric: active payments
        target:
          type: AverageValue
          averageValue: "30"             # Scale when >30 active payments per pod
```

---

## Network Policies (PCI Compliance)

```yaml
# Default deny all ingress/egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payment-system
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow ingress only from ingress controller and internal services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-ingress
  namespace: payment-system
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

---
# Allow egress to payment providers, database, DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-egress
  namespace: payment-system
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    # DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Database
    - to:
        - podSelector:
            matchLabels:
              app: payment-database
      ports:
        - protocol: TCP
          port: 5432
    # External: Stripe, PayPal (HTTPS)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

---

## Helm Chart Structure

```
payment-service-chart/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── networkpolicy.yaml
│   ├── serviceaccount.yaml
│   ├── external-secret.yaml
│   └── _helpers.tpl
```

### values.yaml (defaults)

```yaml
replicaCount: 3

image:
  repository: registry.example.com/payment-service
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilization: 60

ingress:
  enabled: true
  host: payments.example.com
  tlsSecretName: payment-tls
  rateLimitRps: 50

config:
  defaultProvider: Stripe
  fallbackProvider: PayPal
  logLevel: Information

secrets:
  provider: external-secrets  # or sealed-secrets
  awsSecretPrefix: production/payment-service
```

### Deploy Commands

```bash
# Deploy to staging
helm upgrade --install payment-service ./payment-service-chart \
  -f values-staging.yaml \
  --namespace payment-system \
  --set image.tag=$GIT_SHA

# Deploy to production
helm upgrade --install payment-service ./payment-service-chart \
  -f values-production.yaml \
  --namespace payment-system \
  --set image.tag=$RELEASE_TAG \
  --wait --timeout 5m
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Payment Service CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet test --configuration Release

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            registry.example.com/payment-service:${{ github.sha }}
            registry.example.com/payment-service:latest

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - run: |
          helm upgrade --install payment-service ./chart \
            -f values-staging.yaml \
            --set image.tag=${{ github.sha }} \
            --namespace payment-system \
            --wait --timeout 5m

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      - run: |
          helm upgrade --install payment-service ./chart \
            -f values-production.yaml \
            --set image.tag=${{ github.sha }} \
            --namespace payment-system \
            --wait --timeout 5m
```

---

## Operational Runbooks

### Rollback

```bash
# View rollout history
kubectl rollout history deployment/payment-service -n payment-system

# Rollback to previous version
kubectl rollout undo deployment/payment-service -n payment-system

# Rollback to specific revision
kubectl rollout undo deployment/payment-service -n payment-system --to-revision=3

# Or with Helm
helm rollback payment-service 1 -n payment-system
```

### Scaling

```bash
# Manual scale
kubectl scale deployment/payment-service --replicas=10 -n payment-system

# Check HPA status
kubectl get hpa payment-service-hpa -n payment-system
```

### Debugging

```bash
# Check pod status
kubectl get pods -l app=payment-service -n payment-system

# View logs
kubectl logs -l app=payment-service -n payment-system --tail=100 -f

# Exec into pod (read-only filesystem — use /tmp for scratch)
kubectl exec -it payment-service-abc123 -n payment-system -- sh

# Check resource usage
kubectl top pods -l app=payment-service -n payment-system
```

---

## Deployment Checklist

- [ ] Container runs as non-root with read-only filesystem
- [ ] Image scanned for vulnerabilities (Trivy, Snyk)
- [ ] Health checks configured (liveness, readiness, startup)
- [ ] Pod Disruption Budget set (minAvailable: 2+)
- [ ] Anti-affinity spreads pods across nodes/zones
- [ ] Network Policies restrict ingress/egress
- [ ] Secrets managed externally (AWS SM, Azure KV, HashiCorp Vault)
- [ ] HPA configured with appropriate thresholds
- [ ] Resource requests and limits set
- [ ] Graceful shutdown period configured (60s)
- [ ] Rolling update with maxUnavailable: 0
- [ ] Helm chart values separated per environment
- [ ] CI/CD pipeline with staging → production promotion

## Next Steps

- [Cloud-Native AWS](09-CLOUD-NATIVE-AWS.md)
- [Multi-Cloud Alternatives](10-MULTI-CLOUD-ALTERNATIVES.md)
- [Monitoring & Alerting](19-MONITORING-ALERTING.md)
- [Cost Optimization](21-COST-OPTIMIZATION.md)
