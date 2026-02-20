# GitOps

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What is GitOps](#what-is-gitops)
   - [Core Principles](#core-principles)
   - [GitOps vs Traditional CI/CD](#gitops-vs-traditional-cicd)
3. [Push-Based vs Pull-Based Deployment Models](#push-based-vs-pull-based-deployment-models)
   - [Push-Based Deployments](#push-based-deployments)
   - [Pull-Based Deployments](#pull-based-deployments)
   - [Model Comparison](#model-comparison)
4. [GitOps Workflow](#gitops-workflow)
   - [Git as Single Source of Truth](#git-as-single-source-of-truth)
   - [The Reconciliation Loop](#the-reconciliation-loop)
   - [Branching and Promotion Strategies](#branching-and-promotion-strategies)
5. [ArgoCD](#argocd)
   - [ArgoCD Architecture](#argocd-architecture)
   - [Installation](#installation)
   - [Application CRD](#application-crd)
   - [Sync Policies](#sync-policies)
   - [App of Apps Pattern](#app-of-apps-pattern)
   - [ApplicationSets](#applicationsets)
6. [Flux](#flux)
   - [Flux Architecture](#flux-architecture)
   - [Flux Installation](#flux-installation)
   - [GitRepository Source](#gitrepository-source)
   - [Kustomization](#kustomization)
   - [HelmRelease](#helmrelease)
   - [Image Automation](#image-automation)
7. [Repository Strategies](#repository-strategies)
   - [Monorepo vs Polyrepo](#monorepo-vs-polyrepo)
   - [Environment Branches vs Directory-Based](#environment-branches-vs-directory-based)
   - [Recommended Repository Layout](#recommended-repository-layout)
8. [Secrets Management in GitOps](#secrets-management-in-gitops)
   - [The Secrets Problem](#the-secrets-problem)
   - [Sealed Secrets](#sealed-secrets)
   - [SOPS](#sops)
   - [External Secrets Operator](#external-secrets-operator)
   - [HashiCorp Vault Integration](#hashicorp-vault-integration)
   - [Secrets Solutions Comparison](#secrets-solutions-comparison)
9. [Multi-Cluster and Multi-Tenancy Patterns](#multi-cluster-and-multi-tenancy-patterns)
   - [Multi-Cluster with ArgoCD](#multi-cluster-with-argocd)
   - [Multi-Cluster with Flux](#multi-cluster-with-flux)
   - [Multi-Tenancy Patterns](#multi-tenancy-patterns)
10. [Progressive Delivery with GitOps](#progressive-delivery-with-gitops)
    - [Argo Rollouts](#argo-rollouts)
    - [Flagger](#flagger)
11. [Drift Detection and Reconciliation](#drift-detection-and-reconciliation)
    - [What is Drift](#what-is-drift)
    - [Detection Mechanisms](#detection-mechanisms)
    - [Reconciliation Strategies](#reconciliation-strategies)
12. [Comparison: ArgoCD vs Flux](#comparison-argocd-vs-flux)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

**GitOps** is an operational framework that applies DevOps best practices for infrastructure automation — such as version control, collaboration, compliance, and CI/CD — to infrastructure and application deployment. GitOps uses Git repositories as the single source of truth for declarative infrastructure and application configuration, with automated processes that continuously reconcile the desired state (in Git) with the actual state (in the cluster).

This document provides a comprehensive guide to GitOps principles, workflows, and the two leading GitOps tools — **ArgoCD** and **Flux** — along with supporting patterns for secrets management, multi-cluster deployments, and progressive delivery.

```
GitOps at a Glance
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer          Git Repository         GitOps Operator        Kubernetes
  ┌────────┐        ┌──────────────┐       ┌───────────────┐      ┌──────────┐
  │ Commit │──────▶ │  Desired     │◀──────│  Watches for  │─────▶│  Actual  │
  │ & Push │        │  State       │       │  changes and  │      │  State   │
  └────────┘        │  (YAML/Helm) │       │  reconciles   │      │  (live)  │
                    └──────────────┘       └───────────────┘      └──────────┘
                           │                       │                    │
                           └───────────────────────┘                    │
                                   Continuous                           │
                                   Comparison  ◀────────────────────────┘
```

### Target Audience

- **DevOps Engineers** implementing GitOps workflows and managing deployment pipelines
- **Platform Engineers** building internal developer platforms with GitOps foundations
- **Site Reliability Engineers (SREs)** operating Kubernetes clusters with declarative, auditable deployments
- **Architects** designing deployment strategies for multi-cluster, multi-environment systems
- **Developers** contributing application manifests and understanding how code reaches production

### Scope

- GitOps principles: declarative, versioned, automated, self-healing
- Push-based vs pull-based deployment model comparison
- ArgoCD architecture, Application CRDs, sync policies, App of Apps, and ApplicationSets
- Flux architecture, GitRepository, Kustomization, HelmRelease, and image automation
- Repository strategies: monorepo vs polyrepo, environment branches vs directory-based layouts
- Secrets management: Sealed Secrets, SOPS, External Secrets Operator, Vault
- Multi-cluster and multi-tenancy deployment patterns
- Progressive delivery with Argo Rollouts and Flagger
- Drift detection, reconciliation, and self-healing behavior

---

## What is GitOps

**GitOps** is a set of practices where the entire desired state of a system is described declaratively in Git. Automated agents running inside the target environment (typically Kubernetes) continuously monitor the Git repository and reconcile any divergence between the declared desired state and the actual live state.

The term was coined by **Weaveworks** in 2017, and it has since become a widely adopted standard for Kubernetes-native continuous delivery.

### Core Principles

GitOps rests on four foundational principles, as defined by the OpenGitOps project (a CNCF Sandbox project):

| Principle | Description |
|---|---|
| **Declarative** | The entire desired state of the system is expressed declaratively. You define *what* the system should look like, not *how* to get there. |
| **Versioned and Immutable** | The desired state is stored in Git, providing a complete version history. Every change is a commit with an author, timestamp, and message. |
| **Pulled Automatically** | Approved changes to the desired state are automatically applied to the system. Software agents pull the declared state and apply it. |
| **Continuously Reconciled** | Software agents continuously observe the actual system state and attempt to apply the desired state. Self-healing corrects drift automatically. |

```
The Four Principles of GitOps
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────┐   ┌─────────────────┐   ┌──────────────┐   ┌───────────────────┐
  │ Declarative │   │   Versioned &   │   │   Pulled     │   │   Continuously    │
  │             │   │   Immutable     │   │ Automatically│   │   Reconciled      │
  │ Define WHAT │   │ Store in Git    │   │ Agents pull  │   │ Agents detect     │
  │ not HOW     │   │ with full       │   │ and apply    │   │ drift and         │
  │             │   │ audit trail     │   │ desired state│   │ self-heal         │
  └─────────────┘   └─────────────────┘   └──────────────┘   └───────────────────┘
```

### GitOps vs Traditional CI/CD

In a traditional CI/CD pipeline, the CI server has **push access** to the target cluster. In GitOps, the deployment agent lives **inside the cluster** and **pulls** changes from Git.

| Aspect | Traditional CI/CD | GitOps |
|---|---|---|
| **Deployment trigger** | CI pipeline pushes after build | Agent in cluster pulls from Git |
| **Credentials** | CI server holds cluster credentials | Agent inside cluster uses in-cluster RBAC |
| **Source of truth** | CI pipeline scripts or artifacts | Git repository (declarative manifests) |
| **Drift detection** | None — state may drift undetected | Continuous reconciliation detects and corrects drift |
| **Rollback** | Re-run pipeline with old parameters or tag | `git revert` — revert the commit and the agent reconciles |
| **Audit trail** | CI logs (may expire or be deleted) | Git history — permanent, immutable, signed |
| **Security posture** | Broad external access to cluster API | Minimal — agent runs in cluster with least-privilege RBAC |

---

## Push-Based vs Pull-Based Deployment Models

The fundamental distinction in GitOps tooling is between **push-based** and **pull-based** deployment models.

### Push-Based Deployments

In a push-based model, an external system (typically a CI/CD server) authenticates to the target environment and pushes changes directly.

```
Push-Based Model
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────┐     ┌──────────┐     ┌────────────┐
  │  Git   │────▶│ CI/CD    │────▶│ Kubernetes │
  │  Push  │     │ Server   │     │ Cluster    │
  └────────┘     │          │     └────────────┘
                 │ kubectl  │           ▲
                 │ apply    │───────────┘
                 └──────────┘
                  Holds cluster
                  credentials
```

**Characteristics:**

- CI/CD server requires direct access to the cluster API
- Cluster credentials must be stored in the CI system
- Deployment happens only when the pipeline runs — no continuous reconciliation
- Examples: Jenkins + kubectl, GitHub Actions + kubectl, Azure DevOps pipelines

### Pull-Based Deployments

In a pull-based model, an agent running **inside** the cluster watches the Git repository and pulls changes into the cluster.

```
Pull-Based Model (GitOps)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────┐     ┌──────────────┐     ┌────────────────────┐
  │  Git   │     │ Git          │     │ Kubernetes Cluster │
  │  Push  │────▶│ Repository   │◀────│                    │
  └────────┘     │              │     │  ┌──────────────┐  │
                 │ Desired State│     │  │ GitOps Agent │  │
                 └──────────────┘     │  │ (ArgoCD/Flux)│  │
                                      │  │              │  │
                                      │  │ Watches Git  │  │
                                      │  │ Reconciles   │  │
                                      │  └──────────────┘  │
                                      └────────────────────┘
                                        Agent uses in-cluster
                                        service account
```

**Characteristics:**

- No external system needs cluster credentials
- Agent continuously watches Git for changes
- Drift is automatically detected and corrected
- Examples: ArgoCD, Flux

### Model Comparison

| Aspect | Push-Based | Pull-Based (GitOps) |
|---|---|---|
| **Credential exposure** | CI system holds cluster credentials externally | Agent uses in-cluster service account — no external exposure |
| **Attack surface** | Larger — CI server compromise can access cluster | Smaller — only Git read access from inside cluster |
| **Drift handling** | None — state may diverge silently | Continuous reconciliation corrects drift automatically |
| **Deployment latency** | Immediate — pipeline pushes on trigger | Near-real-time — agent polls or receives webhooks |
| **Scalability** | Pipeline must target each cluster explicitly | Agent per cluster scales independently |
| **Complexity** | Simpler initial setup | Requires operator installation and configuration |

---

## GitOps Workflow

### Git as Single Source of Truth

In GitOps, the Git repository is the **single source of truth** for the desired state of your infrastructure and applications. Every change to the system — whether a new deployment, a configuration update, or a rollback — is expressed as a Git commit.

```
GitOps Change Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. Developer           2. Pull Request        3. Review & Merge
  ┌────────────────┐    ┌────────────────┐     ┌────────────────┐
  │ Edit manifests │───▶│ Open PR with   │────▶│ Team reviews   │
  │ in Git branch  │    │ desired changes│     │ and approves   │
  └────────────────┘    └────────────────┘     └───────┬────────┘
                                                       │
                                                       ▼
  6. Verified          5. Reconcile            4. Merge to main
  ┌────────────────┐   ┌────────────────┐     ┌────────────────┐
  │ Actual state   │◀──│ GitOps agent   │◀────│ Commit lands   │
  │ matches desired│   │ detects change │     │ on main branch │
  └────────────────┘   │ and applies it │     └────────────────┘
                       └────────────────┘
```

**Key benefits of Git as the source of truth:**

- **Auditability** — every change has an author, timestamp, and commit message
- **Rollback** — revert a commit to undo any change
- **Collaboration** — use pull requests for review, discussion, and approval
- **Reproducibility** — clone the repo and redeploy the entire system from scratch
- **Compliance** — signed commits and branch protection provide verifiable change control

### The Reconciliation Loop

The GitOps agent runs a continuous **reconciliation loop** that compares the desired state in Git with the actual state in the cluster:

```
Reconciliation Loop
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

         ┌──────────────┐
         │  Read desired │
         │  state from   │
         │  Git          │
         └──────┬───────┘
                │
                ▼
         ┌──────────────┐
         │  Read actual  │
    ┌───▶│  state from   │
    │    │  cluster      │
    │    └──────┬───────┘
    │           │
    │           ▼
    │    ┌──────────────┐     ┌──────────────┐
    │    │   Compare    │────▶│   In Sync?   │
    │    └──────────────┘     └──────┬───────┘
    │                                │
    │                     Yes ◀──────┴──────▶ No
    │                      │                   │
    │                      ▼                   ▼
    │              ┌──────────────┐     ┌──────────────┐
    │              │    Sleep     │     │    Apply     │
    │              │  (interval)  │     │   changes    │
    │              └──────┬───────┘     └──────┬───────┘
    │                     │                    │
    └─────────────────────┴────────────────────┘
```

### Branching and Promotion Strategies

GitOps supports multiple strategies for promoting changes across environments:

**Environment branches:**

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│   dev    │────▶│ staging  │────▶│   prod   │
│  branch  │     │  branch  │     │  branch  │
└──────────┘     └──────────┘     └──────────┘
  Auto-sync       Auto-sync       Manual sync
```

**Directory-based promotion (single branch):**

```
main branch
├── environments/
│   ├── dev/           ← auto-synced
│   ├── staging/       ← auto-synced after dev verified
│   └── prod/          ← manual PR required
```

---

## ArgoCD

**ArgoCD** is a declarative, GitOps continuous delivery tool for Kubernetes. It is a CNCF graduated project and one of the most widely adopted GitOps tools.

### ArgoCD Architecture

```
ArgoCD Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────────────┐
  │                    Kubernetes Cluster                        │
  │                                                             │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │                    ArgoCD Namespace                    │  │
  │  │                                                       │  │
  │  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │  │
  │  │  │   API       │  │   Repo      │  │  Application │  │  │
  │  │  │   Server    │  │   Server    │  │  Controller  │  │  │
  │  │  │             │  │             │  │              │  │  │
  │  │  │ Serves UI   │  │ Clones Git  │  │ Watches apps │  │  │
  │  │  │ and API     │  │ repos and   │  │ and performs │  │  │
  │  │  │ endpoints   │  │ generates   │  │ sync/diff    │  │  │
  │  │  │             │  │ manifests   │  │ operations   │  │  │
  │  │  └─────────────┘  └─────────────┘  └──────────────┘  │  │
  │  │                                                       │  │
  │  │  ┌─────────────┐  ┌─────────────────────────────────┐ │  │
  │  │  │   Redis     │  │  ApplicationSet Controller      │ │  │
  │  │  │  (cache)    │  │  (generates Applications from   │ │  │
  │  │  │             │  │   templates)                     │ │  │
  │  │  └─────────────┘  └─────────────────────────────────┘ │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  │  ┌─────────────────┐  ┌─────────────────┐                  │
  │  │  App Namespace  │  │  App Namespace  │  ...             │
  │  │  (deployed by   │  │  (deployed by   │                  │
  │  │   ArgoCD)       │  │   ArgoCD)       │                  │
  │  └─────────────────┘  └─────────────────┘                  │
  └─────────────────────────────────────────────────────────────┘

         ▲                        ▲
         │ UI / CLI / API         │ Git poll / webhook
         │                        │
    ┌────┴────┐            ┌──────┴──────┐
    │  User   │            │   Git Repo  │
    └─────────┘            └─────────────┘
```

**Components:**

| Component | Role |
|---|---|
| **API Server** | Exposes the REST/gRPC API and the web UI. Handles authentication and RBAC. |
| **Repo Server** | Clones Git repositories, generates Kubernetes manifests (Helm, Kustomize, plain YAML, Jsonnet). |
| **Application Controller** | Monitors running applications, compares live state to desired state, and performs sync operations. |
| **Redis** | Caches repository data and application state for performance. |
| **ApplicationSet Controller** | Generates multiple Application resources from templates using generators. |

### Installation

Install ArgoCD into a Kubernetes cluster:

```yaml
# Create the argocd namespace and install ArgoCD
# Option 1: Quick install (non-HA)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Option 2: HA install for production
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

```bash
# Install the ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Get the initial admin password
argocd admin initial-password -n argocd

# Login to ArgoCD
argocd login argocd-server.argocd.svc.cluster.local
```

### Application CRD

The **Application** custom resource is the core abstraction in ArgoCD. It defines the source (Git repository), the destination (Kubernetes cluster and namespace), and sync configuration.

```yaml
# Basic ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  # Source: where to read desired state
  source:
    repoURL: https://github.com/my-org/my-app-config.git
    targetRevision: main
    path: manifests/overlays/production

  # Destination: where to deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app

  # Sync policy: how to reconcile
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

**Application with Helm source:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 55.5.0
    helm:
      releaseName: prometheus
      values: |
        grafana:
          enabled: true
          adminPassword: admin
        prometheus:
          prometheusSpec:
            retention: 30d
            resources:
              requests:
                memory: 2Gi
                cpu: 500m
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Application with Kustomize source:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app-config.git
    targetRevision: main
    path: kustomize/overlays/staging
    kustomize:
      images:
        - my-org/my-app:v1.2.3
      commonLabels:
        environment: staging
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-staging
```

### Sync Policies

ArgoCD supports fine-grained control over how and when synchronization occurs:

| Policy | Description |
|---|---|
| **Manual sync** | User explicitly triggers sync from the UI or CLI. Default behavior. |
| **Automated sync** | ArgoCD automatically syncs when it detects Git changes. |
| **Self-heal** | If someone modifies a live resource directly (kubectl edit), ArgoCD reverts it. |
| **Prune** | Resources that exist in the cluster but no longer exist in Git are deleted. |
| **Retry** | Automatically retry failed sync operations with configurable backoff. |

```yaml
# Comprehensive sync policy example
spec:
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual changes to live resources
      allowEmpty: false   # Do not sync if source returns empty
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
    syncOptions:
      - Validate=true               # Validate manifests before applying
      - CreateNamespace=true         # Create namespace if it does not exist
      - PrunePropagationPolicy=foreground  # Wait for dependents on prune
      - PruneLast=true               # Prune after all other resources sync
      - ServerSideApply=true         # Use server-side apply for large CRDs
      - RespectIgnoreDifferences=true  # Honor ignoreDifferences in diff
```

### App of Apps Pattern

The **App of Apps** pattern uses one ArgoCD Application to manage other ArgoCD Applications. This enables bootstrapping an entire cluster's workloads from a single root Application.

```yaml
# Root Application — manages all other applications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/cluster-config.git
    targetRevision: main
    path: apps/            # Directory containing Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
App of Apps Pattern
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────┐
  │      Root App           │
  │  (manages /apps/ dir)   │
  └────────┬────────────────┘
           │
     ┌─────┼──────────┬──────────────┐
     ▼     ▼          ▼              ▼
  ┌──────┐ ┌──────┐ ┌───────────┐ ┌──────────┐
  │ App: │ │ App: │ │ App:      │ │ App:     │
  │ nginx│ │ redis│ │ prometheus│ │ my-app   │
  └──┬───┘ └──┬───┘ └────┬──────┘ └────┬─────┘
     │        │           │             │
     ▼        ▼           ▼             ▼
  Deploys  Deploys    Deploys       Deploys
  nginx    redis      monitoring    application
  stack    cluster    stack         workloads
```

### ApplicationSets

**ApplicationSets** automate the generation of ArgoCD Application resources using templates and generators. They are essential for managing applications across multiple clusters, environments, or teams at scale.

**List generator — deploy to multiple clusters:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster.example.com
            namespace: my-app-dev
          - cluster: staging
            url: https://staging-cluster.example.com
            namespace: my-app-staging
          - cluster: prod
            url: https://prod-cluster.example.com
            namespace: my-app-prod
  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/my-app-config.git
        targetRevision: main
        path: 'overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Git directory generator — one Application per directory:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/my-org/cluster-config.git
        revision: main
        directories:
          - path: addons/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/cluster-config.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

**Matrix generator — combine multiple generators:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-addons
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/my-org/cluster-config.git
              revision: main
              directories:
                - path: addons/*
          - clusters:
              selector:
                matchLabels:
                  environment: production
  template:
    metadata:
      name: '{{name}}-{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/cluster-config.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
```

---

## Flux

**Flux** is a set of continuous delivery solutions for Kubernetes that are open and extensible. Flux is a CNCF graduated project and follows the GitOps Toolkit approach — a set of composable APIs and specialized controllers.

### Flux Architecture

```
Flux Architecture (GitOps Toolkit)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────────────────────┐
  │                      Kubernetes Cluster                             │
  │                                                                     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │                    flux-system Namespace                      │  │
  │  │                                                               │  │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │  │
  │  │  │   Source      │  │ Kustomize    │  │  Helm              │  │  │
  │  │  │   Controller  │  │ Controller   │  │  Controller        │  │  │
  │  │  │              │  │              │  │                    │  │  │
  │  │  │ Fetches      │  │ Builds and   │  │ Reconciles         │  │  │
  │  │  │ artifacts    │  │ applies      │  │ HelmReleases       │  │  │
  │  │  │ from Git,    │  │ Kustomize    │  │ from               │  │  │
  │  │  │ Helm, OCI,   │  │ overlays     │  │ HelmRepositories   │  │  │
  │  │  │ S3 buckets   │  │              │  │                    │  │  │
  │  │  └──────────────┘  └──────────────┘  └────────────────────┘  │  │
  │  │                                                               │  │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │  │
  │  │  │ Notification │  │ Image        │  │ Image              │  │  │
  │  │  │ Controller   │  │ Reflector    │  │ Automation         │  │  │
  │  │  │              │  │ Controller   │  │ Controller         │  │  │
  │  │  │ Sends alerts │  │              │  │                    │  │  │
  │  │  │ to Slack,    │  │ Scans        │  │ Updates Git with   │  │  │
  │  │  │ Teams, etc.  │  │ registries   │  │ new image tags     │  │  │
  │  │  └──────────────┘  └──────────────┘  └────────────────────┘  │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘
```

**Components:**

| Controller | CRDs | Purpose |
|---|---|---|
| **Source Controller** | GitRepository, HelmRepository, HelmChart, Bucket, OCIRepository | Fetches artifacts from external sources |
| **Kustomize Controller** | Kustomization | Builds and applies Kustomize overlays or plain YAML |
| **Helm Controller** | HelmRelease | Reconciles Helm chart releases |
| **Notification Controller** | Alert, Provider, Receiver | Sends/receives notifications and webhooks |
| **Image Reflector Controller** | ImageRepository, ImagePolicy | Scans container registries for new image tags |
| **Image Automation Controller** | ImageUpdateAutomation | Commits image tag updates back to Git |

### Flux Installation

```bash
# Install the Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux into a cluster with GitHub
flux bootstrap github \
  --owner=my-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# Check Flux status
flux check
flux get all
```

### GitRepository Source

The **GitRepository** resource tells the Source Controller where to fetch manifests:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m                    # How often to check for changes
  url: https://github.com/my-org/my-app-config.git
  ref:
    branch: main
  secretRef:
    name: github-token            # Secret containing Git credentials
  ignore: |
    # Exclude non-deployment files
    /*
    !/manifests
```

### Kustomization

The **Kustomization** resource tells the Kustomize Controller what to build and apply from a source:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m                   # Reconciliation interval
  targetNamespace: my-app
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./manifests/overlays/production
  prune: true                     # Delete removed resources
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: my-app
  timeout: 5m
  patches:
    - patch: |
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
        spec:
          replicas: 3
      target:
        kind: Deployment
        name: my-app
```

**Kustomization with dependencies (ordered deployment):**

```yaml
# Infrastructure first
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./infrastructure
  prune: true
---
# Applications depend on infrastructure
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: applications
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: infrastructure       # Wait for infrastructure to be ready
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./applications
  prune: true
```

### HelmRelease

The **HelmRelease** resource manages Helm chart installations:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: redis
spec:
  interval: 30m
  chart:
    spec:
      chart: redis
      version: "18.x"            # Semver range
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 12h
  values:
    architecture: replication
    replica:
      replicaCount: 3
    auth:
      enabled: true
      existingSecret: redis-auth
    metrics:
      enabled: true
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
  test:
    enable: true
  rollback:
    cleanupOnFail: true
```

```yaml
# HelmRepository source for the above HelmRelease
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 24h
  url: https://charts.bitnami.com/bitnami
```

### Image Automation

Flux can automatically detect new container images and update the Git repository with the latest tags — closing the GitOps loop for image promotions.

```yaml
# Step 1: Define which registry to scan
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  image: ghcr.io/my-org/my-app
  interval: 5m
  secretRef:
    name: ghcr-auth
---
# Step 2: Define the policy for selecting the latest image
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    semver:
      range: ">=1.0.0"           # Use semver to pick the latest
---
# Step 3: Automate Git updates
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: my-app
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@my-org.com
        name: Flux Image Automation
      messageTemplate: |
        chore: update image {{range .Changed.Changes}}{{.OldValue}} -> {{.NewValue}} {{end}}
    push:
      branch: main
  update:
    path: ./manifests
    strategy: Setters             # Uses inline markers in YAML
```

In the deployment manifest, add **image policy markers** so Flux knows where to update:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: ghcr.io/my-org/my-app:1.2.3 # {"$imagepolicy": "flux-system:my-app"}
```

---

## Repository Strategies

How you organize your Git repositories has a significant impact on team workflows, access control, and operational complexity.

### Monorepo vs Polyrepo

| Aspect | Monorepo | Polyrepo |
|---|---|---|
| **Structure** | Single repo for app code and deployment config | Separate repos for app code and deployment config |
| **Simplicity** | Easier to get started, fewer repos to manage | More repos, but clear separation of concerns |
| **Access control** | Harder — everyone with app code access sees infra config | Easier — independent RBAC per repo |
| **CI coupling** | App build and config change in the same commit | Decoupled — app CI updates config repo via automation |
| **Blast radius** | A misconfigured CI pipeline could affect config | Config repo changes are isolated |
| **Best for** | Small teams, simple applications | Larger teams, strict separation of duties |

**Recommended approach:** Use separate repositories for application source code and deployment configuration.

```
Polyrepo Strategy (Recommended)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  App Repo (my-org/my-app)           Config Repo (my-org/my-app-config)
  ┌──────────────────────────┐       ┌───────────────────────────────┐
  │ src/                     │       │ base/                         │
  │ tests/                   │       │   deployment.yaml             │
  │ Dockerfile               │       │   service.yaml                │
  │ .github/workflows/       │       │   kustomization.yaml          │
  │   ci.yaml  ──────────────┼──────▶│ overlays/                     │
  │                          │ (CI   │   dev/                        │
  │ Application source code  │ updates│   staging/                   │
  │ Build, test, push image  │ image │   production/                 │
  └──────────────────────────┘ tag)  └───────────────────────────────┘
                                              ▲
                                              │ GitOps agent watches
                                              │ and reconciles
```

### Environment Branches vs Directory-Based

**Environment branches** use separate Git branches for each environment:

| Pros | Cons |
|---|---|
| Clear branch-per-environment mapping | Hard to see differences across environments |
| Branch protection rules per environment | Cherry-pick or merge conflicts between branches |
| Familiar Git branching workflow | Difficult to promote changes atomically |

**Directory-based** uses a single branch with directories per environment:

| Pros | Cons |
|---|---|
| Single source of truth on one branch | Branch protection applies uniformly |
| Easy to diff environments (`diff dev/ prod/`) | All environments visible in same branch |
| Kustomize overlays work naturally | Requires disciplined PR review for prod paths |
| Atomic promotion via PR | |

### Recommended Repository Layout

```
cluster-config/
├── clusters/
│   ├── dev/
│   │   ├── flux-system/         # Flux bootstrap (auto-generated)
│   │   ├── infrastructure.yaml  # Kustomization pointing to infra
│   │   └── applications.yaml   # Kustomization pointing to apps
│   ├── staging/
│   │   ├── flux-system/
│   │   ├── infrastructure.yaml
│   │   └── applications.yaml
│   └── production/
│       ├── flux-system/
│       ├── infrastructure.yaml
│       └── applications.yaml
├── infrastructure/
│   ├── sources/                 # HelmRepository, GitRepository
│   ├── configs/                 # ConfigMaps, cluster-wide resources
│   └── controllers/             # Ingress, cert-manager, monitoring
└── applications/
    ├── base/                    # Shared base manifests
    │   ├── my-app/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── kustomization.yaml
    │   └── another-app/
    └── overlays/                # Environment-specific overrides
        ├── dev/
        ├── staging/
        └── production/
```

---

## Secrets Management in GitOps

### The Secrets Problem

GitOps requires all desired state to live in Git, but secrets (passwords, API keys, TLS certificates) must **never** be stored as plaintext in a Git repository. This creates a tension that requires dedicated solutions.

```
The Secrets Challenge
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Git Repository                    Kubernetes Cluster
  ┌─────────────────┐              ┌─────────────────┐
  │ deployment.yaml │              │ Deployment      │
  │ service.yaml    │  ──────────▶ │ Service         │
  │ ???secret???    │   How to     │ Secret (needed) │
  │                 │   bridge     │                 │
  │ Must not store  │   this gap?  │ Needs actual    │
  │ plaintext       │              │ secret values   │
  └─────────────────┘              └─────────────────┘
```

### Sealed Secrets

**Sealed Secrets** (by Bitnami) encrypts secrets client-side so they can be safely stored in Git. A cluster-side controller decrypts them into regular Kubernetes Secrets.

```yaml
# Encrypt a secret using kubeseal CLI
# kubeseal --format yaml < my-secret.yaml > my-sealed-secret.yaml

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-app-secret
  namespace: my-app
spec:
  encryptedData:
    DATABASE_URL: AgBy3i4OJSWK+PiTySYZZA9rO...  # encrypted
    API_KEY: AgCtr8HGTO2hx+PJYC3SkOPp0...       # encrypted
  template:
    metadata:
      name: my-app-secret
      namespace: my-app
    type: Opaque
```

**How it works:**

1. The controller generates a public/private key pair in the cluster
2. You encrypt secrets locally using `kubeseal` with the public key
3. The encrypted SealedSecret is committed to Git
4. The controller in the cluster decrypts it into a regular Secret

### SOPS

**SOPS** (Secrets OPerationS by Mozilla) encrypts specific values in YAML/JSON files, leaving keys and structure visible. It integrates with cloud KMS services (AWS KMS, GCP KMS, Azure Key Vault) and PGP/age keys.

```yaml
# Encrypted with SOPS — keys visible, values encrypted
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
type: Opaque
stringData:
  DATABASE_URL: ENC[AES256_GCM,data:kD83kf9gT...,type:str]
  API_KEY: ENC[AES256_GCM,data:2hx+PJYC3...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:123456789:key/abcd-1234
  encrypted_regex: "^(data|stringData)$"
  version: 3.8.1
```

```bash
# Encrypt a file with SOPS and age
sops --encrypt --age age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml

# Decrypt locally for editing
sops secret.enc.yaml
```

**Flux has native SOPS support** — add a decryption provider to Kustomization:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./manifests
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age                # Secret containing the age private key
```

### External Secrets Operator

**External Secrets Operator (ESO)** synchronizes secrets from external secret stores (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault) into Kubernetes Secrets.

```yaml
# SecretStore — defines the connection to the external provider
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: my-app
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
# ExternalSecret — defines which secrets to sync
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secret
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: my-app-secret
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: my-app/production/database-url
    - secretKey: API_KEY
      remoteRef:
        key: my-app/production/api-key
```

### HashiCorp Vault Integration

Both ArgoCD and Flux can integrate with **HashiCorp Vault** through plugins or the External Secrets Operator.

```yaml
# External Secrets Operator with Vault backend
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: my-app
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app"
          serviceAccountRef:
            name: my-app-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-vault-secret
  namespace: my-app
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-app-secret
  dataFrom:
    - extract:
        key: secret/data/my-app/production
```

### Secrets Solutions Comparison

| Solution | Encryption Location | Key Management | GitOps Native | Multi-Cloud |
|---|---|---|---|---|
| **Sealed Secrets** | Client-side (kubeseal) | Controller-managed RSA keys | Yes — encrypted resource in Git | No — cluster-specific keys |
| **SOPS** | Client-side (sops CLI) | AWS KMS, GCP KMS, Azure KV, age, PGP | Yes — Flux native support | Yes — cloud KMS agnostic |
| **External Secrets Operator** | External provider | Managed by cloud provider | Yes — ExternalSecret CRD in Git | Yes — supports all major clouds |
| **HashiCorp Vault** | Vault server | Vault-managed (auto-unseal) | Via ESO or ArgoCD Vault Plugin | Yes — cloud agnostic |

---

## Multi-Cluster and Multi-Tenancy Patterns

### Multi-Cluster with ArgoCD

ArgoCD supports managing multiple clusters from a single ArgoCD instance (hub-spoke model):

```bash
# Register an external cluster with ArgoCD
argocd cluster add my-staging-cluster --name staging
argocd cluster add my-prod-cluster --name production

# List registered clusters
argocd cluster list
```

```yaml
# Deploy to a remote cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/my-org/my-app-config.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://prod-cluster.example.com   # Remote cluster
    namespace: my-app
```

```
ArgoCD Multi-Cluster (Hub-Spoke)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                 ┌──────────────────┐
                 │   Management     │
                 │   Cluster        │
                 │                  │
                 │   ┌──────────┐   │
                 │   │  ArgoCD  │   │
                 │   └────┬─────┘   │
                 └────────┼─────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
    ┌──────────────┐ ┌──────────┐ ┌──────────────┐
    │ Dev Cluster  │ │ Staging  │ │ Prod Cluster │
    │              │ │ Cluster  │ │              │
    │ my-app-dev   │ │ my-app   │ │ my-app-prod  │
    └──────────────┘ └──────────┘ └──────────────┘
```

**ArgoCD Projects** provide multi-tenancy by restricting what each team can deploy:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: argocd
spec:
  description: "Frontend team project"
  sourceRepos:
    - 'https://github.com/my-org/frontend-*'
  destinations:
    - namespace: 'frontend-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
  roles:
    - name: developer
      description: "Frontend developers"
      policies:
        - p, proj:team-frontend:developer, applications, get, team-frontend/*, allow
        - p, proj:team-frontend:developer, applications, sync, team-frontend/*, allow
```

### Multi-Cluster with Flux

Flux follows a **flat model** — each cluster runs its own Flux instance. A management repository coordinates all clusters:

```
Flux Multi-Cluster (Flat Model)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Git Repository (fleet-infra)
  ┌─────────────────────────────────────────┐
  │ clusters/                               │
  │   ├── dev/                              │
  │   │   └── flux-system/ (bootstrapped)   │──────▶ Dev Cluster (own Flux)
  │   ├── staging/                          │
  │   │   └── flux-system/ (bootstrapped)   │──────▶ Staging Cluster (own Flux)
  │   └── production/                       │
  │       └── flux-system/ (bootstrapped)   │──────▶ Prod Cluster (own Flux)
  │                                         │
  │ infrastructure/ (shared)                │
  │ applications/ (shared)                  │
  └─────────────────────────────────────────┘
```

### Multi-Tenancy Patterns

**Namespace-based tenancy** — each team gets its own namespace(s):

```yaml
# Flux: per-tenant Kustomization with service account impersonation
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-frontend
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: frontend
  sourceRef:
    kind: GitRepository
    name: team-frontend-repo
  path: ./manifests
  prune: true
  serviceAccountName: team-frontend-sa   # Impersonate tenant SA for RBAC
```

---

## Progressive Delivery with GitOps

Progressive delivery extends GitOps with advanced deployment strategies — canary releases, blue-green deployments, and A/B testing — while keeping Git as the source of truth.

### Argo Rollouts

**Argo Rollouts** is a Kubernetes controller that provides advanced deployment strategies. It replaces standard Kubernetes Deployments with a **Rollout** resource.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: ghcr.io/my-org/my-app:v1.2.3
          ports:
            - containerPort: 8080
  strategy:
    canary:
      canaryService: my-app-canary
      stableService: my-app-stable
      trafficRouting:
        istio:
          virtualServices:
            - name: my-app-vsvc
              routes:
                - primary
      steps:
        - setWeight: 10          # Send 10% traffic to canary
        - pause: { duration: 5m }
        - setWeight: 30          # Increase to 30%
        - pause: { duration: 5m }
        - analysis:              # Run automated analysis
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: my-app-canary
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100         # Full promotion
```

```yaml
# AnalysisTemplate — defines automated canary validation
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status=~"2.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

### Flagger

**Flagger** (a CNCF project by Flux) is a progressive delivery operator that works with Istio, Linkerd, App Mesh, NGINX, Contour, and Gloo. It uses standard Kubernetes Deployments (no custom resource replacement).

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: my-app
  namespace: my-app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  service:
    port: 8080
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  analysis:
    interval: 1m
    threshold: 5                    # Max failed checks before rollback
    maxWeight: 50                   # Max canary traffic percentage
    stepWeight: 10                  # Increment per step
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500                  # Milliseconds
        interval: 1m
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.flagger-system/
        metadata:
          cmd: "hey -z 2m -q 10 -c 2 http://my-app-canary.my-app:8080/"
```

---

## Drift Detection and Reconciliation

### What is Drift

**Drift** occurs when the actual state of a resource in the cluster diverges from the desired state declared in Git. Common causes include:

- Manual changes via `kubectl edit` or `kubectl patch`
- Admission webhooks or mutating controllers modifying resources
- Operators updating resource status or annotations
- Direct API access by debugging tools or scripts

### Detection Mechanisms

| Tool | Detection Method | Default Interval |
|---|---|---|
| **ArgoCD** | Compares live manifests with rendered Git manifests using a three-way diff | 3 minutes (configurable) |
| **Flux** | Compares applied resource hash with cluster resource hash via server-side apply | Per Kustomization/HelmRelease interval |

**ArgoCD drift detection configuration:**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 180s      # How often to check for drift (default 3m)
```

**Ignoring expected differences** — some fields are legitimately modified by controllers:

```yaml
# ArgoCD: ignore specific fields in diff
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas              # Ignore HPA-managed replicas
    - group: ""
      kind: Service
      jqPathExpressions:
        - .spec.clusterIP             # Ignore cluster-assigned IP
```

### Reconciliation Strategies

| Strategy | Description | Use Case |
|---|---|---|
| **Auto-sync + self-heal** | Automatically revert any drift back to Git state | Production workloads where Git is strictly authoritative |
| **Auto-sync without self-heal** | Sync on Git changes but allow live modifications | Development environments with manual debugging |
| **Manual sync** | Require explicit sync trigger from UI or CLI | Critical environments requiring human approval |
| **Diff-only (notify)** | Detect and alert on drift without auto-correcting | Audit and compliance monitoring |

---

## Comparison: ArgoCD vs Flux

| Feature | ArgoCD | Flux |
|---|---|---|
| **CNCF status** | Graduated | Graduated |
| **Architecture** | Monolithic (single install) | Modular (GitOps Toolkit controllers) |
| **UI** | Built-in web UI with app visualization | No built-in UI (use Weave GitOps or Capacitor) |
| **CLI** | `argocd` CLI | `flux` CLI |
| **Multi-cluster** | Hub-spoke — single ArgoCD manages many clusters | Flat — each cluster runs its own Flux instance |
| **Application CRD** | Application, ApplicationSet, AppProject | GitRepository, Kustomization, HelmRelease |
| **Manifest rendering** | Helm, Kustomize, Jsonnet, plain YAML, custom plugins | Kustomize, Helm (via controllers) |
| **SOPS support** | Via plugin (argocd-vault-plugin or custom) | Native — built into Kustomize Controller |
| **Image automation** | No built-in (use Argo CD Image Updater, separate project) | Built-in (Image Reflector + Automation controllers) |
| **Notifications** | Built-in notification engine | Built-in Notification Controller |
| **RBAC** | Built-in RBAC with AppProject-level policies | Kubernetes-native RBAC with service account impersonation |
| **Diff/Drift** | Three-way diff with UI visualization | Server-side apply with hash comparison |
| **Webhooks** | Supports GitHub/GitLab/Bitbucket webhooks | Supports generic and provider-specific webhooks |
| **Helm management** | Full Helm lifecycle (install, upgrade, rollback) | Full Helm lifecycle with test and remediation support |
| **Learning curve** | Lower — UI helps visualization and onboarding | Higher — CLI-first, requires understanding of CRD composition |
| **Extensibility** | Config Management Plugins (CMP) | GitOps Toolkit APIs — build custom controllers |

**When to choose ArgoCD:**

- You want a rich built-in UI for visualization and onboarding
- You prefer a hub-spoke model for multi-cluster management
- Your team uses Jsonnet or custom config management tools
- You need fine-grained RBAC with project-level policies

**When to choose Flux:**

- You prefer a modular, composable architecture
- You want native SOPS integration for secrets
- You need built-in image automation for auto-updating tags
- You prefer Kubernetes-native RBAC over a custom RBAC layer
- You are building a platform and want to extend GitOps with custom controllers

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Foundational CI/CD concepts, principles, and pipeline design |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build, test, and integration automation |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment strategies, release management, and delivery pipelines |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflow authoring, actions marketplace, and CI/CD with GitHub |
| [04-AZURE-DEVOPS.md](04-AZURE-DEVOPS.md) | Azure DevOps | Azure Pipelines, Boards, Repos, and Artifacts |
| [05-JENKINS.md](05-JENKINS.md) | Jenkins | Jenkins pipelines, distributed builds, and plugin ecosystem |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial GitOps documentation |
