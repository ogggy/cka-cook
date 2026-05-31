# YAML Fundamentals and Pod Manifests

## Objective

The goal of this lab is to understand:

* Imperative vs Declarative Kubernetes management
* Basic YAML structure
* Pod manifests
* Generating Kubernetes YAML from kubectl
* Essential pod lifecycle commands

This knowledge forms the foundation for all Kubernetes resources that will be used later in Deployments, Services, Storage, RBAC, and Troubleshooting.

---

# Imperative vs Declarative

## Imperative Approach

Imperative commands tell Kubernetes exactly what action to perform.

Example:

```bash
kubectl run nginx --image=nginx
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
```

### Advantages

* Fast
* Simple
* Useful for troubleshooting
* Useful during CKA exam

### Disadvantages

* Hard to version control
* Difficult to reproduce consistently
* Not suitable for production environments

---

## Declarative Approach

Declarative configuration describes the desired state of the system.

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pod.yaml
```

### Advantages

* Version controlled
* Reproducible
* Easy to review
* Production standard

### Disadvantages

* Slightly slower to create initially
* Requires YAML knowledge

---

# Key Takeaway

```text
Imperative = Tell Kubernetes what to do.

Declarative = Describe the desired state.
```

Production environments primarily use the declarative model.

---

# YAML Fundamentals

Kubernetes resources are defined using YAML files.

Most manifests contain four major sections:

```yaml
apiVersion:
kind:
metadata:
spec:
```

---

## apiVersion

Defines which Kubernetes API version manages the resource.

Example:

```yaml
apiVersion: v1
```

---

## kind

Defines the resource type.

Examples:

```yaml
kind: Pod
kind: Deployment
kind: Service
kind: ConfigMap
```

---

## metadata

Defines object identity and labels.

Example:

```yaml
metadata:
  name: nginx
```

---

## spec

Defines the desired state.

Example:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
```

---

# Key Takeaway

```text
metadata = Who am I?

spec = What should I look like?
```

---

# Pod Manifest Basics

A Pod is the smallest deployable unit in Kubernetes.

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
  - name: nginx
    image: nginx
```

Create:

```bash
kubectl apply -f pod.yaml
```

Verify:

```bash
kubectl get pods
```

---

# Understanding the Pod Structure

```text
Pod
└── Container
    └── nginx
```

A Pod can contain:

```text
1 container
or
multiple containers
```

Although possible, most applications use:

```text
1 Pod
└── 1 main container
```

---

# Generate YAML Using kubectl

Instead of writing YAML from memory:

```bash
kubectl run nginx \
  --image=nginx \
  --dry-run=client \
  -o yaml
```

Generate and save:

```bash
kubectl run nginx \
  --image=nginx \
  --dry-run=client \
  -o yaml > nginx.yaml
```

Edit:

```bash
vim nginx.yaml
```

Apply:

```bash
kubectl apply -f nginx.yaml
```

---

# Key Takeaway

During the CKA exam:

```text
Generate → Edit → Apply
```

is usually faster and safer than writing manifests from scratch.

---

# Essential Pod Commands

## Create a Pod

```bash
kubectl run nginx --image=nginx
```

---

## List Pods

```bash
kubectl get pods
kubectl get pods -o wide
```

---

## Describe a Pod

```bash
kubectl describe pod nginx
```

Useful for:

* Events
* Scheduling issues
* Image pull failures
* Resource information

---

## View Logs

```bash
kubectl logs nginx
```

---

## Execute Commands Inside a Pod

```bash
kubectl exec -it nginx -- sh
```

For Ubuntu-based images:

```bash
kubectl exec -it ubuntu-pod -- bash
```

---

## Delete a Pod

```bash
kubectl delete pod nginx
```

---

# Troubleshooting Workflow

When a Pod is not working:

## Step 1

Check status:

```bash
kubectl get pods
```

---

## Step 2

Inspect details:

```bash
kubectl describe pod <pod-name>
```

---

## Step 3

Check logs:

```bash
kubectl logs <pod-name>
```

---

## Step 4

Access the container:

```bash
kubectl exec -it <pod-name> -- sh
```

---

# Homelab Exercises

## Exercise 1 - Create a Pod

```bash
kubectl run nginx --image=nginx
```

Verify:

```bash
kubectl get pods -o wide
```

---

## Exercise 2 - Inspect the Pod

```bash
kubectl describe pod nginx
```

Observe:

* Node assignment
* Events
* Pod IP
* Container details

---

## Exercise 3 - Access the Container

```bash
kubectl exec -it nginx -- sh
```

Inside:

```bash
hostname
ip addr
```

Exit:

```bash
exit
```

---

## Exercise 4 - Generate YAML

```bash
kubectl run nginx \
  --image=nginx \
  --dry-run=client \
  -o yaml > nginx.yaml
```

Review:

```bash
cat nginx.yaml
```

---

## Exercise 5 - Recreate Using YAML

Delete:

```bash
kubectl delete pod nginx
```

Create:

```bash
kubectl apply -f nginx.yaml
```

---

# Key Takeaways

## Core Concepts

```text
Imperative = Quick actions

Declarative = Desired state
```

---

## YAML Structure

```text
apiVersion
kind
metadata
spec
```

---

## Pod Fundamentals

```text
Pod = Smallest deployable unit in Kubernetes
```

---

## CKA Exam Tip

```text
Generate YAML whenever possible.

kubectl ... --dry-run=client -o yaml
```

---

## Troubleshooting Commands

```bash
kubectl get pods
kubectl describe pod
kubectl logs <pod-name> -c <container-name> -f 
kubectl exec -it <pod-name> -c <container-name> -- sh 
```

These commands are among the most frequently used commands in the CKA exam and in real-world Kubernetes operations.
