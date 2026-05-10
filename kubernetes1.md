
> **#90DaysOfCloudDevOps** | Hands-on Kubernetes with KIND Cluster

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![KIND](https://img.shields.io/badge/KIND-Cluster-orange?style=for-the-badge)
![DevOps](https://img.shields.io/badge/DevOps-Practical-green?style=for-the-badge)

---

## 📌 Table of Contents

- [What is Kubernetes](#-what-is-kubernetes)
- [Core Architecture](#-core-architecture)
- [What I Built Today](#-what-i-built-today)
- [Practical 1 – Namespace](#-practical-1--creating-a-namespace)
- [Practical 2 – Pod](#-practical-2--creating-a-pod)
- [Practical 3 – Deployment](#-practical-3--creating-a-deployment)
- [Practical 4 – Self Healing](#-practical-4--self-healing-proof)
- [Practical 5 – Round Robin](#-practical-5--round-robin-load-balancing)
- [Practical 6 – Scaling](#-practical-6--scaling-updown)
- [Practical 7 – Service & Access](#-practical-7--service--accessing-the-app)
- [Key Concepts](#-key-concepts)
- [Interview Questions](#-interview-questions)
- [Cheatsheet](#-kubectl-cheatsheet)

---

## 🤔 What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform** that:

```
✅ Runs your containers automatically
✅ Heals them when they crash
✅ Scales them when traffic increases
✅ Distributes traffic across multiple copies
✅ Updates them with zero downtime
```

> **Simple Definition:**
> Docker runs ONE container.
> Kubernetes manages THOUSANDS of containers across MULTIPLE machines.

---

## 🏗️ Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  Control Plane                        │  │
│  │                                                       │  │
│  │  ┌─────────────────┐    ┌──────────────────────┐    │  │
│  │  │  kube-apiserver  │    │         etcd          │    │  │
│  │  │  (The Brain)     │    │  (The Database)       │    │  │
│  │  └─────────────────┘    └──────────────────────┘    │  │
│  │                                                       │  │
│  │  ┌─────────────────┐    ┌──────────────────────┐    │  │
│  │  │ kube-controller  │    │    kube-scheduler     │    │  │
│  │  │ (Self Healing)   │    │  (Pod Placement)      │    │  │
│  │  └─────────────────┘    └──────────────────────┘    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Worker Node │  │ Worker Node │  │ Worker Node │        │
│  │             │  │             │  │             │        │
│  │  kubelet    │  │  kubelet    │  │  kubelet    │        │
│  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │        │
│  │  │  Pod  │  │  │  │  Pod  │  │  │  │  Pod  │  │        │
│  │  └───────┘  │  │  └───────┘  │  │  └───────┘  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role | What happens if it dies |
|---|---|---|
| **kube-apiserver** | Entry point for all commands | Nothing works. All kubectl commands fail |
| **etcd** | Stores ALL cluster state | Cluster loses memory. Critical to backup |
| **kube-controller** | Watches and self-heals | Pods won't restart if they crash |
| **kube-scheduler** | Decides which node gets the pod | New pods won't be placed |

### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Manages pods on that specific node |
| **containerd** | Actually runs the containers |
| **kube-proxy** | Handles networking and routing |

---

## 🛠️ What I Built Today

```
KIND Cluster (Kubernetes IN Docker)
│
├── tech-control-plane    → Manages everything
├── tech-worker           → Runs app pods
├── tech-worker2          → Runs app pods
└── tech-worker3          → Runs app pods
    │
    └── nginx-ns (Namespace)
        │
        ├── nginx-pod              → Manual pod
        ├── static-app-deployment  → 3 replicas
        └── static-app-service     → NodePort LB
```

### Cluster Setup (cluster.yml)

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2

  - role: worker
    image: kindest/node:v1.31.2
```

```bash
# Create the cluster
kind create cluster --config cluster.yml --name tech

# Verify nodes
kubectl get nodes
```

**Output:**
```
NAME                 STATUS   ROLES           AGE     VERSION
tech-control-plane   Ready    control-plane   3h32m   v1.31.2
tech-worker          Ready    <none>          3h32m   v1.31.2
tech-worker2         Ready    <none>          3h32m   v1.31.2
tech-worker3         Ready    <none>          3h32m   v1.31.2
```

> 📸 **Screenshot:**
>
> `[images/cluster-nodes.png]`

---

## 📁 Practical 1 – Creating a Namespace

### What is a Namespace?

```
Without Namespace:              With Namespace:
┌────────────────────┐         ┌──────────────────────────┐
│      default       │         │  nginx-ns  │  payment-ns  │
│                    │         │  ────────  │  ──────────  │
│  pod-1             │         │  nginx-pod │  payment-pod │
│  pod-2             │         │  static-   │  payment-db  │
│  pod-3             │         │  app       │              │
│  Everything mixed  │         │            │              │
│  Hard to manage ❌ │         │  Isolated  │  Isolated    │
└────────────────────┘         │     ✅     │     ✅       │
                               └──────────────────────────┘
```

> **Simple Rule:** Namespace = Folder for your Kubernetes resources

### namespace.yml

```yaml
kind: Namespace
apiVersion: v1

metadata:
  name: nginx-ns
```

### Commands

```bash
# Create namespace
kubectl apply -f namespace.yml

# Verify
kubectl get namespaces
# or shorthand
kubectl get ns
```

**Output:**
```
NAME                 STATUS   AGE
default              Active   43m
kube-node-lease      Active   43m
kube-public          Active   43m
kube-system          Active   43m
local-path-storage   Active   43m
nginx-ns             Active   1s   ← Our namespace created ✅
```

### ⚠️ Mistake I Made

```bash
# First attempt - forgot the name field
kubectl apply -f namespace.yml

# Error
error: resource name may not be empty
# Reason: metadata.name was missing in the YAML

# Fix: Added name: nginx-ns under metadata
# Second attempt worked ✅
```

> 📸 **Screenshot:**
>
> `[images/namespace-created.png]`

---

## 🐳 Practical 2 – Creating a Pod

### What is a Pod?

```
Container  ──────►  Pod  ──────►  Node
(Docker)           (K8s          (Machine)
                   wrapper)

Pod = Smallest deployable unit in Kubernetes
    = One or more containers running together
    = Gets its own IP address
```

### Pod vs Deployment

| | Pod | Deployment |
|---|---|---|
| **Self Healing** | ❌ No | ✅ Yes |
| **Scaling** | ❌ No | ✅ Yes |
| **Rolling Update** | ❌ No | ✅ Yes |
| **Use Case** | Testing only | Production always |

### pod.yml

```yaml
kind: Pod
apiVersion: v1

metadata:
  name: nginx-pod
  namespace: nginx-ns

spec:
  containers:
    - name: app
      image: codinggaurav/static_world-app:latest
      ports:
        - containerPort: 80
```

### Commands

```bash
# Create the pod
kubectl apply -f pod.yml

# Check pod status
kubectl get pods -n nginx-ns

# Get detailed info
kubectl describe pod nginx-pod -n nginx-ns

# Go inside the pod
kubectl exec -it nginx-pod -n nginx-ns -- bash

# Check pod logs
kubectl logs nginx-pod -n nginx-ns
```

**Output:**
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5s ✅
```

> 📸 **Screenshot:**
>
> `[images/pod-running.png]`

---

## 🚀 Practical 3 – Creating a Deployment

### What is a Deployment?

```
Deployment
│
├── Manages ReplicaSet
│   │
│   ├── Pod 1  ← Running your app
│   ├── Pod 2  ← Running your app
│   └── Pod 3  ← Running your app
│
├── Self Heals if pod dies ✅
├── Scales up/down on command ✅
└── Updates with zero downtime ✅
```

### RollingUpdate vs Recreate

```
RollingUpdate (what we used):      Recreate:
─────────────────────────          ──────────────
Pod1 [v1] → Pod1 [v2]             Kill ALL pods
Pod2 [v1] → Pod2 [v2]                  ↓
Pod3 [v1] → Pod3 [v2]             Start ALL pods
                                        ↓
App always available ✅            Downtime ❌
```

### deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: static-app-deployment
  namespace: nginx-ns
  labels:
    app: static-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: static-app

  strategy:
    type: RollingUpdate

  template:
    metadata:
      labels:
        app: static-app

    spec:
      containers:
        - name: static-app-container
          image: codinggaurav/static_world-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

### Commands

```bash
# Create deployment
kubectl apply -f deployment.yml

# Check deployment
kubectl get deployment -n nginx-ns

# Check pods created by deployment
kubectl get pods -n nginx-ns

# Check everything in namespace
kubectl get all -n nginx-ns
```

**Output:**
```
NAME                                      READY  STATUS   RESTARTS  AGE
pod/nginx-pod                             1/1    Running  0         139m
pod/static-app-deployment-54bf7f7975-d9qbp 1/1   Running  0         4m26s
pod/static-app-deployment-54bf7f7975-pnb4c 1/1   Running  0         69s
pod/static-app-deployment-54bf7f7975-xdx6m 1/1   Running  0         25m
```

> 📸 **Screenshot:**
>
> `[images/deployment-created.png]`

---

## 🔥 Practical 4 – Self Healing Proof

### How Self Healing Works

```
kube-controller-manager runs a loop FOREVER:

    Check Desired State (3 pods)
           │
           ▼
    Check Actual State (? pods)
           │
           ▼
    Are they equal?
    │              │
   YES             NO
    │              │
    │         Create/Delete pods
    │         to match desired state
    │              │
    └──────────────┘
    Loop again in milliseconds
```

### The Experiment

```bash
# Step 1: Check running pods
kubectl get pods -n nginx-ns

# Output
NAME                                        READY  STATUS   AGE
static-app-deployment-54bf7f7975-d9qbp      1/1    Running  4m26s
static-app-deployment-54bf7f7975-pnb4c      1/1    Running  69s    ← DELETE THIS
static-app-deployment-54bf7f7975-xdx6m      1/1    Running  25m

# Step 2: Delete one pod manually
kubectl delete pod static-app-deployment-54bf7f7975-pnb4c -n nginx-ns

# Output
pod "static-app-deployment-54bf7f7975-pnb4c" deleted

# Step 3: Check pods immediately
kubectl get pods -n nginx-ns

# Output
NAME                                        READY  STATUS   AGE
static-app-deployment-54bf7f7975-d9qbp      1/1    Running  4m52s
static-app-deployment-54bf7f7975-xdx6m      1/1    Running  25m
static-app-deployment-54bf7f7975-zmrmv      1/1    Running  4s  ← NEW POD ✅
```

**What happened:**
```
1. We deleted pnb4c
2. kube-controller noticed: Desired=3, Actual=2
3. Created zmrmv automatically
4. Total time: under 5 seconds
5. App never went down ✅
```

> 📸 **Screenshot:**
>
> `[images/self-healing.png]`

---

## ⚡ Practical 5 – Round Robin Load Balancing

### How Round Robin Works in Kubernetes

```
                    ┌─────────────┐
  User Request ────►│   Service   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         ┌─────────┐  ┌─────────┐  ┌─────────┐
         │  Pod 1  │  │  Pod 2  │  │  Pod 3  │
         │ (d9qbp) │  │ (xdx6m) │  │ (bw8qb) │
         └─────────┘  └─────────┘  └─────────┘

Request 1  ──► Pod 1
Request 2  ──► Pod 2
Request 3  ──► Pod 3
Request 4  ──► Pod 1  ← cycle repeats automatically
```

### service.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: static-app-service
  namespace: nginx-ns

spec:
  selector:
    app: static-app      # Connects to all pods with this label

  ports:
    - protocol: TCP
      port: 80           # Service listens on 80
      targetPort: 80     # Forwards to pod port 80
      nodePort: 30080    # External access port

  type: NodePort
```

### Commands

```bash
# Create service
kubectl apply -f service.yml

# Check service
kubectl get service -n nginx-ns

# Check endpoints (shows which pods are connected)
kubectl get endpoints -n nginx-ns
```

**Output:**
```
NAME                 TYPE       CLUSTER-IP     PORT(S)        AGE
static-app-service   NodePort   10.96.45.123   80:30080/TCP   5s
```

> 📸 **Screenshot:**
>
> `[images/service-roundrobin.png]`

---

## 📈 Practical 6 – Scaling Up/Down

### Scale Up to 10 Replicas

```bash
kubectl scale deployment static-app-deployment -n nginx-ns --replicas=10
```

**Output:**
```
deployment.apps/static-app-deployment scaled

NAME                                        READY   STATUS              AGE
static-app-deployment-54bf7f7975-2b64d      0/1     ContainerCreating   4s
static-app-deployment-54bf7f7975-446sp      1/1     Running             4s
static-app-deployment-54bf7f7975-5v4x7      1/1     Running             4s
static-app-deployment-54bf7f7975-6bctg      0/1     ContainerCreating   4s
static-app-deployment-54bf7f7975-bw8qb      0/1     ContainerCreating   4s
static-app-deployment-54bf7f7975-d9qbp      1/1     Running             5m7s
static-app-deployment-54bf7f7975-q7l2g      0/1     ContainerCreating   4s
static-app-deployment-54bf7f7975-t7bj5      1/1     Running             4s
static-app-deployment-54bf7f7975-xdx6m      1/1     Running             25m
static-app-deployment-54bf7f7975-zmrmv      1/1     Running             19s
```

### Scale Down to 3 Replicas

```bash
kubectl scale deployment static-app-deployment -n nginx-ns --replicas=3
```

**Output:**
```
NAME                                        READY   STATUS    AGE
static-app-deployment-54bf7f7975-bw8qb      1/1     Running   43s
static-app-deployment-54bf7f7975-d9qbp      1/1     Running   5m46s
static-app-deployment-54bf7f7975-xdx6m      1/1     Running   26m
```

```
Scale Up:    3 ──────────────────► 10 pods  (30 seconds)
Scale Down:  10 ─────────────────►  3 pods  (instant)
Downtime:    ZERO throughout ✅
```

> 📸 **Screenshot:**
>
> `[images/scaling.png]`

---

## 🌐 Practical 7 – Service & Accessing the App

### Three Ways to Access Your App

```
1. Port Forward (Development)        2. NodePort (Testing)
   localhost:9090                       <node-ip>:30080
        │                                    │
        ▼                                    ▼
    Service                              Service
        │                                    │
        ▼                                    ▼
      Pods                                 Pods

3. LoadBalancer (Production)
   <aws-elb-url>
        │
        ▼
    Service
        │
        ▼
      Pods
```

### Port Forward

```bash
# First attempt - Failed
kubectl port-forward service/static-app-service 8080:80 -n nginx-ns

# Error
Error: bind: address already in use
# Port 8080 was already occupied

# Second attempt - Success ✅
kubectl port-forward service/static-app-service 9090:80 -n nginx-ns

# Output
Forwarding from 127.0.0.1:9090 → 80
Forwarding from [::1]:9090 → 80
Handling connection for 9090
```

**Website accessible at `localhost:9090` ✅**

> 📸 **Screenshot:**
>
> `[images/website-running.png]`

---

## 💡 Key Concepts

### Pod Lifecycle

```
Pending ──► ContainerCreating ──► Running ──► Terminating ──► Terminated
   │                                              │
   │                                              │
   └── Waiting for node            ←── kubectl delete pod
       or image pull
```

### Labels and Selectors

```yaml
# Deployment creates pods with this label
labels:
  app: static-app

# Service finds pods using this selector
selector:
  app: static-app

# They MUST match for the service to route traffic correctly
```

### Namespace Scope

```bash
# Without namespace flag → looks in default
kubectl get pods

# With namespace flag → looks in nginx-ns
kubectl get pods -n nginx-ns

# All namespaces at once
kubectl get pods --all-namespaces
# or
kubectl get pods -A
```

---

## 🎯 Interview Questions

**Q1. What happens when a pod crashes in production?**
```
The kube-controller-manager detects that actual state
does not match desired state and creates a new pod
automatically within seconds. No manual intervention needed.
```

**Q2. What is the difference between a Pod and a Deployment?**
```
Pod:        Single unit. No self healing. No scaling.
            Good for testing only.

Deployment: Manages multiple pods. Self heals.
            Scales on command. Rolling updates.
            Always use in production.
```

**Q3. How does Kubernetes do Round Robin Load Balancing?**
```
The Service object distributes incoming traffic
equally across all pods that match its selector label.
This is built in by default. No extra configuration needed.
```

**Q4. What is etcd and why is it critical?**
```
etcd is a distributed key-value database that stores
the entire cluster state. If etcd is lost, the cluster
loses all configuration. Always backup etcd in production.
```

**Q5. What is the difference between RollingUpdate and Recreate?**
```
RollingUpdate: Updates pods one by one. App stays live.
               Zero downtime. ✅ Use this always.

Recreate:      Kills all pods then starts new ones.
               Causes downtime. ❌ Avoid in production.
```

**Q6. What is a Namespace in Kubernetes?**
```
A Namespace is a logical isolation boundary within a cluster.
It separates resources so different teams or environments
(dev/staging/prod) don't interfere with each other.
```

**Q7. How do you access an app running inside Kubernetes?**
```
1. Port Forward  → Development/debugging only
2. NodePort      → Simple external access
3. LoadBalancer  → Production on cloud (AWS/GCP/Azure)
4. Ingress       → Advanced routing with domain names
```

---

## 📋 kubectl Cheatsheet

### Cluster

```bash
kubectl get nodes                    # List all nodes
kubectl cluster-info                 # Cluster details
kubectl config get-contexts          # List all clusters
kubectl config use-context <name>    # Switch cluster
```

### Namespace

```bash
kubectl get ns                       # List namespaces
kubectl apply -f namespace.yml       # Create namespace
kubectl delete ns nginx-ns           # Delete namespace
```

### Pods

```bash
kubectl get pods -n nginx-ns         # List pods
kubectl get pods -A                  # All namespaces
kubectl describe pod <name> -n nginx-ns  # Pod details
kubectl logs <pod-name> -n nginx-ns  # Pod logs
kubectl exec -it <pod> -n nginx-ns -- bash  # Enter pod
kubectl delete pod <name> -n nginx-ns       # Delete pod
```

### Deployment

```bash
kubectl apply -f deployment.yml          # Create deployment
kubectl get deployment -n nginx-ns       # List deployments
kubectl scale deployment <name> -n nginx-ns --replicas=10  # Scale
kubectl rollout status deployment/<name> # Rollout status
kubectl rollout undo deployment/<name>   # Rollback
kubectl delete deployment <name> -n nginx-ns  # Delete
```

### Service

```bash
kubectl apply -f service.yml             # Create service
kubectl get service -n nginx-ns          # List services
kubectl get endpoints -n nginx-ns        # Check connections
kubectl port-forward service/<name> 9090:80 -n nginx-ns  # Forward
```

### Useful Combos

```bash
kubectl get all -n nginx-ns              # Everything in namespace
kubectl get pods -n nginx-ns -o wide     # Pods with node info
kubectl get pods -n nginx-ns --watch     # Watch live changes
kubectl apply -f .                       # Apply all YAMLs in folder
```

---

## 📁 File Structure

```
DEVOPS-PRACTICAL/
│
├── images/
│   ├── cluster-nodes.png
│   ├── namespace-created.png
│   ├── pod-running.png
│   ├── deployment-created.png
│   ├── self-healing.png
│   ├── service-roundrobin.png
│   ├── scaling.png
│   └── website-running.png
│
├── cluster.yml
├── namespace.yml
├── pod.yml
├── deployment.yml
├── service.yml
└── kubernetes1.md
```

---

🔗 Resources

- [Official Kubernetes Docs](https://kubernetes.io/docs/)
- [KIND Documentation](https://kind.sigs.k8s.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

<div align="center">

⭐ Star this repo if it helped you

🔔 Follow for daily DevOps content

Made with 💙 during #90DaysOfCloudDevOps

</div>
```
