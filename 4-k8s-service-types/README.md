# Kubernetes Services

## Objective

Understand Kubernetes Service types:

* ClusterIP
* NodePort
* LoadBalancer
* ExternalName

This lab is adapted for a kubeadm cluster:

| Node        | Role          | IP          |
| ----------- | ------------- | ----------- |
| k8s-master1 | Control Plane | 10.67.10.10 |
| k8s-worker1 | Worker        | 10.67.10.11 |
| k8s-worker2 | Worker        | 10.67.10.12 |

---

# Why Do We Need Services?

Pods are ephemeral.

A Pod can be:

* Created
* Deleted
* Restarted
* Rescheduled to another node

Every time this happens, the Pod IP may change.

Without Services:

```text
Frontend Pod -> Backend Pod IP
```

This is not reliable because Pod IPs are dynamic.

With Services:

```text
Frontend Pod -> backend-svc -> Backend Pods
```

A Service provides:

* Stable ClusterIP
* Stable DNS name
* Load balancing across matching Pods
* Decoupling between clients and Pod IPs

---

# Service Uses Labels and Selectors

A Service does not select Pods by name.

It selects Pods by labels.

Example:

```yaml
selector:
  app: backend
```

This means:

```text
Send traffic to all Pods where app=backend
```

Check endpoints:

```bash
kubectl get endpoints
kubectl get endpointslice
```

---

# ClusterIP Service

ClusterIP is the default Service type.

It is used for internal communication inside the cluster.

Example flow:

```text
frontend Pod
    ↓
backend-svc ClusterIP
    ↓
backend Pod
```

## Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend-container
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Backend"
        ports:
        - containerPort: 5678
```

## Backend ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 5678
```

Apply:

```bash
kubectl apply -f backend-deploy.yaml
kubectl apply -f backend-svc.yaml
```

Test from inside the cluster:

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  -it --rm \
  --restart=Never \
  -- curl http://backend-svc:9090
```

Expected:

```text
Hello from Backend
```

---

# NodePort Service

NodePort exposes a Service externally using:

```text
<NodeIP>:<NodePort>
```

NodePort range is usually:

```text
30000-32767
```

In this kubeadm lab, use the real node IPs:

```text
10.67.10.10
10.67.10.11
10.67.10.12
```

## Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend-container
        image: nginx
        ports:
        - containerPort: 80
```

## Frontend NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 31000
```

Apply:

```bash
kubectl apply -f frontend-deploy.yaml
kubectl apply -f frontend-svc.yaml
```

Test from WSL jumpbox:

```bash
curl http://10.67.10.10:31000
curl http://10.67.10.11:31000
curl http://10.67.10.12:31000
```

Important:

```text
NodePort is available on every node,
even if the actual Pod is running on another node.
```

Request flow:

```text
Client
  ↓
NodeIP:NodePort
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
```

---

# LoadBalancer Service

LoadBalancer provides a single external IP.

In cloud environments, Kubernetes asks the cloud provider to create a load balancer.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

In a bare-metal kubeadm homelab, LoadBalancer usually stays as:

```text
EXTERNAL-IP: <pending>
```

because there is no cloud provider load balancer.

Check:

```bash
kubectl get svc
```

For homelab, use one of these later:

* MetalLB
* Ingress Controller
* NodePort
* Tailscale
* Reverse proxy

Request flow in cloud:

```text
Client
  ↓
Cloud LoadBalancer External IP
  ↓
NodePort
  ↓
ClusterIP
  ↓
Pod
```

---

# ExternalName Service

ExternalName maps an internal Kubernetes Service name to an external DNS name.

It does not create:

* ClusterIP
* Endpoints
* Proxy rules

It returns a DNS CNAME.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: example-db.company.com
```

Application can connect to:

```text
external-db.default.svc.cluster.local
```

Instead of hardcoding:

```text
example-db.company.com
```

Use cases:

* External databases
* Third-party APIs
* Legacy systems
* External services outside Kubernetes

---

# Important Built-in Service: kubernetes

Every cluster has a default Service:

```bash
kubectl get svc kubernetes
```

Example:

```text
NAME         TYPE        CLUSTER-IP
kubernetes   ClusterIP   10.96.0.1
```

This Service points to the Kubernetes API Server.

Check:

```bash
kubectl get endpoints kubernetes
```

Example:

```text
10.67.10.10:6443
```

Pods inside the cluster can reach the API Server using:

```text
https://kubernetes.default.svc
```

---

# Port Definitions

For a Service:

```yaml
ports:
- port: 80
  targetPort: 8080
  nodePort: 31000
```

Meaning:

```text
port:
  Service port

targetPort:
  Container port inside the Pod

nodePort:
  Port exposed on each node
```

Example:

```text
Client -> NodeIP:31000 -> Service:80 -> Pod:8080
```

---

# Debug Commands

Check Services:

```bash
kubectl get svc
```

Check Service details:

```bash
kubectl describe svc frontend-svc
```

Check Endpoints:

```bash
kubectl get endpoints frontend-svc
```

Check EndpointSlices:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=frontend-svc
```

Check Pods and labels:

```bash
kubectl get pods --show-labels
```

Check if Service selector matches Pods:

```bash
kubectl get pods -l app=frontend
```

DNS test:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  -it --rm \
  --restart=Never \
  -- nslookup backend-svc
```

---

# Common Issues

## Service has no endpoints

Check:

```bash
kubectl get endpoints <service-name>
```

If empty, likely causes:

```text
Service selector does not match Pod labels.
Pods are not Ready.
Pods are in another namespace.
```

## NodePort is not reachable

Check:

```bash
kubectl get svc
kubectl get nodes -o wide
```

Then test:

```bash
curl http://<NodeIP>:<NodePort>
```

Also check firewall/security rules if needed.

## LoadBalancer stays pending

This is normal on kubeadm bare-metal unless a load balancer implementation such as MetalLB is installed.

---

# Service Type Summary

| Type         | Scope                    | Use Case                                  |
| ------------ | ------------------------ | ----------------------------------------- |
| ClusterIP    | Internal cluster only    | Pod-to-Pod / app-to-app communication     |
| NodePort     | External via NodeIP:Port | Simple lab/testing external access        |
| LoadBalancer | External single IP       | Cloud production exposure                 |
| ExternalName | DNS alias                | Access external services by internal name |

---

# Key Takeaways

```text
Services solve the dynamic Pod IP problem.
```

```text
Services select Pods using labels and selectors.
```

```text
ClusterIP is for internal communication.
```

```text
NodePort exposes the app on every node IP.
```

```text
LoadBalancer requires cloud provider integration or bare-metal replacement.
```

```text
ExternalName is DNS aliasing, not load balancing.
```

```text
EndpointSlice stores the actual backend Pod IPs behind a Service.
```

---

# Cleanup

```bash
kubectl delete svc backend-svc frontend-svc frontend-lb external-db --ignore-not-found
kubectl delete deploy backend-deploy frontend-deploy --ignore-not-found
kubectl delete pod curl-test dns-test --ignore-not-found
```
