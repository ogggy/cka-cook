# Kubernetes Contexts and Kubeconfig

## Objective

The goal of this lab is to understand how Kubernetes uses **clusters**, **users**, **contexts**, and **namespaces** through the kubeconfig file.

Unlike the original lab which creates multiple clusters using Kind, this lab uses an existing kubeadm cluster and focuses on context management.

---

## Environment

### Cluster

| Node        | Role          | IP          |
| ----------- | ------------- | ----------- |
| k8s-master1 | Control Plane | 10.67.10.10 |
| k8s-worker1 | Worker        | 10.67.10.11 |
| k8s-worker2 | Worker        | 10.67.10.12 |

### Access Method

* Kubernetes cluster created with kubeadm
* kubectl executed from WSL jumpbox
* kubeconfig copied from the control plane node

---

# Kubernetes Configuration Components

In the kubeconfig file, each cluster, user, and namespace combination is stored as a context. The contexts will contain:

- Cluster details: API server URL, certificate authority, etc.
- User details: Authentication method (e.g., username/password, token, certificate).
- Namespace details: The default namespace to work in for the context (though the namespace can be overridden on a per-command basis).

A **Kubernetes namespace** is a virtual cluster within a physical cluster that provides logical segregation for resources, enabling multiple environments (e.g., dev, staging, prod) to coexist on the same cluster.

For example, in a cluster running multiple applications, each application can run in its own namespace, ensuring isolation and avoiding conflicts between resources like services or pods.


A kubeconfig file contains three major objects:

## Cluster

Defines the Kubernetes API endpoint.

Example:

```yaml
cluster:
  server: https://10.67.10.10:6443
```

---

## User

Defines the authentication method used to access the cluster.

Example:

```yaml
user:
  client-certificate-data: ...
  client-key-data: ...
```

---

## Context

A context combines:

```text
Cluster + User + Namespace
```

Example:

```yaml
context:
  cluster: kubernetes
  user: kubernetes-admin
  namespace: dev
```

---

# Inspect Existing Configuration

View all contexts:

```bash
kubectl config get-contexts
```

View the active context:

```bash
kubectl config current-context
```

View the complete configuration:

```bash
kubectl config view
```

View only the active context:

```bash
kubectl config view --minify
```

---

# Create Namespaces

Create namespaces to simulate multiple environments.

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

Verify:

```bash
kubectl get namespaces
```

---

# Create Additional Contexts

Create a context for the development environment:

```bash
kubectl config set-context admin-dev \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --namespace=dev
```

Create a context for staging:

```bash
kubectl config set-context admin-staging \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --namespace=staging
```

Create a context for production:

```bash
kubectl config set-context admin-prod \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --namespace=prod
```

List contexts:

```bash
kubectl config get-contexts
```

---

# Switch Contexts

Switch to development:

```bash
kubectl config use-context admin-dev
```

Verify:

```bash
kubectl config current-context
```

---

# Deploy Resources in Different Contexts

Create a test pod inside the dev namespace:

```bash
kubectl run nginx \
  --image=nginx
```

Verify:

```bash
kubectl get pods
```

Check namespaces:

```bash
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n prod
```

The pod should only exist in the dev namespace.

---

# Switch to Production Context

```bash
kubectl config use-context admin-prod
```

Verify:

```bash
kubectl get pods
```

No pods should be displayed because the current namespace is prod.

---

# Key Takeaways

## One Cluster Can Have Multiple Contexts

You do not need multiple Kubernetes clusters to learn contexts.

A single cluster can have:

* Multiple users
* Multiple namespaces
* Multiple contexts

---

## Context Is a Convenience Layer

A context simply tells kubectl:

```text
Which cluster?
Which user?
Which namespace?
```

---

## Context Helps Prevent Mistakes

Before running critical commands:

```bash
kubectl config current-context
```

Always verify that you are operating against the correct cluster and namespace.

This becomes extremely important in environments with:

* Development clusters
* Staging clusters
* Production clusters

---

# Cleanup

Delete test resources:

```bash
kubectl delete pod nginx -n dev
```

Restore default namespace:

```bash
kubectl config set-context --current --namespace=default
```

Delete lab namespaces:

```bash
kubectl delete namespace dev
kubectl delete namespace staging
kubectl delete namespace prod
```

---

# CKA Exam Notes

Useful commands to remember:

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>
kubectl config view --minify
kubectl config set-context
```

Understanding kubeconfig and context management is a common operational skill and frequently appears in CKA-style troubleshooting scenarios.
