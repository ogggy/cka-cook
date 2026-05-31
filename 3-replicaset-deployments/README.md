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
