# Kubernetes Learning Resources

A comprehensive guide to Kubernetes — from fundamentals to production-grade operations, covering container orchestration, cloud-native technologies, and the K8s ecosystem.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Architecture, core concepts, kubectl basics | **Start here** |
| [01-SERVICES](01-SERVICES.md) | ClusterIP, NodePort, LoadBalancer, ExternalName, Headless | When configuring service discovery |
| [02-CLOUD-SERVICES-AWS-EKS](02-CLOUD-SERVICES-AWS-EKS.md) | EKS setup, IAM, VPC CNI, managed node groups | When deploying on AWS |
| [03-CLOUD-SERVICES-OKE](03-CLOUD-SERVICES-OKE.md) | OKE setup, compartments, VCN networking | When deploying on Oracle Cloud |
| [04-CLOUD-SERVICES-AKS](04-CLOUD-SERVICES-AKS.md) | AKS setup, Entra ID, Azure CNI, node pools | When deploying on Azure |
| [05-LOCAL-SETUP-KIND-MINIKUBE](05-LOCAL-SETUP-KIND-MINIKUBE.md) | kind, Minikube, k3d, Docker Desktop | When setting up local development |
| [06-MANIFESTS](06-MANIFESTS.md) | Pods, Deployments, StatefulSets, DaemonSets, Jobs, ConfigMaps, Secrets | When writing YAML manifests |
| [07-CRDS-AND-OPERATORS](07-CRDS-AND-OPERATORS.md) | Custom Resource Definitions, Kubebuilder, Operator SDK | When extending Kubernetes |
| [08-CONTROLLERS](08-CONTROLLERS.md) | Built-in controllers, custom controllers, reconciliation loop | When building controllers |
| [09-NETWORKING](09-NETWORKING.md) | CNI, NetworkPolicies, Ingress, Gateway API, Service Mesh | When designing network architecture |
| [10-BEST-PRACTICES-AND-PATTERNS](10-BEST-PRACTICES-AND-PATTERNS.md) | Resource management, security, reliability, deployment patterns | **Essential — production checklist** |
| [11-ANTI-PATTERNS](11-ANTI-PATTERNS.md) | Common mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 12-week training guide | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand Kubernetes architecture
   - Learn core concepts (Pods, Nodes, Namespaces)
   - Get comfortable with kubectl

2. **Set Up a Local Cluster** ([05-LOCAL-SETUP-KIND-MINIKUBE](05-LOCAL-SETUP-KIND-MINIKUBE.md))
   - Install kind or Minikube
   - Create your first cluster
   - Deploy a sample application

3. **Learn Manifests** ([06-MANIFESTS](06-MANIFESTS.md))
   - Write Pods, Deployments, Services
   - Configure with ConfigMaps and Secrets

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured, phased curriculum
   - Hands-on exercises and knowledge checks

### For Experienced Users

1. **Review Best Practices** ([10-BEST-PRACTICES-AND-PATTERNS](10-BEST-PRACTICES-AND-PATTERNS.md))
   - Resource management and QoS
   - Security hardening
   - Deployment patterns

2. **Avoid Anti-Patterns** ([11-ANTI-PATTERNS](11-ANTI-PATTERNS.md))
   - Common operational mistakes
   - Security pitfalls

3. **Deep Dive into Networking** ([09-NETWORKING](09-NETWORKING.md))
   - CNI plugins and comparison
   - NetworkPolicies, Ingress, Gateway API

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                Control Plane                     │
│  ┌──────────┐ ┌───────────┐ ┌────────────────┐ │
│  │API Server│ │ Scheduler │ │Controller Mgr  │ │
│  └────┬─────┘ └─────┬─────┘ └───────┬────────┘ │
│       │             │               │            │
│  ┌────▼─────────────▼───────────────▼──────────┐│
│  │                  etcd                        ││
│  └──────────────────────────────────────────────┘│
└─────────────────────┬───────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────▼────┐  ┌─────▼────┐ ┌─────▼────┐
   │ Node 1  │  │  Node 2  │ │  Node 3  │
   │┌───────┐│  │┌───────┐ │ │┌───────┐ │
   ││kubelet││  ││kubelet│ │ ││kubelet│ │
   │├───────┤│  │├───────┤ │ │├───────┤ │
   ││kube-  ││  ││kube-  │ │ ││kube-  │ │
   ││proxy  ││  ││proxy  │ │ ││proxy  │ │
   │├───────┤│  │├───────┤ │ │├───────┤ │
   ││Pods   ││  ││Pods   │ │ ││Pods   │ │
   │└───────┘│  │└───────┘ │ │└───────┘ │
   └─────────┘  └──────────┘ └──────────┘
```

## 🔑 Key Concepts

### Core Resources

```
Pod → ReplicaSet → Deployment      (stateless workloads)
Pod → StatefulSet                   (stateful workloads)
Pod → DaemonSet                     (node-level agents)
Pod → Job → CronJob                 (batch processing)
```

### Quick Reference

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Deploy an application
kubectl create deployment nginx --image=nginx:1.27
kubectl expose deployment nginx --port=80 --type=ClusterIP

# View resources
kubectl get pods,svc,deploy -A
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Debug
kubectl exec -it <pod-name> -- /bin/sh
kubectl port-forward svc/nginx 8080:80
```

## 📋 Topics Covered

- **Fundamentals** — Architecture, Pods, Nodes, Namespaces, kubectl
- **Services** — ClusterIP, NodePort, LoadBalancer, ExternalName, Headless
- **Cloud Providers** — AWS EKS, Oracle OKE, Azure AKS setup and management
- **Local Development** — kind, Minikube, k3d, Docker Desktop
- **Manifests** — Deployments, StatefulSets, DaemonSets, Jobs, ConfigMaps, Secrets
- **Extending K8s** — CRDs, Operators, Custom Controllers
- **Networking** — CNI, NetworkPolicies, Ingress, Gateway API, Service Mesh, DNS
- **Best Practices** — Resource management, security, reliability, deployment patterns
- **Anti-Patterns** — Common mistakes in resource, security, deployment, and networking

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to Kubernetes?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Setting up a cluster?** → Choose [local setup](05-LOCAL-SETUP-KIND-MINIKUBE.md) or cloud ([EKS](02-CLOUD-SERVICES-AWS-EKS.md), [OKE](03-CLOUD-SERVICES-OKE.md), [AKS](04-CLOUD-SERVICES-AKS.md))

**Going to production?** → Review [Best Practices](10-BEST-PRACTICES-AND-PATTERNS.md) and [Anti-Patterns](11-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [Learning Path](LEARNING-PATH.md)
