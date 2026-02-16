# Kubernetes Services

## Table of Contents

1. [Overview](#overview)
2. [Why Services Exist](#why-services-exist)
3. [Service Types](#service-types)
   - [ClusterIP](#clusterip)
   - [NodePort](#nodeport)
   - [LoadBalancer](#loadbalancer)
   - [ExternalName](#externalname)
   - [Headless Services](#headless-services)
4. [Service Discovery](#service-discovery)
5. [Endpoints and EndpointSlices](#endpoints-and-endpointslices)
6. [Session Affinity](#session-affinity)
7. [Multi-Port Services](#multi-port-services)
8. [Traffic Policies](#traffic-policies)
9. [Troubleshooting Services](#troubleshooting-services)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

## Overview

This document covers Kubernetes **Services** — the abstraction layer that provides stable networking and service discovery for Pods. Services decouple consumers from the details of which Pods are running, where they are scheduled, and how many replicas exist.

### Target Audience

- Developers exposing applications within or outside a cluster
- Platform engineers designing internal networking topologies
- SREs debugging connectivity and load balancing issues

### Scope

- Service types and when to use each one
- DNS-based and environment-variable-based service discovery
- Endpoints, EndpointSlices, and how traffic reaches Pods
- Traffic policies, session affinity, and multi-port configuration
- Practical troubleshooting and best practices

## Why Services Exist

Pods are ephemeral. They are created, destroyed, and rescheduled constantly. Each Pod receives a unique IP address, but that IP is not stable — it changes every time a Pod is replaced. Services solve this by providing a **stable virtual IP** (ClusterIP) and a **DNS name** that remain constant regardless of which Pods are backing the Service.

```
Without Services                     With a Service
┌──────────────┐                    ┌──────────────┐
│   Consumer   │                    │   Consumer   │
└──────┬───────┘                    └──────┬───────┘
       │ Which Pod IP?                     │ my-svc:8080
       │ 10.244.1.5? Gone!                 │ (stable)
       │ 10.244.2.9? Gone!          ┌─────▼──────┐
       │ 10.244.3.7? Maybe?         │  Service    │
       ▼                            │  ClusterIP  │
  ╳ Unreliable                      │ 10.96.0.50  │
                                    └──┬───┬───┬──┘
                                       │   │   │
                                    ┌──▼┐┌─▼─┐┌▼──┐
                                    │Pod││Pod ││Pod│
                                    └───┘└────┘└───┘
```

### Key Problems Services Solve

| Problem | How Services Help |
|---|---|
| **Pod IP instability** | Provides a stable ClusterIP that never changes |
| **Service discovery** | Registers a DNS record (`<svc>.<ns>.svc.cluster.local`) |
| **Load balancing** | Distributes traffic across all healthy Pod replicas |
| **Decoupling** | Consumers reference the Service name, not individual Pods |
| **External access** | NodePort and LoadBalancer types expose apps outside the cluster |

## Service Types

Kubernetes supports several Service types, each designed for different access patterns.

| Type | Scope | Use Case |
|---|---|---|
| **ClusterIP** | Internal only | Default; inter-service communication |
| **NodePort** | External via node IP | Dev/test or on-prem without a load balancer |
| **LoadBalancer** | External via cloud LB | Production-grade external access in cloud environments |
| **ExternalName** | DNS alias | Mapping to external services outside the cluster |
| **Headless** | Direct Pod IPs | StatefulSets, custom discovery, peer-to-peer |

### ClusterIP

The default Service type. It assigns a cluster-internal IP address that is only reachable from within the cluster. Other Pods and Services can reach it, but external clients cannot.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: production
  labels:
    app: backend
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
    - name: http
      protocol: TCP
      port: 80          # Port the Service listens on
      targetPort: 8080  # Port on the Pod container
```

```bash
# Create the Service
kubectl apply -f backend-svc.yaml

# Verify ClusterIP assignment
kubectl get svc backend-api -n production

# Test connectivity from another Pod
kubectl run curl-test --rm -it --image=curlimages/curl -- \
  curl -s http://backend-api.production.svc.cluster.local/health
```

```
Traffic Flow — ClusterIP

  ┌────────────┐
  │ Frontend   │
  │ Pod        │
  └─────┬──────┘
        │  http://backend-api:80
        ▼
  ┌─────────────┐       ┌──────────┐
  │  kube-proxy │──────▶│ iptables │
  │  (per node) │       │ / IPVS   │
  └─────────────┘       └────┬─────┘
                              │ DNAT to Pod IP
                   ┌──────────┼──────────┐
                   ▼          ▼          ▼
              ┌────────┐ ┌────────┐ ┌────────┐
              │ Pod :  │ │ Pod :  │ │ Pod :  │
              │  8080  │ │  8080  │ │  8080  │
              └────────┘ └────────┘ └────────┘
```

### NodePort

Extends ClusterIP by opening a static port (default range 30000–32767) on **every node** in the cluster. External clients can reach the Service by connecting to `<NodeIP>:<NodePort>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
  namespace: production
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 31080  # Optional; auto-assigned if omitted
```

```bash
# Get the assigned NodePort
kubectl get svc web-frontend -n production -o jsonpath='{.spec.ports[0].nodePort}'

# Access from outside the cluster
curl http://<NODE_IP>:31080/
```

```
Traffic Flow — NodePort

  External Client
        │
        │  http://<NodeIP>:31080
        ▼
  ┌──────────────┐
  │  Node        │
  │  (port 31080)│
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │  ClusterIP   │  (NodePort builds on ClusterIP)
  │  10.96.x.x   │
  └──────┬───────┘
         │
    ┌────┼────┐
    ▼    ▼    ▼
  ┌───┐┌───┐┌───┐
  │Pod││Pod││Pod│
  └───┘└───┘└───┘
```

### LoadBalancer

Extends NodePort by provisioning an **external load balancer** through the cloud provider (AWS ELB, GCP Network LB, Azure LB). This is the standard way to expose services to the internet in cloud environments.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-api
  namespace: production
  annotations:
    # Cloud-specific annotations (AWS example)
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
```

```bash
# Wait for the external IP to be provisioned
kubectl get svc public-api -n production -w

# Once EXTERNAL-IP is assigned
curl https://<EXTERNAL-IP>/api/v1/status
```

```
Traffic Flow — LoadBalancer

  Internet Client
        │
        ▼
  ┌──────────────────┐
  │  Cloud Load      │
  │  Balancer        │
  │  (external IP)   │
  └───────┬──────────┘
          │
  ┌───────▼──────────┐
  │  NodePort        │  (LB routes to NodePorts)
  │  (all nodes)     │
  └───────┬──────────┘
          │
  ┌───────▼──────────┐
  │  ClusterIP       │
  └───────┬──────────┘
          │
     ┌────┼────┐
     ▼    ▼    ▼
   ┌───┐┌───┐┌───┐
   │Pod││Pod││Pod│
   └───┘└───┘└───┘
```

### ExternalName

Maps a Service to an external DNS name via a **CNAME record**. No proxying occurs — kube-dns or CoreDNS returns the CNAME directly. This is useful for integrating with services outside the cluster (managed databases, third-party APIs) while keeping a consistent internal naming scheme.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.example.cloud-provider.com
  # No selector, no ports, no ClusterIP
```

```bash
# Pods can now reference the external database by Service name
kubectl run db-test --rm -it --image=busybox -- \
  nslookup external-db.production.svc.cluster.local

# Returns CNAME: mydb.example.cloud-provider.com
```

> **Note:** ExternalName Services do not support ports, selectors, or proxying. They only provide DNS-level aliasing.

### Headless Services

A headless Service is created by setting `clusterIP: None`. Instead of returning a single virtual IP, DNS queries return the **individual Pod IPs** directly. This is essential for StatefulSets where each Pod has a stable identity.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
  namespace: data
  labels:
    app: cassandra
spec:
  clusterIP: None  # Makes it headless
  selector:
    app: cassandra
  ports:
    - name: cql
      port: 9042
      targetPort: 9042
```

```bash
# DNS returns individual Pod IPs instead of a single ClusterIP
kubectl run dns-test --rm -it --image=busybox -- \
  nslookup cassandra.data.svc.cluster.local

# For StatefulSets, each Pod gets a stable DNS entry:
#   cassandra-0.cassandra.data.svc.cluster.local
#   cassandra-1.cassandra.data.svc.cluster.local
#   cassandra-2.cassandra.data.svc.cluster.local
```

```
Headless Service — DNS Resolution

  DNS Query: cassandra.data.svc.cluster.local

  ┌────────────┐
  │  CoreDNS   │
  └─────┬──────┘
        │ Returns A records (not a ClusterIP):
        │   10.244.1.10
        │   10.244.2.11
        │   10.244.3.12
        ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ cassandra-0  │     │ cassandra-1  │     │ cassandra-2  │
  │ 10.244.1.10  │     │ 10.244.2.11  │     │ 10.244.3.12  │
  └──────────────┘     └──────────────┘     └──────────────┘
```

## Service Discovery

Kubernetes provides two mechanisms for Pods to discover Services.

### DNS-Based Discovery (Recommended)

CoreDNS (or kube-dns) automatically creates DNS records for every Service. This is the preferred discovery method.

| Record Type | Format | Example |
|---|---|---|
| **A/AAAA record** | `<svc>.<ns>.svc.cluster.local` | `backend-api.production.svc.cluster.local` |
| **SRV record** | `_<port>._<proto>.<svc>.<ns>.svc.cluster.local` | `_http._tcp.backend-api.production.svc.cluster.local` |
| **Pod A record** (headless) | `<pod>.<svc>.<ns>.svc.cluster.local` | `cassandra-0.cassandra.data.svc.cluster.local` |

```bash
# Full DNS name
curl http://backend-api.production.svc.cluster.local/health

# Short name (same namespace)
curl http://backend-api/health

# Cross-namespace (namespace-qualified)
curl http://backend-api.production/health
```

### Environment Variable Discovery

When a Pod starts, Kubernetes injects environment variables for every active Service in the **same namespace**. This method is older and less flexible than DNS.

```bash
# For a Service named "backend-api" on port 80, these variables are set:
#   BACKEND_API_SERVICE_HOST=10.96.0.50
#   BACKEND_API_SERVICE_PORT=80
#   BACKEND_API_PORT=tcp://10.96.0.50:80

# View injected variables in a running Pod
kubectl exec <pod-name> -- env | grep BACKEND_API
```

> **Caveat:** Environment variables are set only at Pod startup. Services created after the Pod starts will not appear. Prefer DNS-based discovery.

## Endpoints and EndpointSlices

When a Service has a selector, Kubernetes automatically creates **Endpoints** (and **EndpointSlices**) resources that list the IP addresses of matching Pods.

### Endpoints

```bash
# View the Endpoints for a Service
kubectl get endpoints backend-api -n production

# Detailed output showing Pod IPs and ports
kubectl describe endpoints backend-api -n production
```

### EndpointSlices

EndpointSlices are the modern replacement for Endpoints, designed to scale better in large clusters. Each EndpointSlice holds up to 100 endpoints by default, and Kubernetes creates multiple slices as needed.

```bash
# List EndpointSlices for a Service
kubectl get endpointslices -n production -l kubernetes.io/service-name=backend-api

# Detailed view
kubectl describe endpointslice -n production -l kubernetes.io/service-name=backend-api
```

```
Selector ──▶ Endpoints/EndpointSlices ──▶ Pod IPs

  ┌──────────────┐     ┌──────────────────┐     ┌──────────┐
  │  Service     │     │  EndpointSlice   │     │ Pod      │
  │  selector:   │────▶│  - 10.244.1.5    │────▶│ 10.244.1.5
  │   app: web   │     │  - 10.244.2.9    │     ├──────────┤
  └──────────────┘     │  - 10.244.3.7    │     │ Pod      │
                       └──────────────────┘     │ 10.244.2.9
                                                ├──────────┤
                                                │ Pod      │
                                                │ 10.244.3.7
                                                └──────────┘
```

### Services Without Selectors

You can create a Service without a selector and manually manage Endpoints. This is useful for pointing to external systems or non-Kubernetes backends.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-backend
  namespace: production
spec:
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: legacy-backend  # Must match Service name
  namespace: production
subsets:
  - addresses:
      - ip: 192.168.1.100
      - ip: 192.168.1.101
    ports:
      - port: 8080
```

## Session Affinity

By default, kube-proxy distributes requests across all healthy Pods randomly. **Session affinity** pins a client to the same Pod for the duration of a session.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-web
spec:
  type: ClusterIP
  selector:
    app: web
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # Sticky for 1 hour
  ports:
    - port: 80
      targetPort: 8080
```

| Setting | Options | Description |
|---|---|---|
| `sessionAffinity` | `None` (default), `ClientIP` | Pin traffic based on client IP |
| `sessionAffinityConfig.clientIP.timeoutSeconds` | 0–86400 | How long the affinity persists (default: 10800 / 3 hours) |

> **Note:** Kubernetes Services only support `ClientIP` affinity. For cookie-based session affinity, use an Ingress controller.

## Multi-Port Services

A single Service can expose multiple ports. Each port **must** have a unique name when more than one port is defined.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-server
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: server
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
    - name: metrics
      protocol: TCP
      port: 9090
      targetPort: 9090
```

```bash
# Access specific ports
curl http://app-server.production:80/api
curl http://app-server.production:9090/metrics
```

## Traffic Policies

Traffic policies control how traffic is routed to Pods, affecting latency, cost, and availability.

### Internal Traffic Policy

Controls how traffic originating from **within** the cluster is routed.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-local-cache
spec:
  type: ClusterIP
  selector:
    app: cache
  internalTrafficPolicy: Local  # Only route to Pods on the same node
  ports:
    - port: 6379
      targetPort: 6379
```

| Value | Behavior |
|---|---|
| `Cluster` (default) | Traffic may be routed to any Pod on any node |
| `Local` | Traffic is routed only to Pods on the same node as the client |

### External Traffic Policy

Controls how traffic from **outside** the cluster is routed. Applies to NodePort and LoadBalancer types.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Preserve client source IP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

| Value | Client IP Preserved | Load Distribution | Risk |
|---|---|---|---|
| `Cluster` (default) | ❌ (SNAT applied) | Even across all Pods | Extra network hop |
| `Local` | ✅ | Uneven (only local Pods) | Traffic dropped if no local Pod |

```
externalTrafficPolicy: Cluster vs Local

  Cluster (default)                    Local
  ┌──────────────────┐                ┌──────────────────┐
  │ Node A           │                │ Node A           │
  │  ┌────┐          │                │  ┌────┐          │
  │  │Pod │ ◄── yes  │                │  │Pod │ ◄── yes  │
  │  └────┘          │                │  └────┘          │
  └──────────────────┘                └──────────────────┘
  ┌──────────────────┐                ┌──────────────────┐
  │ Node B           │                │ Node B           │
  │  (no local Pod)  │                │  (no local Pod)  │
  │  Traffic hops ──▶│── to Node A    │  Traffic dropped │
  └──────────────────┘                │  (no endpoint)   │
                                      └──────────────────┘
```

## Troubleshooting Services

### Common Issues

| Symptom | Likely Cause | Diagnostic Command |
|---|---|---|
| Service has no Endpoints | Selector does not match any Pod labels | `kubectl get endpoints <svc>` |
| DNS resolution fails | CoreDNS is not running or misconfigured | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| Connection refused | Pod container is not listening on `targetPort` | `kubectl exec <pod> -- ss -tlnp` |
| Intermittent timeouts | Some Pods are unhealthy or failing readiness probes | `kubectl get pods -o wide` and check `READY` column |
| NodePort unreachable | Firewall rules blocking the port range | Check cloud security groups / firewall rules |
| LoadBalancer stuck in `<pending>` | No cloud controller manager or insufficient permissions | `kubectl describe svc <svc>` — check Events |

### Debugging Step by Step

```bash
# 1. Verify the Service exists and has a ClusterIP
kubectl get svc <service-name> -n <namespace>

# 2. Check that Endpoints are populated
kubectl get endpoints <service-name> -n <namespace>

# 3. Verify that matching Pods are running and ready
kubectl get pods -n <namespace> -l <label-selector> -o wide

# 4. Test DNS resolution from inside the cluster
kubectl run dns-debug --rm -it --image=busybox:1.36 -- \
  nslookup <service-name>.<namespace>.svc.cluster.local

# 5. Test direct connectivity to a Pod IP
kubectl run curl-debug --rm -it --image=curlimages/curl -- \
  curl -sv http://<pod-ip>:<targetPort>/health

# 6. Test connectivity through the Service ClusterIP
kubectl run curl-debug --rm -it --image=curlimages/curl -- \
  curl -sv http://<cluster-ip>:<port>/health

# 7. Check kube-proxy mode and iptables/IPVS rules on a node
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# 8. Inspect Service events for provisioning errors
kubectl describe svc <service-name> -n <namespace>
```

### Verifying Label Selectors

A mismatch between Service selectors and Pod labels is the most common source of empty Endpoints.

```bash
# Show the Service selector
kubectl get svc <service-name> -o jsonpath='{.spec.selector}' | jq .

# Show labels on candidate Pods
kubectl get pods -n <namespace> --show-labels

# Test the selector directly
kubectl get pods -n <namespace> -l app=backend,tier=api
```

## Best Practices

### Naming and Organization

- ✅ Use descriptive Service names that reflect the workload (`payment-api`, not `svc1`)
- ✅ Always set the `name` field on ports when exposing multiple ports
- ✅ Place Services in the same namespace as the Pods they select
- ❌ Do not hardcode ClusterIPs — use DNS names instead

### Type Selection

- ✅ Use **ClusterIP** for all internal communication (default and most common)
- ✅ Use **LoadBalancer** for production external access in cloud environments
- ✅ Use **Headless** (`clusterIP: None`) for StatefulSets and peer discovery
- ✅ Use **ExternalName** to abstract external dependencies behind a stable internal name
- ❌ Avoid **NodePort** in production — use an Ingress controller or LoadBalancer instead

### Reliability

- ✅ Always define **readiness probes** on Pods — Services only route to ready Pods
- ✅ Use `externalTrafficPolicy: Local` when you need to preserve the client source IP
- ✅ Set `internalTrafficPolicy: Local` for node-local caches to reduce latency
- ❌ Do not rely on environment-variable-based discovery — prefer DNS

### Security

- ✅ Use **NetworkPolicies** to restrict which Pods can communicate with a Service
- ✅ Expose only necessary ports — avoid wildcards or overly broad configurations
- ❌ Do not expose internal Services via NodePort or LoadBalancer without access controls

## Next Steps

Continue exploring Kubernetes networking and workload topics:

- [Kubernetes Overview](00-OVERVIEW.md) — Review the architecture and core concepts
- **Ingress and Ingress Controllers** — HTTP/HTTPS routing, TLS termination, path-based routing
- **Network Policies** — Controlling Pod-to-Pod and Pod-to-Service traffic
- **DNS and CoreDNS** — Customizing cluster DNS resolution
- **Pods and Workloads** — Deployments, StatefulSets, DaemonSets, and Jobs
