# Controllers

## Table of Contents

1. [Overview](#overview)
   - [What Are Controllers](#what-are-controllers)
   - [The Control Loop](#the-control-loop)
2. [Built-in Controllers](#built-in-controllers)
   - [Deployment Controller](#deployment-controller)
   - [ReplicaSet Controller](#replicaset-controller)
   - [StatefulSet Controller](#statefulset-controller)
   - [DaemonSet Controller](#daemonset-controller)
   - [Job Controller](#job-controller)
   - [CronJob Controller](#cronjob-controller)
   - [Node Controller](#node-controller)
   - [Service Controller](#service-controller)
   - [EndpointSlice Controller](#endpointslice-controller)
   - [Namespace Controller](#namespace-controller)
3. [Controller Architecture](#controller-architecture)
   - [Informers and SharedInformers](#informers-and-sharedinformers)
   - [Listers and Caches](#listers-and-caches)
   - [Work Queues](#work-queues)
   - [Event Handlers](#event-handlers)
   - [Leader Election for HA](#leader-election-for-ha)
4. [The Reconciliation Loop](#the-reconciliation-loop)
   - [Desired State vs Actual State](#desired-state-vs-actual-state)
   - [Idempotency](#idempotency)
   - [Level-Triggered vs Edge-Triggered](#level-triggered-vs-edge-triggered)
   - [Requeue Strategies](#requeue-strategies)
5. [Building Custom Controllers](#building-custom-controllers)
   - [Using client-go](#using-client-go)
   - [Using controller-runtime](#using-controller-runtime)
   - [Using Python (kopf)](#using-python-kopf)
   - [Using Java (Java Operator SDK)](#using-java-java-operator-sdk)
6. [Controller Best Practices](#controller-best-practices)
   - [Idempotent Reconciliation](#idempotent-reconciliation)
   - [Status Subresource Updates](#status-subresource-updates)
   - [Finalizers for Cleanup](#finalizers-for-cleanup)
   - [Owner References and Garbage Collection](#owner-references-and-garbage-collection)
   - [Rate Limiting and Backoff](#rate-limiting-and-backoff)
   - [Metrics and Observability](#metrics-and-observability)
7. [Debugging Controllers](#debugging-controllers)
   - [Logging Patterns](#logging-patterns)
   - [Events](#events)
   - [Common Failure Modes](#common-failure-modes)
8. [Next Steps](#next-steps)

## Overview

This document provides a deep dive into **Kubernetes controllers** — the core mechanism that drives the platform's declarative model. Controllers are the bridge between the desired state you declare in manifests and the actual state running in your cluster.

### Target Audience

- Platform engineers building or extending Kubernetes
- Developers writing custom controllers or operators
- SREs debugging controller behavior in production
- DevOps engineers seeking a deeper understanding of Kubernetes internals

### Scope

- Built-in controller behavior and reconciliation logic
- Internal architecture: informers, caches, work queues
- Building custom controllers in Go, Python, and Java
- Best practices, debugging techniques, and common pitfalls

### What Are Controllers

A **controller** is a control loop that watches the state of your cluster through the API server and makes changes to move the current state toward the desired state. Every Kubernetes resource that "does something" — scaling Pods, scheduling Jobs, managing Nodes — has a controller behind it.

Controllers do **not** run as part of the API server. They are separate processes (typically packaged in `kube-controller-manager`) that communicate exclusively through the Kubernetes API.

### The Control Loop

Every controller follows the same fundamental pattern: **Observe → Diff → Act**.

```
┌─────────────────────────────────────────────────┐
│                 CONTROL LOOP                    │
│                                                 │
│   ┌──────────┐   ┌────────┐   ┌─────────────┐  │
│   │ OBSERVE  │──▶│  DIFF  │──▶│     ACT     │  │
│   │          │   │        │   │             │  │
│   │ Read the │   │Compare │   │ Create,     │  │
│   │ current  │   │desired │   │ update, or  │  │
│   │ state    │   │vs real │   │ delete      │  │
│   └──────────┘   └────────┘   └─────────────┘  │
│        ▲                            │           │
│        └────────────────────────────┘           │
│                                                 │
│        API Server (single source of truth)      │
└─────────────────────────────────────────────────┘
```

| Step    | Description                                                     |
|---------|-----------------------------------------------------------------|
| Observe | Read the current state of the resource from the API server      |
| Diff    | Compare what exists with the desired state (the spec)           |
| Act     | Make API calls to create, update, or delete child resources     |

## Built-in Controllers

Kubernetes ships with dozens of controllers bundled inside `kube-controller-manager`. Below are the most important ones.

### Deployment Controller

**Manages:** Deployment resources and their child ReplicaSets.

The Deployment controller implements rolling updates, rollbacks, and scaling. When you update a Deployment's Pod template, the controller creates a new ReplicaSet with the updated template and gradually scales it up while scaling down the old ReplicaSet.

**Reconciliation behavior:**
- Compares `.spec.replicas` and `.spec.template` against existing ReplicaSets
- Creates a new ReplicaSet when the Pod template changes
- Manages `maxSurge` and `maxUnavailable` during rollouts
- Records revision history for rollback support

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

### ReplicaSet Controller

**Manages:** ReplicaSet resources and their child Pods.

The ReplicaSet controller ensures that the correct number of Pod replicas are running at all times. If a Pod is deleted or fails, the controller creates a replacement. If there are too many Pods, it terminates the excess.

**Reconciliation behavior:**
- Counts Pods matching the label selector
- Creates or deletes Pods to match `.spec.replicas`
- Uses owner references to track which Pods it owns

> **Note:** You rarely create ReplicaSets directly. Deployments manage them for you.

### StatefulSet Controller

**Manages:** StatefulSet resources and their ordered, uniquely identified Pods.

Unlike Deployments, the StatefulSet controller creates Pods with **stable network identities** (`pod-0`, `pod-1`, …) and **persistent storage**. Pods are created and deleted in strict sequential order.

**Reconciliation behavior:**
- Creates Pods sequentially (waits for `pod-N` to be Ready before creating `pod-N+1`)
- Maintains PersistentVolumeClaims across Pod rescheduling
- Supports ordered rolling updates (reverse ordinal order by default)
- Handles `partition` for canary-style staged rollouts

### DaemonSet Controller

**Manages:** DaemonSet resources — ensures a copy of a Pod runs on every (or a subset of) Node(s).

**Reconciliation behavior:**
- Watches the Node list and compares it to existing Pods
- Creates a Pod on each eligible Node that does not already have one
- Removes Pods from Nodes that no longer match the `nodeSelector` or tolerations
- Uses `ControllerRevision` objects for rolling update history

Common use cases: log collectors (Fluentd), monitoring agents (Prometheus Node Exporter), storage daemons (Ceph), CNI plugins.

### Job Controller

**Manages:** Job resources — runs Pods to completion.

**Reconciliation behavior:**
- Creates Pods to execute work units
- Tracks `.status.succeeded` and `.status.failed` counts
- Respects `.spec.backoffLimit` for retry logic
- Supports parallel execution via `.spec.parallelism` and `.spec.completions`
- Cleans up completed Pods based on `ttlSecondsAfterFinished`

### CronJob Controller

**Manages:** CronJob resources — creates Jobs on a cron schedule.

**Reconciliation behavior:**
- Evaluates `.spec.schedule` against the current time
- Creates a new Job when the schedule fires
- Enforces `.spec.concurrencyPolicy` (Allow, Forbid, Replace)
- Tracks missed schedules and honors `.spec.startingDeadlineSeconds`
- Maintains `.spec.successfulJobsHistoryLimit` and `.spec.failedJobsHistoryLimit`

### Node Controller

**Manages:** Node lifecycle within the cluster.

**Reconciliation behavior:**
- Monitors Node heartbeats (Lease objects in `kube-node-lease` namespace)
- Sets Node conditions (`Ready`, `MemoryPressure`, `DiskPressure`, etc.)
- Assigns CIDR blocks to Nodes when `--allocate-node-cidrs` is enabled
- Initiates Pod eviction when a Node becomes `NotReady` (after `--pod-eviction-timeout`)

### Service Controller

**Manages:** Service resources of type `LoadBalancer`.

**Reconciliation behavior:**
- Calls the cloud provider API to provision an external load balancer
- Updates `Service.status.loadBalancer.ingress` with the external IP/hostname
- Reconciles health checks and listener configuration
- Deletes the load balancer when the Service is removed

> **Note:** This controller only runs when the cluster is configured with a cloud provider. ClusterIP and NodePort Services are handled by `kube-proxy`, not a controller.

### EndpointSlice Controller

**Manages:** EndpointSlice resources, which map Services to backend Pod IPs.

**Reconciliation behavior:**
- Watches Services and Pods, creating EndpointSlice objects that list ready Pod IPs
- Splits endpoints across multiple EndpointSlice objects (default limit: 100 endpoints per slice)
- Updates slices when Pods become ready/unready or are added/removed
- Replaced the legacy Endpoints controller for improved scalability

### Namespace Controller

**Manages:** Namespace deletion and cascading cleanup.

**Reconciliation behavior:**
- When a Namespace is set to `Terminating`, enumerates all resources in that namespace
- Deletes all resources (Pods, Services, ConfigMaps, Secrets, CRDs, etc.)
- Removes the `kubernetes` finalizer once all resources are cleaned up
- Marks the Namespace as fully deleted

## Controller Architecture

All Kubernetes controllers share a common internal architecture built on top of the `client-go` library.

```
┌──────────────────────────────────────────────────────────────────┐
│                     CONTROLLER ARCHITECTURE                      │
│                                                                  │
│  API Server                                                      │
│      │                                                           │
│      ▼                                                           │
│  ┌──────────┐    ┌─────────────────┐    ┌──────────────────────┐ │
│  │ Reflector │──▶│  Informer Cache  │──▶│    Event Handlers    │ │
│  │ (List &   │   │  (Thread-safe    │   │  (Add/Update/Delete) │ │
│  │  Watch)   │   │   in-memory      │   │                      │ │
│  │          │   │   store)          │   │                      │ │
│  └──────────┘   └─────────────────┘   └──────────┬───────────┘ │
│                         │                          │             │
│                         ▼                          ▼             │
│                  ┌─────────────┐          ┌──────────────┐       │
│                  │   Lister    │          │  Work Queue  │       │
│                  │ (Read from  │          │ (Rate-limited│       │
│                  │  cache)     │          │  deduped)    │       │
│                  └─────────────┘          └──────┬───────┘       │
│                         │                        │               │
│                         ▼                        ▼               │
│                  ┌───────────────────────────────────────┐       │
│                  │         RECONCILE FUNCTION            │       │
│                  │  (Business logic: observe → diff → act)│       │
│                  └───────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### Informers and SharedInformers

An **Informer** establishes a long-lived watch connection to the API server and maintains a local cache of resources. A **SharedInformer** multiplexes a single API watch across multiple consumers to reduce API server load.

Key properties:
- **List** on startup to populate the cache, then **Watch** for incremental updates
- Automatically reconnects and re-lists if the watch connection drops
- Supports resource version tracking to resume watches efficiently

```go
// Creating a SharedInformer for Pods
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
podInformer := factory.Core().V1().Pods()

// Start all informers
factory.Start(stopCh)

// Wait for cache sync before processing
factory.WaitForCacheSync(stopCh)
```

### Listers and Caches

**Listers** provide read-only access to the informer's in-memory cache. Reading from a Lister is fast and does not hit the API server.

```go
// Read Pods from the local cache — no API call
pods, err := podInformer.Lister().Pods("default").List(labels.Everything())
if err != nil {
    log.Fatalf("failed to list pods: %v", err)
}
```

> **Note:** Cache data may be slightly stale (bounded by the informer resync period). For critical reads where staleness matters, use a direct API call instead of the cache.

### Work Queues

Controllers use **work queues** to decouple event handling from reconciliation. When an informer fires an event, the handler enqueues the resource key (`namespace/name`). Worker goroutines then dequeue items and run the reconcile function.

| Queue Type     | Behavior                                                   |
|----------------|------------------------------------------------------------|
| Simple         | FIFO, deduplication only                                   |
| Rate-limited   | Adds backoff delay between retries of the same key         |
| Delayed        | Enqueues items to be processed after a specified duration   |

```go
queue := workqueue.NewRateLimitingQueue(
    workqueue.DefaultControllerRateLimiter(),
)

// Enqueue a resource key
queue.Add("default/my-pod")

// Enqueue with a delay
queue.AddAfter("default/my-pod", 30*time.Second)

// Worker loop
for {
    key, shutdown := queue.Get()
    if shutdown {
        return
    }
    err := reconcile(key.(string))
    if err != nil {
        queue.AddRateLimited(key) // retry with backoff
    } else {
        queue.Forget(key)         // reset backoff
    }
    queue.Done(key)
}
```

### Event Handlers

Informers fire three event types. Handlers typically extract the resource key and enqueue it for processing.

```go
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        key, _ := cache.MetaNamespaceKeyFunc(obj)
        queue.Add(key)
    },
    UpdateFunc: func(oldObj, newObj interface{}) {
        key, _ := cache.MetaNamespaceKeyFunc(newObj)
        queue.Add(key)
    },
    DeleteFunc: func(obj interface{}) {
        key, _ := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
        queue.Add(key)
    },
})
```

> **Note:** The work queue deduplicates keys. If a resource changes three times before the worker processes it, it only reconciles once — using the latest state from the cache.

### Leader Election for HA

In production, `kube-controller-manager` runs with multiple replicas but only **one active leader**. Leader election ensures that only one instance runs the control loops at a time.

```go
leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock: &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "my-controller",
            Namespace: "kube-system",
        },
        Client: clientset.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{
            Identity: hostname,
        },
    },
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            runControllers(ctx)
        },
        OnStoppedLeading: func() {
            log.Fatal("lost leader election")
        },
    },
})
```

## The Reconciliation Loop

### Desired State vs Actual State

The reconciliation loop is the heart of every controller. It answers one question: **does the actual state match the desired state?** If not, it takes corrective action.

```
Desired State (spec)          Actual State (cluster)
──────────────────           ──────────────────────
replicas: 3                  2 running Pods
image: nginx:1.25            1 Pod running nginx:1.24

                    ▼ Reconcile ▼

Action: create 1 Pod, update 1 Pod image
```

The desired state is declared in the resource's `.spec`. The actual state is discovered by listing child resources (Pods, Services, etc.) from the cache or directly from the API.

### Idempotency

A reconcile function must be **idempotent** — calling it multiple times with the same input must produce the same result. This is critical because controllers may be restarted, events may be replayed, or the same key may be requeued multiple times.

✅ **Idempotent:** "Ensure 3 Pods exist" — safe to call repeatedly
❌ **Not idempotent:** "Create 1 Pod" — calling twice creates 2 Pods

### Level-Triggered vs Edge-Triggered

Kubernetes controllers are **level-triggered**, not edge-triggered. This is a crucial design decision:

| Approach         | Meaning                            | Risk                                  |
|------------------|------------------------------------|---------------------------------------|
| Edge-triggered   | React to individual change events  | Missed events cause permanent drift   |
| Level-triggered  | React to current state every time  | Self-healing; missed events are OK    |

A level-triggered controller does not care **what** changed — it reads the full current state and drives toward the desired state regardless. This makes controllers resilient to missed events, restarts, and partial failures.

### Requeue Strategies

When reconciliation fails or is not yet complete, the controller requeues the key for another attempt.

| Strategy             | When to Use                                        | Example                        |
|----------------------|----------------------------------------------------|--------------------------------|
| Immediate requeue    | Temporary error, worth retrying right away         | `return ctrl.Result{Requeue: true}, nil` |
| Requeue after delay  | Waiting for an external condition to change        | `return ctrl.Result{RequeueAfter: 30 * time.Second}, nil` |
| Rate-limited requeue | Persistent error with exponential backoff          | `return ctrl.Result{}, err`    |
| No requeue           | Reconciliation succeeded, no further work needed   | `return ctrl.Result{}, nil`    |

## Building Custom Controllers

### Using client-go

The `client-go` library provides the foundational building blocks for controllers in Go. This approach gives you full control but requires more boilerplate.

```go
package main

import (
    "context"
    "fmt"
    "time"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/util/wait"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/workqueue"
)

func main() {
    config, _ := clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
    clientset, _ := kubernetes.NewForConfig(config)

    factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
    podInformer := factory.Core().V1().Pods()
    queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            key, _ := cache.MetaNamespaceKeyFunc(obj)
            queue.Add(key)
        },
        UpdateFunc: func(old, new interface{}) {
            key, _ := cache.MetaNamespaceKeyFunc(new)
            queue.Add(key)
        },
        DeleteFunc: func(obj interface{}) {
            key, _ := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
            queue.Add(key)
        },
    })

    stopCh := make(chan struct{})
    factory.Start(stopCh)
    factory.WaitForCacheSync(stopCh)

    // Worker loop
    wait.Until(func() {
        for {
            key, shutdown := queue.Get()
            if shutdown {
                return
            }
            func() {
                defer queue.Done(key)
                namespace, name, _ := cache.SplitMetaNamespaceKey(key.(string))
                pod, err := podInformer.Lister().Pods(namespace).Get(name)
                if err != nil {
                    queue.AddRateLimited(key)
                    return
                }
                fmt.Printf("Reconciling Pod %s/%s (phase: %s)\n", namespace, name, pod.Status.Phase)
                queue.Forget(key)
            }()
        }
    }, time.Second, stopCh)
}
```

### Using controller-runtime

The `controller-runtime` library (used by Kubebuilder and Operator SDK) provides higher-level abstractions that eliminate most boilerplate.

```go
package main

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
)

// DeploymentWatcherReconciler watches Deployments and logs replica counts.
type DeploymentWatcherReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *DeploymentWatcherReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {
    var deployment appsv1.Deployment
    if err := r.Get(ctx, req.NamespacedName, &deployment); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    fmt.Printf("Deployment %s: desired=%d, available=%d\n",
        req.NamespacedName,
        *deployment.Spec.Replicas,
        deployment.Status.AvailableReplicas,
    )

    return ctrl.Result{}, nil
}

func main() {
    ctrl.SetLogger(zap.New())

    mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{})

    _ = ctrl.NewControllerManagedBy(mgr).
        For(&appsv1.Deployment{}).
        Owns(&corev1.Pod{}).
        Complete(&DeploymentWatcherReconciler{
            Client: mgr.GetClient(),
            Scheme: mgr.GetScheme(),
        })

    _ = mgr.Start(ctrl.SetupSignalHandler())
}
```

### Using Python (kopf)

The [kopf](https://kopf.readthedocs.io/) framework makes it simple to write controllers in Python.

```python
import kopf
import kubernetes

@kopf.on.create('mygroup.example.com', 'v1', 'myresources')
def on_create(spec, name, namespace, **kwargs):
    """Handle creation of a MyResource."""
    replicas = spec.get('replicas', 1)

    api = kubernetes.client.AppsV1Api()
    deployment = kubernetes.client.V1Deployment(
        metadata=kubernetes.client.V1ObjectMeta(name=name),
        spec=kubernetes.client.V1DeploymentSpec(
            replicas=replicas,
            selector=kubernetes.client.V1LabelSelector(
                match_labels={"app": name},
            ),
            template=kubernetes.client.V1PodTemplateSpec(
                metadata=kubernetes.client.V1ObjectMeta(
                    labels={"app": name},
                ),
                spec=kubernetes.client.V1PodSpec(
                    containers=[
                        kubernetes.client.V1Container(
                            name="app",
                            image=spec.get('image', 'nginx'),
                        )
                    ]
                ),
            ),
        ),
    )
    api.create_namespaced_deployment(namespace=namespace, body=deployment)
    return {"message": f"Deployment {name} created with {replicas} replicas"}


@kopf.on.update('mygroup.example.com', 'v1', 'myresources')
def on_update(spec, name, namespace, **kwargs):
    """Handle updates to a MyResource."""
    api = kubernetes.client.AppsV1Api()
    api.patch_namespaced_deployment(
        name=name,
        namespace=namespace,
        body={"spec": {"replicas": spec.get('replicas', 1)}},
    )
```

### Using Java (Java Operator SDK)

The [Java Operator SDK](https://javaoperatorsdk.io/) provides a framework for building controllers in Java.

```java
@ControllerConfiguration
public class MyResourceReconciler
    implements Reconciler<MyResource>, Cleaner<MyResource> {

    @Override
    public UpdateControl<MyResource> reconcile(
            MyResource resource,
            Context<MyResource> context) {

        String name = resource.getMetadata().getName();
        int replicas = resource.getSpec().getReplicas();

        // Build the desired Deployment
        Deployment deployment = new DeploymentBuilder()
            .withNewMetadata()
                .withName(name)
                .withNamespace(resource.getMetadata().getNamespace())
            .endMetadata()
            .withNewSpec()
                .withReplicas(replicas)
                .withNewSelector()
                    .addToMatchLabels("app", name)
                .endSelector()
                .withNewTemplate()
                    .withNewMetadata()
                        .addToLabels("app", name)
                    .endMetadata()
                    .withNewSpec()
                        .addNewContainer()
                            .withName("app")
                            .withImage(resource.getSpec().getImage())
                        .endContainer()
                    .endSpec()
                .endTemplate()
            .endSpec()
            .build();

        context.getClient().apps().deployments()
            .inNamespace(resource.getMetadata().getNamespace())
            .createOrReplace(deployment);

        // Update status
        resource.getStatus().setReadyReplicas(replicas);
        return UpdateControl.patchStatus(resource);
    }

    @Override
    public DeleteControl cleanup(
            MyResource resource,
            Context<MyResource> context) {
        // Cleanup logic when the resource is deleted
        return DeleteControl.defaultDelete();
    }
}
```

## Controller Best Practices

### Idempotent Reconciliation

- ✅ Use **server-side apply** or **create-or-update** patterns
- ✅ Check if a resource already exists before creating it
- ✅ Use `resourceVersion` for optimistic concurrency control
- ❌ Avoid counting events to determine what action to take
- ❌ Never assume the previous reconcile completed successfully

```go
// Create-or-update pattern in controller-runtime
op, err := ctrl.CreateOrUpdate(ctx, r.Client, &service, func() error {
    service.Spec.Ports = []corev1.ServicePort{{Port: 80}}
    service.Spec.Selector = map[string]string{"app": name}
    return ctrl.SetControllerReference(owner, &service, r.Scheme)
})
// op is either "created", "updated", or "unchanged"
```

### Status Subresource Updates

Always update the status subresource separately from the spec. This prevents conflicts between the controller updating status and users updating the spec.

```go
// Update status separately
resource.Status.Phase = "Running"
resource.Status.ReadyReplicas = 3
if err := r.Status().Update(ctx, resource); err != nil {
    return ctrl.Result{}, err
}
```

```yaml
# CRD must enable the status subresource
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      subresources:
        status: {}    # Enables /status subresource
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
            status:
              type: object
              properties:
                readyReplicas:
                  type: integer
                phase:
                  type: string
```

### Finalizers for Cleanup

Finalizers let a controller perform cleanup before a resource is deleted. Kubernetes will not remove the resource from etcd until all finalizers are cleared.

```go
const finalizerName = "mygroup.example.com/cleanup"

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var resource myv1.MyResource
    if err := r.Get(ctx, req.NamespacedName, &resource); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Resource is being deleted
    if !resource.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&resource, finalizerName) {
            // Perform external cleanup (e.g., delete cloud resources)
            if err := r.cleanupExternalResources(&resource); err != nil {
                return ctrl.Result{}, err
            }
            controllerutil.RemoveFinalizer(&resource, finalizerName)
            if err := r.Update(ctx, &resource); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(&resource, finalizerName) {
        controllerutil.AddFinalizer(&resource, finalizerName)
        if err := r.Update(ctx, &resource); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Normal reconciliation logic
    return r.reconcileNormal(ctx, &resource)
}
```

### Owner References and Garbage Collection

Set **owner references** on child resources so that Kubernetes automatically garbage collects them when the parent is deleted.

```go
// Set the owner reference so the Pod is deleted when the parent is deleted
if err := ctrl.SetControllerReference(owner, pod, r.Scheme); err != nil {
    return err
}
```

Kubernetes supports three deletion propagation policies:

| Policy       | Behavior                                              |
|--------------|-------------------------------------------------------|
| Foreground   | Children are deleted first, then the owner            |
| Background   | Owner is deleted immediately, children are GC'd later |
| Orphan       | Owner is deleted, children are left running           |

### Rate Limiting and Backoff

Protect both the controller and the API server from hot loops with rate limiting.

```go
// Custom rate limiter with exponential backoff
rateLimiter := workqueue.NewTypedMaxOfRateLimiter(
    workqueue.NewTypedItemExponentialFailureRateLimiter[string](
        5*time.Millisecond,   // base delay
        1000*time.Second,     // max delay
    ),
    &workqueue.TypedBucketRateLimiter[string]{
        Limiter: rate.NewLimiter(rate.Limit(10), 100), // 10 qps, burst 100
    },
)
```

### Metrics and Observability

Expose Prometheus metrics from your controller to monitor its health and performance.

| Metric                              | Purpose                                     |
|-------------------------------------|---------------------------------------------|
| `controller_reconcile_total`        | Total number of reconciliations              |
| `controller_reconcile_errors_total` | Number of failed reconciliations             |
| `controller_reconcile_duration`     | Histogram of reconciliation durations        |
| `workqueue_depth`                   | Current depth of the work queue              |
| `workqueue_adds_total`              | Total items added to the queue               |
| `workqueue_retries_total`           | Total number of retries                      |

> **Note:** `controller-runtime` automatically exposes work queue and reconcile metrics when you enable the metrics server.

## Debugging Controllers

### Logging Patterns

Use structured logging with consistent key-value pairs to make controller logs searchable.

```go
// Using controller-runtime's logr interface
log := ctrl.LoggerFrom(ctx)

log.Info("reconciling resource",
    "name", req.Name,
    "namespace", req.Namespace,
)

log.Error(err, "failed to create deployment",
    "deployment", deploymentName,
    "namespace", req.Namespace,
)

// Set log verbosity levels
// V(0) = info (always shown)
// V(1) = debug
// V(2) = trace
log.V(1).Info("cache hit", "key", cacheKey)
```

```bash
# View kube-controller-manager logs
kubectl logs -n kube-system -l component=kube-controller-manager --tail=100

# Filter for a specific controller
kubectl logs -n kube-system -l component=kube-controller-manager | grep "deployment"
```

### Events

Controllers should emit Kubernetes Events to provide user-visible feedback.

```go
// Record an event on the resource
recorder.Eventf(resource, corev1.EventTypeNormal, "Synced",
    "Successfully synced deployment %s", deploymentName)

recorder.Eventf(resource, corev1.EventTypeWarning, "SyncFailed",
    "Failed to create service: %v", err)
```

```bash
# View events for a resource
kubectl describe deployment web-app

# View all events in a namespace sorted by time
kubectl get events -n default --sort-by='.lastTimestamp'

# Watch events in real time
kubectl get events -n default --watch
```

### Common Failure Modes

| Symptom                          | Likely Cause                                    | Fix                                       |
|----------------------------------|-------------------------------------------------|-------------------------------------------|
| Controller not reconciling       | Informer cache not synced                       | Check `WaitForCacheSync` is called        |
| Rapid restart loop               | Panic in reconcile function                     | Add recover/error handling                |
| Resources not cleaning up        | Missing finalizer or owner reference            | Add finalizer or set owner reference      |
| "too old resource version" error | Watch bookmark expired                          | Informer will auto-recover; no fix needed |
| Duplicate child resources        | Non-idempotent reconcile logic                  | Use create-or-update pattern              |
| Stale cache reads                | Informer resync period too long                 | Reduce resync period or use direct API    |
| Leader election flapping         | Network partitions or aggressive timeouts       | Increase lease duration                   |
| High API server load             | Missing cache; listing directly from API server | Use informer listers, not direct API calls|

## Next Steps

- [CRDs and Operators](07-CRDS-AND-OPERATORS.md) — extend Kubernetes with custom resources and the operator pattern
- [Manifests](06-MANIFESTS.md) — Kubernetes resource definitions and YAML authoring
- [Services](01-SERVICES.md) — networking, load balancing, and service discovery
- [Kubernetes Official Docs: Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [client-go Examples](https://github.com/kubernetes/client-go/tree/master/examples)
- [Kubebuilder Book](https://book.kubebuilder.io/)
