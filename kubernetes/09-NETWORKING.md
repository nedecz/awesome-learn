# Networking

## Table of Contents

1. [Overview](#overview)
   - [The Kubernetes Networking Model](#the-kubernetes-networking-model)
   - [Four Networking Problems](#four-networking-problems)
2. [Container Network Interface (CNI)](#container-network-interface-cni)
   - [What Is CNI](#what-is-cni)
   - [Popular CNI Plugins](#popular-cni-plugins)
   - [Plugin Comparison](#plugin-comparison)
   - [Installing a CNI Plugin](#installing-a-cni-plugin)
3. [Pod Networking](#pod-networking)
   - [Container-to-Container Communication](#container-to-container-communication)
   - [Pod-to-Pod on the Same Node](#pod-to-pod-on-the-same-node)
   - [Pod-to-Pod Across Nodes](#pod-to-pod-across-nodes)
4. [Network Policies](#network-policies)
   - [How Network Policies Work](#how-network-policies-work)
   - [Default Deny All Ingress](#default-deny-all-ingress)
   - [Allow from Specific Namespace](#allow-from-specific-namespace)
   - [Allow Specific Ports](#allow-specific-ports)
   - [Egress Policies](#egress-policies)
   - [Default Deny All Egress](#default-deny-all-egress)
   - [Combined Ingress and Egress](#combined-ingress-and-egress)
5. [Ingress](#ingress)
   - [Ingress Resources vs Ingress Controllers](#ingress-resources-vs-ingress-controllers)
   - [Installing NGINX Ingress Controller](#installing-nginx-ingress-controller)
   - [Path-Based Routing](#path-based-routing)
   - [Host-Based Routing](#host-based-routing)
   - [TLS Termination](#tls-termination)
   - [Common Annotations](#common-annotations)
6. [Gateway API](#gateway-api)
   - [Why Gateway API](#why-gateway-api)
   - [Core Resources](#core-resources)
   - [Gateway API vs Ingress](#gateway-api-vs-ingress)
7. [Service Mesh](#service-mesh)
   - [What Is a Service Mesh](#what-is-a-service-mesh)
   - [Istio](#istio)
   - [Linkerd](#linkerd)
   - [Service Mesh Comparison](#service-mesh-comparison)
8. [DNS in Kubernetes](#dns-in-kubernetes)
   - [CoreDNS Overview](#coredns-overview)
   - [Service DNS Records](#service-dns-records)
   - [Pod DNS Policies](#pod-dns-policies)
   - [Debugging DNS](#debugging-dns)
9. [Load Balancing](#load-balancing)
   - [kube-proxy Modes](#kube-proxy-modes)
   - [External Load Balancers](#external-load-balancers)
   - [Internal Load Balancing](#internal-load-balancing)
10. [Troubleshooting Networking](#troubleshooting-networking)
    - [Common Issues](#common-issues)
    - [Debugging Tools and Commands](#debugging-tools-and-commands)
    - [DNS Debugging](#dns-debugging)
11. [Best Practices](#best-practices)
12. [Next Steps](#next-steps)

## Overview

This document provides a comprehensive guide to **Kubernetes networking** — from the fundamental model that gives every Pod its own IP address, through CNI plugins, network policies, ingress, service meshes, and DNS.

### Target Audience

- Platform engineers designing cluster networking
- Developers who need to understand how their services communicate
- SREs debugging connectivity and DNS issues in production
- DevOps engineers setting up ingress, load balancing, and network policies

### Scope

- The Kubernetes networking model and its guarantees
- CNI plugins, selection criteria, and installation
- Network policies for microsegmentation
- Ingress, Gateway API, and service mesh patterns
- DNS resolution, load balancing, and troubleshooting

### The Kubernetes Networking Model

Kubernetes imposes a set of fundamental requirements on any networking implementation:

1. **Every Pod gets its own IP address** — no NAT between Pods
2. **All Pods can communicate with all other Pods** without NAT (flat network)
3. **All Nodes can communicate with all Pods** (and vice versa) without NAT
4. **The IP a Pod sees itself as is the same IP others see it as**

```
┌──────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                         │
│                                                              │
│   ┌──────────────────────┐    ┌──────────────────────┐       │
│   │       Node A         │    │       Node B         │       │
│   │                      │    │                      │       │
│   │  ┌────────┐ ┌──────┐ │    │  ┌────────┐ ┌──────┐│       │
│   │  │ Pod A1 │ │Pod A2│ │    │  │ Pod B1 │ │Pod B2││       │
│   │  │10.1.1.2│ │10.1. │ │    │  │10.1.2.2│ │10.1. ││       │
│   │  │        │ │  1.3 │ │    │  │        │ │  2.3 ││       │
│   │  └────────┘ └──────┘ │    │  └────────┘ └──────┘│       │
│   │         │       │    │    │         │       │    │       │
│   │    ┌────┴───────┴──┐ │    │    ┌────┴───────┴──┐│       │
│   │    │  veth / cbr0  │ │    │    │  veth / cbr0  ││       │
│   │    └───────┬───────┘ │    │    └───────┬───────┘│       │
│   │            │         │    │            │        │       │
│   └────────────┼─────────┘    └────────────┼────────┘       │
│                │                           │                │
│          ┌─────┴───────────────────────────┴─────┐          │
│          │       Cluster Network (Overlay /       │          │
│          │         Underlay / Cloud VPC)          │          │
│          └───────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

### Four Networking Problems

Kubernetes addresses four distinct networking challenges:

| Problem                  | Solution                                      |
|--------------------------|-----------------------------------------------|
| Container-to-Container   | Shared network namespace (localhost)           |
| Pod-to-Pod               | CNI plugin providing flat network              |
| Pod-to-Service           | kube-proxy (iptables / IPVS) or eBPF           |
| External-to-Service      | NodePort, LoadBalancer, Ingress, Gateway API   |

## Container Network Interface (CNI)

### What Is CNI

The **Container Network Interface (CNI)** is a specification and set of libraries for configuring network interfaces in Linux containers. Kubernetes delegates all Pod networking setup to a CNI plugin, which is responsible for:

- Allocating an IP address to each Pod
- Setting up network interfaces (`veth` pairs)
- Programming routes and firewall rules
- Implementing network policies (if supported)

When a Pod is created, the kubelet calls the CNI plugin binary with the container's network namespace. The plugin attaches a network interface, assigns an IP, and sets up routing.

```
┌────────────────────────────────────────────────┐
│                   kubelet                       │
│                                                │
│  1. Create Pod sandbox                         │
│  2. Call CNI plugin ──────────┐                │
│  3. Start containers          │                │
│                               ▼                │
│                    ┌──────────────────┐         │
│                    │   CNI Plugin     │         │
│                    │                  │         │
│                    │ • Create veth    │         │
│                    │ • Assign IP      │         │
│                    │ • Setup routes   │         │
│                    │ • Apply policy   │         │
│                    └──────────────────┘         │
└────────────────────────────────────────────────┘
```

### Popular CNI Plugins

**Calico** — Uses BGP for routing and supports both overlay (VXLAN/IP-in-IP) and native routing modes. Provides full network policy support and scales well in large clusters.

**Cilium** — Built on eBPF for high-performance dataplane processing. Offers advanced network policies (L7-aware), transparent encryption, and deep observability via Hubble.

**Flannel** — A simple overlay network using VXLAN. Easy to set up but lacks built-in network policy support (requires Calico for policies when used together).

**Weave Net** — Provides an encrypted mesh overlay network. Supports network policies and automatic mesh discovery. Good for smaller clusters.

**AWS VPC CNI** — The default CNI for Amazon EKS. Assigns real VPC IP addresses to Pods, allowing them to participate directly in VPC networking and security groups.

**Azure CNI** — The default CNI for Azure AKS. Assigns VNet IP addresses to Pods for native Azure networking integration with NSGs and route tables.

### Plugin Comparison

| Feature              | Calico       | Cilium       | Flannel     | Weave Net  | AWS VPC CNI | Azure CNI  |
|----------------------|-------------|-------------|-------------|------------|-------------|------------|
| Routing              | BGP / VXLAN  | eBPF / VXLAN | VXLAN       | VXLAN mesh | VPC native  | VNet native|
| Network Policy       | ✅ Full      | ✅ Full+L7   | ❌ None      | ✅ Basic    | ✅ With SGs  | ✅ Basic    |
| eBPF Dataplane       | ✅ Optional  | ✅ Default   | ❌           | ❌          | ❌           | ❌          |
| Encryption           | WireGuard    | WireGuard/IPsec | ❌      | ✅ Sleeve   | ❌           | ❌          |
| Performance          | High         | Very High    | Moderate    | Moderate   | High        | High       |
| Observability        | Basic        | Hubble       | Minimal     | Basic      | VPC Flow    | NSG Flow   |
| Multi-cluster        | ✅           | ✅ ClusterMesh| ❌          | ✅          | ❌           | ❌          |
| Complexity           | Medium       | Medium-High  | Low         | Low        | Low         | Low        |

### Installing a CNI Plugin

Most CNI plugins are installed by applying a manifest to the cluster. Here is an example with Calico:

```bash
# Install Calico operator and custom resources
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# Verify installation
kubectl get pods -n calico-system -w
```

For Cilium, use the Cilium CLI:

```bash
# Install Cilium
cilium install --version 1.15.0

# Validate the installation
cilium status --wait
```

## Pod Networking

### Container-to-Container Communication

Containers within the same Pod share a **network namespace**. They communicate over `localhost` and share the same IP address and port space.

```
┌──────────────────────────────────────────┐
│              Pod (10.1.1.5)              │
│                                          │
│  ┌──────────────┐   ┌──────────────┐    │
│  │ Container A  │   │ Container B  │    │
│  │  (app:8080)  │   │ (sidecar:    │    │
│  │              │   │     9090)    │    │
│  └──────┬───────┘   └──────┬───────┘    │
│         │                  │            │
│         │   localhost      │            │
│         └──────────────────┘            │
│                                          │
│    Shared: IP, ports, loopback,          │
│            network interfaces            │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │        eth0 (10.1.1.5)            │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

Container A can reach Container B at `localhost:9090`, and vice versa at `localhost:8080`. There is no network boundary between them.

### Pod-to-Pod on the Same Node

When two Pods are on the same node, traffic flows through a virtual bridge (e.g., `cbr0`) or directly via eBPF programs depending on the CNI plugin:

```
┌─────────────────────────────────────────────────┐
│                     Node                         │
│                                                 │
│  ┌──────────┐              ┌──────────┐         │
│  │  Pod A   │              │  Pod B   │         │
│  │ 10.1.1.2 │              │ 10.1.1.3 │         │
│  └────┬─────┘              └────┬─────┘         │
│       │ veth-a                  │ veth-b         │
│       │                        │                │
│  ┌────┴────────────────────────┴────┐           │
│  │       Linux Bridge (cbr0)        │           │
│  │       or eBPF datapath           │           │
│  └──────────────────────────────────┘           │
│                                                 │
│  Traffic stays local — no overlay overhead       │
└─────────────────────────────────────────────────┘
```

### Pod-to-Pod Across Nodes

Cross-node communication requires the CNI plugin to route packets between nodes, typically using an overlay (VXLAN, IP-in-IP) or native routing (BGP, cloud VPC routes):

```
┌───────────────────────┐          ┌───────────────────────┐
│        Node A         │          │        Node B         │
│                       │          │                       │
│  ┌──────────┐         │          │         ┌──────────┐  │
│  │  Pod A   │         │          │         │  Pod B   │  │
│  │ 10.1.1.2 │         │          │         │ 10.1.2.3 │  │
│  └────┬─────┘         │          │         └────┬─────┘  │
│       │ veth          │          │          veth │        │
│       │               │          │               │        │
│  ┌────┴─────────┐     │          │     ┌─────────┴────┐  │
│  │   cbr0       │     │          │     │   cbr0       │  │
│  └────┬─────────┘     │          │     └─────────┬────┘  │
│       │               │          │               │        │
│  ┌────┴─────────┐     │          │     ┌─────────┴────┐  │
│  │ eth0 / ens5  │     │          │     │ eth0 / ens5  │  │
│  │  192.168.1.10│     │          │     │ 192.168.1.20 │  │
│  └────┬─────────┘     │          │     └─────────┬────┘  │
│       │               │          │               │        │
└───────┼───────────────┘          └───────────────┼────────┘
        │                                          │
   ┌────┴──────────────────────────────────────────┴────┐
   │           Physical / Cloud Network                  │
   │      (VXLAN tunnel, BGP routes, or VPC routes)     │
   └────────────────────────────────────────────────────┘
```

The overlay encapsulates the Pod IP packet inside a Node IP packet. Native routing avoids this overhead by programming routes directly into the network fabric.

## Network Policies

### How Network Policies Work

Network policies are Kubernetes resources that control traffic flow at the Pod level. They act as a firewall for Pod-to-Pod communication. Key concepts:

- **Policies are additive** — if any policy allows traffic, it is allowed
- **Policies are namespace-scoped** — they apply to Pods in their namespace
- **Selecting no Pods** means the policy has no effect
- **No policies** = all traffic is allowed (default open)
- **Any policy selecting a Pod** = only explicitly allowed traffic is permitted

> **Important:** Network policies require a CNI plugin that supports them (e.g., Calico, Cilium, Weave Net). Flannel alone does **not** enforce network policies.

```
┌─────────────────────────────────────────────────────┐
│                 Network Policy Flow                  │
│                                                     │
│   Incoming        NetworkPolicy        Pod           │
│   Traffic ──────▶ Evaluation ────────▶ (if allowed)  │
│                  (ingress rules)                     │
│                                                     │
│   Pod ──────────▶ NetworkPolicy ──────▶ Outgoing     │
│                  Evaluation            Traffic       │
│                  (egress rules)        (if allowed)  │
└─────────────────────────────────────────────────────┘
```

### Default Deny All Ingress

The most common starting point is to deny all incoming traffic to Pods in a namespace, then selectively allow what is needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}       # Selects ALL pods in the namespace
  policyTypes:
    - Ingress
  # No ingress rules = deny all incoming traffic
```

### Allow from Specific Namespace

Allow traffic from Pods in a namespace labeled `team: frontend` to reach backend Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: frontend
```

### Allow Specific Ports

Restrict traffic to only specific ports on the target Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http-and-metrics
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 9090
```

### Egress Policies

Control outgoing traffic from Pods. This example allows the backend to talk only to a database and DNS:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
    - Egress
  egress:
    # Allow traffic to the database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Default Deny All Egress

Block all outgoing traffic from Pods in a namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = deny all outgoing traffic
```

### Combined Ingress and Egress

A complete policy that controls both ingress and egress for a database tier:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only allow connections from the backend on port 5432
    - from:
        - podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 5432
  egress:
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Allow responses (stateful — usually handled automatically)
    # Allow connections to backup storage
    - to:
        - podSelector:
            matchLabels:
              app: backup-agent
      ports:
        - protocol: TCP
          port: 8200
```

## Ingress

### Ingress Resources vs Ingress Controllers

An **Ingress resource** is a Kubernetes API object that defines HTTP/HTTPS routing rules. An **Ingress controller** is the component that reads Ingress resources and configures a reverse proxy (e.g., NGINX, Envoy, HAProxy) to implement the rules.

```
┌─────────────────────────────────────────────────────────┐
│                    External Traffic                       │
│                         │                                │
│                         ▼                                │
│              ┌────────────────────┐                      │
│              │  Load Balancer     │                      │
│              │  (cloud / bare-    │                      │
│              │   metal)           │                      │
│              └─────────┬──────────┘                      │
│                        │                                 │
│                        ▼                                 │
│              ┌────────────────────┐                      │
│              │ Ingress Controller │                      │
│              │   (NGINX, Envoy,  │                      │
│              │    Traefik, etc.) │                      │
│              └─────────┬──────────┘                      │
│                        │                                 │
│           ┌────────────┼────────────┐                    │
│           │            │            │                    │
│           ▼            ▼            ▼                    │
│    ┌───────────┐ ┌───────────┐ ┌───────────┐            │
│    │ Service A │ │ Service B │ │ Service C │            │
│    │ /api      │ │ /web      │ │ /docs     │            │
│    └───────────┘ └───────────┘ └───────────┘            │
└─────────────────────────────────────────────────────────┘
```

> **Note:** Without an Ingress controller running in the cluster, creating Ingress resources has no effect.

### Installing NGINX Ingress Controller

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Or using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml

# Verify the controller is running
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Path-Based Routing

Route different URL paths to different backend Services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /docs
            pathType: Prefix
            backend:
              service:
                name: docs-service
                port:
                  number: 3000
```

### Host-Based Routing

Route traffic based on the hostname in the HTTP `Host` header:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: production
spec:
  ingressClassName: nginx
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
                  number: 8080
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 8443
```

### TLS Termination

Terminate TLS at the Ingress controller using a Kubernetes Secret:

```yaml
# Create TLS secret
# kubectl create secret tls myapp-tls \
#   --cert=path/to/tls.crt \
#   --key=path/to/tls.key \
#   -n production
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
        - api.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

### Common Annotations

| Annotation                                            | Description                                |
|-------------------------------------------------------|--------------------------------------------|
| `nginx.ingress.kubernetes.io/rewrite-target`          | Rewrite the URL path before forwarding     |
| `nginx.ingress.kubernetes.io/ssl-redirect`            | Force HTTP to HTTPS redirect               |
| `nginx.ingress.kubernetes.io/proxy-body-size`         | Maximum allowed request body size          |
| `nginx.ingress.kubernetes.io/proxy-read-timeout`      | Timeout for reading a response (seconds)   |
| `nginx.ingress.kubernetes.io/proxy-send-timeout`      | Timeout for sending a request (seconds)    |
| `nginx.ingress.kubernetes.io/rate-limit-connections`   | Limit concurrent connections per IP        |
| `nginx.ingress.kubernetes.io/cors-allow-origin`       | Set CORS allowed origins                   |
| `nginx.ingress.kubernetes.io/affinity`                | Enable session affinity (`cookie`)         |
| `nginx.ingress.kubernetes.io/whitelist-source-range`  | IP allowlist in CIDR notation              |

## Gateway API

### Why Gateway API

The Gateway API is the successor to Ingress. It addresses Ingress limitations by providing:

- **Role-oriented design** — separate concerns for infrastructure providers, cluster operators, and application developers
- **Expressiveness** — header-based matching, traffic weighting, request mirroring
- **Extensibility** — typed extension points instead of vendor-specific annotations
- **Multi-protocol support** — HTTP, HTTPS, TLS passthrough, TCP, UDP, gRPC

### Core Resources

The Gateway API defines three core resource types:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  GatewayClass    (infrastructure provider config)   │
│       │                                             │
│       ▼                                             │
│  Gateway         (cluster operator — listener,      │
│       │           ports, TLS)                       │
│       │                                             │
│       ▼                                             │
│  HTTPRoute       (application developer — routing   │
│                   rules, backends)                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**GatewayClass** — Defines the controller that manages Gateways of this class:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-gateway-class
spec:
  controllerName: example.com/gateway-controller
```

**Gateway** — Defines the listener configuration (ports, protocols, TLS):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: gateway-infra
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: production-cert
            kind: Secret
```

**HTTPRoute** — Defines routing rules with advanced matching:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-routes
  namespace: production
spec:
  parentRefs:
    - name: production-gateway
      namespace: gateway-infra
  hostnames:
    - "myapp.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/v2
          headers:
            - name: x-api-version
              value: "2"
      backendRefs:
        - name: api-v2
          port: 8080
          weight: 90
        - name: api-v3-canary
          port: 8080
          weight: 10
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-v1
          port: 8080
```

### Gateway API vs Ingress

| Feature                   | Ingress              | Gateway API            |
|---------------------------|----------------------|------------------------|
| Header-based routing      | Via annotations      | ✅ Native               |
| Traffic splitting         | ❌                    | ✅ Weighted backends     |
| Request mirroring         | ❌                    | ✅ Native               |
| TCP/UDP support           | ❌                    | ✅ TCPRoute/UDPRoute     |
| gRPC support              | Via annotations      | ✅ GRPCRoute             |
| Role-oriented design      | ❌ Single resource    | ✅ GatewayClass/Gateway/Route |
| Extension model           | Annotations          | ✅ Typed extension points |
| Cross-namespace routing   | ❌                    | ✅ ReferenceGrant        |
| Status                    | Stable (frozen)      | ✅ Actively developed    |

## Service Mesh

### What Is a Service Mesh

A service mesh is a dedicated infrastructure layer for managing service-to-service communication. It provides:

- **Mutual TLS (mTLS)** — automatic encryption and identity between services
- **Traffic management** — retries, timeouts, circuit breaking, canary releases
- **Observability** — distributed tracing, metrics, access logs
- **Policy enforcement** — authorization, rate limiting

```
┌─────────────────────────────────────────────────────────┐
│                   Service Mesh                           │
│                                                         │
│  ┌──────────────────┐        ┌──────────────────┐       │
│  │      Pod A       │        │      Pod B       │       │
│  │                  │        │                  │       │
│  │  ┌────────────┐  │        │  ┌────────────┐  │       │
│  │  │    App     │  │        │  │    App     │  │       │
│  │  │ Container  │  │        │  │ Container  │  │       │
│  │  └─────┬──────┘  │        │  └─────▲──────┘  │       │
│  │        │         │        │        │         │       │
│  │  ┌─────▼──────┐  │  mTLS  │  ┌─────┴──────┐  │       │
│  │  │   Proxy    │──┼────────┼──│   Proxy    │  │       │
│  │  │  (Envoy)   │  │        │  │  (Envoy)   │  │       │
│  │  └────────────┘  │        │  └────────────┘  │       │
│  └──────────────────┘        └──────────────────┘       │
│                                                         │
│              ┌──────────────────┐                        │
│              │   Control Plane  │                        │
│              │   (istiod /      │                        │
│              │    linkerd-cp)   │                        │
│              └──────────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

**When to use a service mesh:**

- You have many services that need mTLS encryption
- You need advanced traffic management (canary, mirroring, fault injection)
- You need deep observability across service calls
- You want to enforce authorization policies between services

**When NOT to use one:**

- You have a small number of services (under ~10)
- The added latency and complexity are not justified
- Your platform already handles mTLS and observability externally

### Istio

Istio uses an **Envoy sidecar proxy** injected into every Pod. The control plane (`istiod`) distributes configuration to sidecars.

**VirtualService** — Defines traffic routing rules:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
  namespace: production
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: beta-tester
      route:
        - destination:
            host: reviews
            subset: v3
    - route:
        - destination:
            host: reviews
            subset: v2
          weight: 80
        - destination:
            host: reviews
            subset: v3
          weight: 20
```

**DestinationRule** — Defines policies after routing (load balancing, connection pool, outlier detection):

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
  namespace: production
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
      trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 50
```

### Linkerd

Linkerd is a lightweight service mesh focused on simplicity and performance. It uses its own **linkerd2-proxy** (written in Rust) instead of Envoy, resulting in lower resource consumption.

Key characteristics:

- **Ultra-lightweight proxy** — ~10 MB memory footprint per sidecar
- **Automatic mTLS** — zero-config mutual TLS for all meshed traffic
- **Graduated CNCF project** — production-ready and battle-tested
- **No CRDs for basic traffic management** — uses Service profiles instead
- **Multi-cluster support** — service mirroring across clusters

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Install Linkerd into the cluster
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify installation
linkerd check

# Inject sidecar into a deployment
kubectl get deploy -n production -o yaml | linkerd inject - | kubectl apply -f -
```

### Service Mesh Comparison

| Feature                | Istio               | Linkerd              |
|------------------------|---------------------|----------------------|
| Proxy                  | Envoy (C++)         | linkerd2-proxy (Rust)|
| Memory per sidecar     | ~50–100 MB          | ~10–20 MB            |
| Latency overhead (p99) | ~3–5 ms             | ~1–2 ms              |
| mTLS                   | ✅ Configurable      | ✅ On by default      |
| Traffic splitting      | ✅ VirtualService    | ✅ TrafficSplit (SMI)  |
| Circuit breaking       | ✅                   | ✅ (via retries/timeouts) |
| Fault injection        | ✅                   | ❌                    |
| Multi-cluster          | ✅                   | ✅                    |
| Complexity             | High                | Low-Medium           |
| CNCF status            | Graduated           | Graduated            |

## DNS in Kubernetes

### CoreDNS Overview

CoreDNS is the default DNS server in Kubernetes. It runs as a Deployment in the `kube-system` namespace and provides DNS-based service discovery.

```
┌─────────────────────────────────────────────────────┐
│                   DNS Resolution                     │
│                                                     │
│  Pod ──▶ /etc/resolv.conf ──▶ CoreDNS ──▶ Service  │
│          (nameserver:                     ClusterIP  │
│           10.96.0.10)                               │
│                                                     │
│  If not found in cluster:                           │
│  CoreDNS ──▶ upstream DNS (e.g., 8.8.8.8)          │
└─────────────────────────────────────────────────────┘
```

CoreDNS watches the Kubernetes API for Services and Endpoints and serves DNS records accordingly. Its configuration is stored in a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### Service DNS Records

Kubernetes creates DNS records for Services in the following format:

```
<service-name>.<namespace>.svc.cluster.local
```

| Service Type   | DNS Record                              | Resolves To              |
|----------------|----------------------------------------|--------------------------|
| ClusterIP      | `my-svc.prod.svc.cluster.local`        | ClusterIP address        |
| Headless       | `my-svc.prod.svc.cluster.local`        | Set of Pod IPs           |
| ExternalName   | `my-svc.prod.svc.cluster.local`        | CNAME to external DNS    |

For headless Services (ClusterIP: None), DNS returns individual Pod IPs. Each Pod also gets a DNS record:

```
<pod-ip-dashed>.<namespace>.pod.cluster.local
# Example: 10-1-2-3.production.pod.cluster.local
```

For StatefulSets with a headless Service:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
# Example: postgres-0.postgres-headless.production.svc.cluster.local
```

### Pod DNS Policies

The `dnsPolicy` field on a Pod controls how DNS is configured:

| Policy          | Description                                                  |
|-----------------|--------------------------------------------------------------|
| `ClusterFirst`  | Default. Queries cluster DNS first, falls through to upstream |
| `Default`       | Inherits DNS config from the node                            |
| `None`          | No auto-configuration — you must specify `dnsConfig`         |
| `ClusterFirstWithHostNet` | For Pods using `hostNetwork: true`                |

Custom DNS configuration example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 10.96.0.10
      - 8.8.8.8
    searches:
      - production.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "5"
  containers:
    - name: app
      image: nginx:1.25
```

### Debugging DNS

```bash
# Run a debug pod with DNS tools
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/agnhost:2.39 \
  --command -- sleep infinity

# Look up a Service
kubectl exec dnsutils -- nslookup kubernetes.default.svc.cluster.local

# Check /etc/resolv.conf inside a Pod
kubectl exec dnsutils -- cat /etc/resolv.conf

# Query CoreDNS directly
kubectl exec dnsutils -- dig @10.96.0.10 my-service.production.svc.cluster.local

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check CoreDNS metrics
kubectl exec -n kube-system deploy/coredns -- wget -qO- http://localhost:9153/metrics | head -20
```

## Load Balancing

### kube-proxy Modes

kube-proxy runs on every node and implements the Service abstraction by programming the dataplane. It supports multiple modes:

**iptables mode** (default):
- Creates iptables rules that DNAT Service IPs to Pod IPs
- Random Pod selection using iptables probability rules
- No real health checking — relies on readiness probes to update Endpoints
- Rule updates become slow with thousands of Services

**IPVS mode** (recommended for large clusters):
- Uses Linux IPVS (IP Virtual Server) kernel module
- Supports multiple load-balancing algorithms: round-robin, least connections, shortest expected delay
- O(1) lookup performance vs O(n) for iptables
- Real connection tracking and faster rule updates

```
┌─────────────────────────────────────────────────────────┐
│                   kube-proxy Modes                       │
│                                                         │
│  iptables Mode:                                         │
│  Client ──▶ ClusterIP ──▶ iptables DNAT ──▶ Pod IP     │
│                           (random selection)            │
│                                                         │
│  IPVS Mode:                                             │
│  Client ──▶ ClusterIP ──▶ IPVS (virtual server) ──▶    │
│                           Pod IP                        │
│                           (round-robin, least-conn,     │
│                            source-hash, etc.)           │
│                                                         │
│  eBPF Mode (Cilium):                                    │
│  Client ──▶ eBPF program ──▶ Pod IP                    │
│             (no iptables or IPVS needed)                │
└─────────────────────────────────────────────────────────┘
```

| Mode       | Performance   | Algorithms                  | Scale          |
|------------|---------------|-----------------------------|----------------|
| iptables   | Good          | Random only                 | ~5,000 Services |
| IPVS       | Very good     | RR, LC, SED, NQ, SH, DH    | ~50,000+ Services |
| eBPF       | Excellent     | RR, Maglev, random          | ~100,000+ Services |

### External Load Balancers

When you create a Service of type `LoadBalancer`, Kubernetes provisions an external load balancer through the cloud provider:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
  annotations:
    # AWS-specific: use NLB instead of classic ELB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # Azure-specific: internal LB
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
```

For bare-metal clusters without a cloud provider, use **MetalLB** to provide LoadBalancer functionality:

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

```yaml
# Configure an IP address pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - production-pool
```

### Internal Load Balancing

For internal-only Services that should not be exposed externally, use ClusterIP or internal LoadBalancers:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  namespace: production
  annotations:
    # AWS: internal NLB
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    # GCP: internal LB
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: internal-api
  ports:
    - port: 443
      targetPort: 8443
```

## Troubleshooting Networking

### Common Issues

| Symptom                                    | Likely Cause                                                  |
|--------------------------------------------|---------------------------------------------------------------|
| Pod cannot reach another Pod               | CNI plugin misconfigured, NetworkPolicy blocking traffic      |
| DNS resolution fails                       | CoreDNS not running, `ndots` too high, DNS policy incorrect   |
| Service ClusterIP unreachable              | kube-proxy not running, Endpoints empty                       |
| External traffic not reaching Pods         | No Ingress controller, LoadBalancer not provisioned           |
| Intermittent connectivity                  | Node networking issues, conntrack table full                  |
| Connection resets after idle               | Cloud LB idle timeout shorter than app keepalive              |
| Slow DNS queries                           | `ndots:5` causing unnecessary search domain lookups           |

### Debugging Tools and Commands

```bash
# Check Pod networking
kubectl get pods -o wide                     # See Pod IPs and node placement
kubectl describe pod <pod-name>              # Check events and network status

# Test connectivity from inside the cluster
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash

# Inside the netshoot pod:
ping 10.1.2.3                                # Test Pod-to-Pod connectivity
curl -v http://my-service.production:8080    # Test Service connectivity
traceroute 10.1.2.3                          # Trace network path

# Check Services and Endpoints
kubectl get svc -o wide                      # See Service ClusterIPs
kubectl get endpoints <service-name>         # Verify Endpoints are populated
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

# Inspect network policies
kubectl get networkpolicies -A               # List all network policies
kubectl describe networkpolicy <policy-name> # See policy details

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20
# View iptables rules (on a node)
iptables -t nat -L KUBE-SERVICES | head -30

# Packet capture with tcpdump (from a netshoot pod)
kubectl run tcpdump --rm -it --image=nicolaka/netshoot -- \
  tcpdump -i eth0 -nn port 8080 -c 50

# Check conntrack table
conntrack -L | wc -l                         # Count entries
conntrack -S                                 # Show statistics
```

### DNS Debugging

```bash
# Detailed DNS query with dig
kubectl exec dnsutils -- dig +search +short my-service.production

# Check for NXDOMAIN responses
kubectl exec dnsutils -- nslookup my-service.production.svc.cluster.local

# Measure DNS latency
kubectl exec dnsutils -- dig +stats my-service.production.svc.cluster.local | grep "Query time"

# Test external DNS resolution
kubectl exec dnsutils -- nslookup google.com

# Check CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl exec -n kube-system deploy/coredns -- wget -qO- http://localhost:8080/health

# If DNS is slow, check ndots setting
kubectl exec <pod-name> -- cat /etc/resolv.conf
# Consider reducing ndots or using FQDNs (trailing dot):
# curl http://my-service.production.svc.cluster.local.
```

## Best Practices

1. **Start with default-deny NetworkPolicies** — Apply default deny in every namespace and explicitly allow required traffic. This follows the principle of least privilege.

2. **Use namespaces for network isolation** — Combine namespaces with NetworkPolicies to isolate workloads by team, environment, or sensitivity level.

3. **Always allow DNS egress** — When creating egress policies, remember to allow UDP/TCP port 53 so Pods can resolve Service names.

4. **Choose the right CNI plugin** — Select based on your requirements: Calico for BGP/policy, Cilium for eBPF performance, cloud CNIs for native integration.

5. **Use IPVS mode for large clusters** — Switch kube-proxy to IPVS mode when you have hundreds of Services for better performance and more load-balancing options.

6. **Reduce DNS lookups** — Use fully qualified domain names (with trailing dot) in performance-sensitive paths or lower the `ndots` value to avoid unnecessary search domain queries.

7. **Monitor CoreDNS** — Scrape CoreDNS Prometheus metrics to catch DNS latency spikes and error rates before they impact applications.

8. **Prefer Gateway API over Ingress** — For new deployments, use the Gateway API for more expressive routing and better role separation.

9. **Evaluate service mesh necessity** — Do not add a service mesh by default. Evaluate whether you actually need mTLS, traffic shaping, or deep observability before adding the complexity.

10. **Use `externalTrafficPolicy: Local`** — For LoadBalancer and NodePort Services, set this to preserve client source IPs and avoid extra network hops:

    ```yaml
    spec:
      type: LoadBalancer
      externalTrafficPolicy: Local
    ```

11. **Keep network policies version-controlled** — Treat NetworkPolicies as code. Store them in Git alongside your application manifests and review changes in pull requests.

12. **Test connectivity regularly** — Add network connectivity checks to your CI/CD pipeline or monitoring stack to catch policy regressions early.

## Next Steps

Continue your Kubernetes learning journey:

- [10-BEST-PRACTICES-AND-PATTERNS.md](10-BEST-PRACTICES-AND-PATTERNS.md) — Production-ready patterns, resource management, and operational best practices
- [08-CONTROLLERS.md](08-CONTROLLERS.md) — Deep dive into Kubernetes controllers and the reconciliation loop
- [07-CRDS-AND-OPERATORS.md](07-CRDS-AND-OPERATORS.md) — Custom Resource Definitions and the Operator pattern
- [01-SERVICES.md](01-SERVICES.md) — Kubernetes Service types (ClusterIP, NodePort, LoadBalancer, ExternalName)
- [06-MANIFESTS.md](06-MANIFESTS.md) — Writing and organizing Kubernetes manifests
