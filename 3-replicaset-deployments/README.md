# Day 10 - ReplicationController, ReplicaSet and Deployment

## Objective

The goal of this lab is to understand how Kubernetes ensures that applications remain available and maintain the desired number of running instances.

This chapter introduces:

* ReplicationController (RC)
* ReplicaSet (RS)
* Deployment
* Labels and Selectors
* Rolling Updates
* Rollbacks

---

# Why Do We Need Controllers?

Imagine creating a Pod manually:

```bash
kubectl run nginx --image=nginx
```

If the Pod crashes or is deleted:

```text
Pod disappears
Application becomes unavailable
```

Kubernetes solves this problem using controllers.

A controller continuously compares:

```text
Desired State
vs
Current State
```

and performs corrective actions when necessary.

---

# ReplicationController (RC)

ReplicationController is the original Kubernetes controller responsible for maintaining a specified number of Pod replicas.

Example:

```text
Desired Replicas: 3
Running Replicas: 2
```

ReplicationController detects the difference and creates a new Pod.

---

## Example

```yaml
apiVersion: v1
kind: ReplicationController

metadata:
  name: nginx-rc

spec:
  replicas: 3

  selector:
    app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx
```

---

## Verify

```bash
kubectl get rc
kubectl get pods
```

Delete a Pod:

```bash
kubectl delete pod <pod-name>
```

Observe:

```bash
kubectl get pods
```

A replacement Pod will be created automatically.

---

## Limitations

ReplicationController only supports equality-based selectors:

```yaml
selector:
  app: nginx
```

It cannot use advanced matching logic.

---

# ReplicaSet (RS)

ReplicaSet is the successor to ReplicationController.

It provides the same replica management functionality while supporting more powerful label selectors.

---

## Example

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: nginx-rs

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx
```

---

## Verify

```bash
kubectl get rs
kubectl get pods
```

---

# Advanced Selectors

ReplicaSet supports:

```yaml
matchExpressions:
```

Example:

```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - dev
    - staging
```

This allows more flexible Pod selection.

---

# Deployment

Deployment is the standard way to run stateless applications in Kubernetes.

In production environments, you usually create Deployments rather than ReplicaSets directly.

---

## Deployment Architecture

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

---

## Example

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx:1.19
```

---

## Create Deployment

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get deploy
kubectl get rs
kubectl get pods
```

Expected:

```text
Deployment
└── ReplicaSet
    └── Pods
```

---

# Scaling Applications

Increase replicas:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get pods
```

Kubernetes automatically creates additional Pods.

---

# Rolling Updates

One of the most important Deployment features is rolling updates.

Update the container image:

```bash
kubectl set image deployment/nginx-deployment \
nginx=nginx:1.20
```

Check rollout status:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

# What Actually Happens?

Kubernetes does NOT modify existing Pods.

Instead:

```text
Old ReplicaSet
    ↓
New ReplicaSet Created
    ↓
New Pods Created
    ↓
Old Pods Removed
```

This is a critical concept for CKA.

---

# View ReplicaSets During an Update

```bash
kubectl get rs
```

Example:

```text
nginx-deployment-6d4cf56db
nginx-deployment-7b5fd4bcf
```

One ReplicaSet represents the old version.

One ReplicaSet represents the new version.

---

# Rollback

If a deployment update fails:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Kubernetes restores the previous ReplicaSet.

---

## View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

---

## Rollback to Specific Revision

```bash
kubectl rollout undo deployment/nginx-deployment \
--to-revision=1
```

---

# Labels and Selectors

Controllers depend heavily on labels.

Example:

```yaml
labels:
  app: nginx
```

Deployment selector:

```yaml
selector:
  matchLabels:
    app: nginx
```

Deployment uses the selector to identify which Pods belong to it.

---

# Important Rule

The Deployment selector must match the Pod template labels.

Correct:

```yaml
selector:
  matchLabels:
    app: nginx

template:
  metadata:
    labels:
      app: nginx
```

Incorrect:

```yaml
selector:
  matchLabels:
    app: nginx

template:
  metadata:
    labels:
      app: apache
```

---

# Self-Healing

Delete a Pod:

```bash
kubectl delete pod <pod-name>
```

ReplicaSet immediately creates a replacement.

This behavior is called:

```text
Self-Healing
```

---

# Ownership Chain

Inspect a Pod:

```bash
kubectl describe pod <pod-name>
```

Observe:

```text
Controlled By:
ReplicaSet/...
```

Inspect ReplicaSet:

```bash
kubectl describe rs <replicaset-name>
```

Observe:

```text
Controlled By:
Deployment/...
```

Ownership hierarchy:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

---

# Production Recommendations

Use:

```text
Deployment
```

for:

* Web applications
* APIs
* Frontend services
* Stateless workloads

Avoid using:

```text
ReplicationController
```

for new workloads.

---

# Comparison

| Feature             | ReplicationController | ReplicaSet | Deployment |
| ------------------- | --------------------- | ---------- | ---------- |
| Maintain Replicas   | Yes                   | Yes        | Yes        |
| Advanced Selectors  | No                    | Yes        | Yes        |
| Rolling Updates     | No                    | No         | Yes        |
| Rollback            | No                    | No         | Yes        |
| Production Standard | No                    | Rarely     | Yes        |

---

# Common CKA Commands

```bash
kubectl get deploy
kubectl get rs
kubectl get pods

kubectl describe deployment nginx

kubectl scale deployment nginx --replicas=5

kubectl rollout status deployment nginx

kubectl rollout history deployment nginx

kubectl rollout undo deployment nginx

kubectl set image deployment/nginx nginx=nginx:1.25
```

---

# Key Takeaways

## 1. Controllers Maintain Desired State

```text
Desired State
    vs
Current State
```

Kubernetes continuously reconciles differences.

---

## 2. Deployment Is the Production Standard

Most workloads should be deployed using Deployments.

---

## 3. Deployment Owns ReplicaSets

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

---

## 4. Labels and Selectors Connect Everything

Controllers use labels and selectors to identify resources.

---

## 5. Rolling Updates Create New ReplicaSets

Kubernetes does NOT modify existing Pods.

Instead:

```text
Create New ReplicaSet
    ↓
Create New Pods
    ↓
Remove Old Pods
```

---

## 6. Rollback Is Possible Because Old ReplicaSets Are Preserved

```bash
kubectl rollout undo deployment nginx
```

restores a previous ReplicaSet revision.

---

Understanding this hierarchy is essential for both the CKA exam and real-world Kubernetes operations.

# Why Use Deployments Instead of ReplicaSets?

At first glance, ReplicaSets appear to provide everything needed:

* Maintain desired replica count
* Self-healing
* Scaling

This raises a common question:

```text
Why do we need Deployments if ReplicaSets already manage Pods?
```

The answer is that Deployments are not designed to replace ReplicaSets.

Instead:

```text
Deployment
    =
Application Release Controller

ReplicaSet
    =
Pod Replica Controller
```

A ReplicaSet focuses on maintaining Pod replicas.

A Deployment focuses on managing application releases.

---

## Release Management

ReplicaSets only know the current desired state.

Deployments maintain release history.

Example:

```text
Revision 1
nginx:1.25

Revision 2
nginx:1.26

Revision 3
nginx:1.27
```

View history:

```bash
kubectl rollout history deployment nginx
```

---

## Declarative Application Lifecycle

With a Deployment, you only declare the desired state:

```yaml
replicas: 3
image: nginx:1.27
```

Kubernetes automatically determines:

```text
Current ReplicaSet
Desired ReplicaSet
Required Changes
```

Without Deployments, these operations must be managed manually.

---

## Rollout Management

Deployments provide built-in rollout capabilities:

```bash
kubectl rollout status deployment nginx
```

Monitor:

```text
Updated Pods
Ready Pods
Available Pods
```

ReplicaSets do not provide rollout management.

---

## Pause and Resume Updates

Pause a deployment:

```bash
kubectl rollout pause deployment nginx
```

Apply multiple changes.

Resume deployment:

```bash
kubectl rollout resume deployment nginx
```

This feature is not available in ReplicaSets.

---

## Deployment Strategies

Deployments support update strategies:

```yaml
strategy:
  type: RollingUpdate
```

or:

```yaml
strategy:
  type: Recreate
```

This allows controlled application upgrades.

ReplicaSets do not provide deployment strategies.

---

## Rollbacks

Deployments maintain previous ReplicaSets and can roll back easily:

```bash
kubectl rollout undo deployment nginx
```

Rollback works because Deployments preserve previous ReplicaSet revisions.

ReplicaSets do not provide rollback functionality.

### Good to know

Every time you update a deployment (e.g., by using kubectl set to update the image), Kubernetes creates a new ReplicaSet in the background. The previous ReplicaSet's replica count is scaled down to 0, and the new ReplicaSet is scaled up to the desired number of replicas.

When you perform a rollback, Kubernetes scales down the current ReplicaSet to 0 and scales up the ReplicaSet associated with the revision you're rolling back to, effectively switching the active ReplicaSet to the previous version.

---

## Production Reality

In modern Kubernetes environments:

```text
Deployment
↓
ReplicaSet
↓
Pods
```

is the standard architecture.

Most GitOps tools such as:

* ArgoCD
* Flux
* Helm

primarily manage Deployments rather than ReplicaSets directly.

---

## Ownership Hierarchy

Deployments own ReplicaSets.

ReplicaSets own Pods.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

Each layer has a different responsibility.

---

## DBA Analogy

For database professionals:

```text
Deployment
    =
Application Release Manager

ReplicaSet
    =
Process Manager

Pods
    =
Running Processes
```

ReplicaSets keep processes running.

Deployments manage application versions, upgrades, rollbacks, and release history.

---

## Key Takeaway

Do not think of Deployments as a better ReplicaSet.

Instead:

```text
Deployment
    =
Release Management Layer

ReplicaSet
    =
Replica Management Layer

Pod
    =
Workload Layer
```

This separation of responsibilities is one of the most important Kubernetes architecture concepts to understand for both the CKA exam and real-world production environments.

# Understanding Rollbacks vs last-applied-configuration

One common source of confusion is the difference between:

```text
Deployment Revision History
```

and

```text
kubectl.kubernetes.io/last-applied-configuration
```

These are completely different mechanisms.

---

## Deployment Revision History

Deployment revisions are created and managed by Kubernetes Deployments.

Example:

```text
Revision 1
nginx:1.25

Revision 2
nginx:1.26

Revision 3
nginx:1.27
```

View revisions:

```bash
kubectl rollout history deployment nginx
```

Rollback:

```bash
kubectl rollout undo deployment nginx
```

Deployment rollback works by activating a previous ReplicaSet revision.

---

## last-applied-configuration

When using:

```bash
kubectl apply -f deployment.yaml
```

kubectl stores a copy of the applied manifest inside the object metadata.

Example:

```yaml
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: ...
```

This annotation is used by:

```bash
kubectl apply
```

to calculate future changes and perform three-way merges.

---

## Example Workflow

Initial deployment:

```yaml
image: nginx:1.25
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

State:

```text
Cluster:
  nginx:1.25

last-applied:
  nginx:1.25
```

---

Update deployment:

```yaml
image: nginx:1.26
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

State:

```text
Cluster:
  nginx:1.26

last-applied:
  nginx:1.26
```

---

Rollback deployment:

```bash
kubectl rollout undo deployment nginx
```

State:

```text
Cluster:
  nginx:1.25

last-applied:
  nginx:1.26
```

Notice that:

```text
Deployment was rolled back
Annotation was NOT rolled back
```

This is the reason Kubernetes displays warnings such as:

```text
Rolling back will not update the
kubectl.kubernetes.io/last-applied-configuration annotation
```

---

## Why Does This Matter?

After a rollback:

```text
Actual Cluster State
≠
last-applied-configuration
```

Future:

```bash
kubectl apply -f deployment.yaml
```

operations may behave differently than expected because kubectl still uses the previous applied configuration as its merge baseline.

---

## Production Best Practice

Treat:

```text
Git Repository
```

as the source of truth.

Instead of relying on:

```bash
kubectl rollout undo
```

for permanent recovery, teams usually:

```text
Git Revert
    ↓
Commit
    ↓
CI/CD
    ↓
Cluster Updated
```

This keeps:

```text
Git
Deployment
Cluster State
```

fully synchronized.

---

## Useful Commands

View deployment revisions:

```bash
kubectl rollout history deployment nginx
```

Rollback:

```bash
kubectl rollout undo deployment nginx
```

View last applied configuration:

```bash
kubectl apply view-last-applied deployment/nginx
kubectl apply view-last-applied deployment/nginx-deployment -o yaml > last-applied-deployment.yaml
```

---

## Key Takeaway

```text
Deployment Revision History
    =
Used by rollout undo

last-applied-configuration
    =
Used by kubectl apply

They are completely independent mechanisms.
```


