# Kubernetes Scheduling & Placement Control


> Custom notes for a kubeadm cluster:
>
> * k8s-master1
> * k8s-worker1
> * k8s-worker2
>
> Focused on CKA preparation and real-world production concepts.

---

# 1. Why This Topic Matters

One of the most important responsibilities of the Kubernetes scheduler is deciding:

```text
WHERE should a Pod run?
```

By default, Kubernetes only considers:

* Available CPU
* Available Memory
* Node conditions
* Scheduling constraints

Without additional controls, Pods can run on any suitable node.

Scheduling controls allow us to:

* Reserve nodes for specific workloads
* Separate production and development workloads
* Protect GPU nodes
* Protect database nodes
* Isolate critical services
* Control maintenance operations

---

# 2. Scheduling Control Overview

There are four major concepts:

| Feature       | Configured On | Purpose                 |
| ------------- | ------------- | ----------------------- |
| nodeName      | Pod           | Force a pod to a node   |
| NodeSelector  | Pod           | Simple node selection   |
| Node Affinity | Pod           | Advanced node selection |
| Taint         | Node          | Reject workloads        |
| Toleration    | Pod           | Allow workloads         |

---

# 3. Scheduling Flow

Think of scheduling in this order:

```text
Scheduler
    |
    +--> Is node healthy?
    |
    +--> Does node satisfy nodeSelector?
    |
    +--> Does node satisfy nodeAffinity?
    |
    +--> Does node have taints?
              |
              +--> Does pod tolerate them?
```

Only then:

```text
Pod gets scheduled
```

---

# 4. Manual Scheduling (nodeName)

## Concept

Bypasses the scheduler completely.

Example:

```yaml
spec:
  nodeName: k8s-worker1
```

Kubernetes will directly bind the pod to that node.

---

## Important

Taints are ignored.

Example:

```yaml
spec:
  nodeName: k8s-master1
```

Even if:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

exists, the pod will still run.

---

## When to Use

Rarely used in production.

Mostly:

* Debugging
* Labs
* Static environments

---

# 5. Static Pods

Static Pods are managed directly by kubelet.

Manifest location:

```bash
/etc/kubernetes/manifests
```

Example:

```bash
ls /etc/kubernetes/manifests
```

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

---

## Important

Static Pods:

```text
Do NOT depend on the API Server
```

This is why control plane components survive API Server failures.

---

# 6. Node Labels

Labels are metadata attached to nodes.

Example:

```bash
kubectl label node k8s-worker1 disk=ssd
kubectl label node k8s-worker2 disk=hdd
```

Verify:

```bash
kubectl get nodes --show-labels
```

---

# 7. NodeSelector

## Concept

Simple node matching.

Example:

```yaml
spec:
  nodeSelector:
    disk: ssd
```

Scheduler only considers nodes with:

```text
disk=ssd
```

---

## Advantages

Simple.

Easy to understand.

---

## Limitations

Only supports:

```text
key=value
```

Cannot express:

```text
OR
NOT
Preferred
```

---

# 8. Taints

## Concept

Applied on nodes.

Think:

```text
Node says:
"Do not schedule workloads here"
```

Syntax:

```bash
kubectl taint node NODE KEY=VALUE:EFFECT
```

Example:

```bash
kubectl taint node k8s-worker1 storage=ssd:NoSchedule
```

---

# 9. Taint Effects

## NoSchedule

Blocks new Pods.

Existing Pods remain.

Example:

```text
storage=ssd:NoSchedule
```

---

## PreferNoSchedule

Soft restriction.

Scheduler tries to avoid the node.

Not guaranteed.

---

## NoExecute

Most aggressive.

Existing Pods without matching tolerations are evicted.

Example:

```bash
kubectl taint node k8s-worker1 maintenance=true:NoExecute
```

---

# 10. Tolerations

## Concept

Applied on Pods.

Think:

```text
Pod says:
"I am allowed to enter"
```

Example:

```yaml
tolerations:
- key: "storage"
  operator: "Equal"
  value: "ssd"
  effect: "NoSchedule"
```

---

## Important

Toleration only grants permission.

It does NOT force scheduling.

Example:

```text
worker1
storage=ssd:NoSchedule

worker2
(no taint)
```

Pod with toleration:

```text
Can run on worker1
Can run on worker2
```

Scheduler still chooses.

---

# 11. Exists Operator

Example:

```yaml
tolerations:
- key: storage
  operator: Exists
  effect: NoSchedule
```

Matches:

```text
storage=ssd
storage=hdd
storage=nvme
```

---

Example:

```yaml
tolerations:
- operator: Exists
```

Matches:

```text
ALL TAINTS
```

This is commonly used by:

* kube-proxy
* Calico
* Cilium
* Node Exporter

---

# 12. Node Affinity

NodeSelector 2.0

More flexible and production-ready.

---

## Required Affinity

Hard requirement.

Example:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disk
          operator: In
          values:
          - ssd
```

Equivalent to:

```text
Must run on SSD node
```

---

## Preferred Affinity

Soft requirement.

Example:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: disk
          operator: In
          values:
          - ssd
```

Meaning:

```text
Prefer SSD node
```

but not mandatory.

---

# 13. Mental Model

## Taints & Tolerations

```text
Node says:
"Keep out"

Pod says:
"I can enter"
```

---

## Node Affinity

```text
Pod says:
"I want to run here"
```

---

## Combined

```text
Toleration
=
Permission

Affinity
=
Preference / Requirement
```

---

# 14. Production Pattern

GPU Example.

Node:

```bash
kubectl label node k8s-worker1 accelerator=nvidia

kubectl taint node k8s-worker1 gpu=true:NoSchedule
```

Pod:

```yaml
tolerations:
- key: gpu
  value: "true"

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
```

Result:

```text
Allowed on GPU node
AND
Must use GPU node
```

---

# 15. Production Pattern (Database)

Node:

```bash
kubectl taint node k8s-worker2 db=true:NoSchedule

kubectl label node k8s-worker2 workload=database
```

Database Pod:

```yaml
tolerations:
- key: db
  value: "true"

affinity:
  nodeAffinity:
```

Result:

```text
Database workloads stay isolated
```

---

# 16. Lab 1 - Control Plane Taint

Check default taint:

```bash
kubectl describe node k8s-master1 | grep Taints
```

Expected:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Observe system pods:

```bash
kubectl get pods -n kube-system -o wide
```

Check tolerations:

```bash
kubectl describe pod <pod-name> -n kube-system
```

---

# 17. Lab 2 - NodeSelector

Label nodes:

```bash
kubectl label node k8s-worker1 disk=ssd
kubectl label node k8s-worker2 disk=hdd
```

Create a pod with:

```yaml
nodeSelector:
  disk: ssd
```

Verify:

```bash
kubectl get pod -o wide
```

---

# 18. Lab 3 - Taints & Tolerations

Apply taint:

```bash
kubectl taint node k8s-worker1 storage=ssd:NoSchedule
```

Create nginx pod without toleration.

Observe:

```bash
kubectl get pod -o wide
```

Pod should avoid worker1.

---

Add toleration.

Observe:

```bash
kubectl get pod -o wide
```

Now worker1 becomes eligible.

---

# 19. Lab 4 - Node Affinity Deep Dive

## Objective

Learn how Kubernetes uses **Node Affinity** to control where Pods are scheduled.

This lab demonstrates:

* Required Node Affinity
* Preferred Node Affinity
* Match Expressions
* In / NotIn operators
* Real-world production patterns
* Combining Taints, Tolerations, and Node Affinity

---

## Prerequisites

Cluster:

```text
k8s-master1
k8s-worker1
k8s-worker2
```

Verify nodes:

```bash
kubectl get nodes
```

---

## Step 1 - Add Node Labels

Assign labels to worker nodes:

```bash
kubectl label node k8s-worker1 zone=primary
kubectl label node k8s-worker2 zone=secondary
```

Verify:

```bash
kubectl get nodes --show-labels | grep zone
```

Expected output:

```text
k8s-worker1   zone=primary
k8s-worker2   zone=secondary
```

---

## Lab 4.1 - Required Node Affinity

## Goal

Force Pods to run only on nodes labeled:

```text
zone=primary
```

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-required
spec:
  replicas: 3

  selector:
    matchLabels:
      app: affinity-required

  template:
    metadata:
      labels:
        app: affinity-required

    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: zone
                operator: In
                values:
                - primary

      containers:
      - name: nginx
        image: nginx
```

Apply:

```bash
kubectl apply -f affinity-required.yaml
```

Verify:

```bash
kubectl get pods -o wide
```

Expected:

```text
affinity-required-xxx   k8s-worker1
affinity-required-yyy   k8s-worker1
affinity-required-zzz   k8s-worker1
```

---

## Explanation

`requiredDuringSchedulingIgnoredDuringExecution`

means:

```text
This requirement MUST be satisfied
before scheduling can occur.
```

If no matching node exists:

```text
Pod remains Pending
```

---

## Lab 4.2 - Affinity Failure Scenario

## Remove the Label

```bash
kubectl label node k8s-worker1 zone-
```

Verify:

```bash
kubectl get nodes --show-labels
```

No node should have:

```text
zone=primary
```

---

## Recreate Deployment

```bash
kubectl delete deployment affinity-required

kubectl apply -f affinity-required.yaml
```

---

## Check Status

```bash
kubectl get pods
```

Expected:

```text
Pending
Pending
Pending
```

---

## Investigate

```bash
kubectl describe pod <pod-name>
```

Expected scheduler event:

```text
0/3 nodes are available:
node(s) didn't match Pod's node affinity
```

---

## Key Learning

Required affinity behaves like:

```text
Hard Requirement
```

No matching node:

```text
No scheduling
```

---

# Restore Label

```bash
kubectl label node k8s-worker1 zone=primary
```

---

## Lab 4.3 - Preferred Node Affinity

## Goal

Prefer worker1, but allow worker2 if needed.

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-preferred

spec:
  replicas: 5

  selector:
    matchLabels:
      app: affinity-preferred

  template:
    metadata:
      labels:
        app: affinity-preferred

    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - primary

      containers:
      - name: nginx
        image: nginx
```

Apply:

```bash
kubectl apply -f affinity-preferred.yaml
```

Verify:

```bash
kubectl get pods -o wide
```

---

## Expected Behavior

Most Pods should land on:

```text
k8s-worker1
```

However:

```text
k8s-worker2
```

remains a valid option.

---

## Key Learning

Preferred affinity means:

```text
Try this node first
```

NOT:

```text
Must use this node
```

---

## Lab 4.4 - Multiple Values (In Operator)

## Add Labels

```bash
kubectl label node k8s-worker1 disk=ssd
kubectl label node k8s-worker2 disk=hdd
```

Verify:

```bash
kubectl get nodes --show-labels | grep disk
```

---

## Affinity Example

```yaml
matchExpressions:
- key: disk
  operator: In
  values:
  - ssd
  - nvme
```

---

## Logic

Equivalent to:

```text
disk=ssd OR disk=nvme
```

---

## Result

```text
worker1 (ssd)
    -> Match

worker2 (hdd)
    -> No Match
```

---

## Lab 4.5 - NotIn Operator

## Example

```yaml
matchExpressions:
- key: disk
  operator: NotIn
  values:
  - hdd
```

---

## Logic

Equivalent to:

```text
Any node except HDD nodes
```

---

## Result

```text
worker1 (ssd)
    -> Match

worker2 (hdd)
    -> Reject
```

---

## Lab 4.6 - Real Production Pattern

## Goal

Reserve a node for database workloads only.

---

## Configure Node

### Label

```bash
kubectl label node k8s-worker1 workload=database
```

### Taint

```bash
kubectl taint node k8s-worker1 db=true:NoSchedule
```

Verify:

```bash
kubectl describe node k8s-worker1 | grep Taints
```

Expected:

```text
db=true:NoSchedule
```

---

## Database Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-demo

spec:
  replicas: 1

  selector:
    matchLabels:
      app: postgres-demo

  template:
    metadata:
      labels:
        app: postgres-demo

    spec:

      tolerations:
      - key: "db"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - database

      containers:
      - name: postgres
        image: postgres:16
```

Apply:

```bash
kubectl apply -f postgres-demo.yaml
```

---

## Verify Placement

```bash
kubectl get pods -o wide
```

Expected:

```text
postgres-demo-xxx   k8s-worker1
```

---

## Why This Works

## Taint

```text
db=true:NoSchedule
```

Node says:

```text
Keep out
```

---

## Toleration

```yaml
tolerations:
```

Pod says:

```text
I am allowed to enter
```

---

## Node Affinity

```yaml
requiredDuringSchedulingIgnoredDuringExecution
```

Pod says:

```text
I must use a database node
```

---

## Production Mental Model

```text
Taint
    = Security Gate

Toleration
    = Access Pass

Node Affinity
    = Routing Policy
```

---

# 20. CKA Exam Cheatsheet

View node labels:

```bash
kubectl get nodes --show-labels
```

---

Label a node:

```bash
kubectl label node NODE key=value
```

---

Add taint:

```bash
kubectl taint node NODE key=value:NoSchedule
```

---

Remove taint:

```bash
kubectl taint node NODE key=value:NoSchedule-
```

---

Describe scheduling failures:

```bash
kubectl describe pod POD
```

---

Check node assignment:

```bash
kubectl get pods -o wide
```

---

# Final Key Takeaway

Remember this hierarchy:

```text
nodeName
    = Force scheduling

NodeSelector
    = Simple node selection

NodeAffinity
    = Advanced node selection

Taint
    = Node rejects workloads

Toleration
    = Pod receives permission
```

Production environments typically combine:

```text
Labels
+
Node Affinity
+
Taints
+
Tolerations
```

to achieve predictable and controlled workload placement.
