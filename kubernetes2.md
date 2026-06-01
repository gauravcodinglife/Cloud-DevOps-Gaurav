**#90DaysOfCloudDevOps** | Dockerizing a Real App & Deploying to Kubernetes

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Security-red?style=for-the-badge)
![DevOps](https://img.shields.io/badge/DevOps-Practical-green?style=for-the-badge)

---

> 📸 **App Running Live:**
>
> <!-- Image 3: InfraLens app live in browser at 13.206.233.135:3000 -->

## 📌 Table of Contents

- [What We're Building](#-what-were-building)
- [The App – InfraLens](#-the-app--infralens)
- [Practical 1 – Dockerizing the App](#-practical-1--dockerizing-the-app)
- [Practical 2 – Security Scanning with Trivy](#-practical-2--security-scanning-with-trivy)
- [Practical 3 – SSH & EC2 Setup](#-practical-3--ssh--ec2-setup)
- [Practical 4 – Writing Kubernetes YAMLs](#-practical-4--writing-kubernetes-yamls)
- [Practical 5 – Deploying to Kubernetes](#-practical-5--deploying-to-kubernetes)
- [Mistakes I Made](#-mistakes-i-made)
- [Key Concepts](#-key-concepts)
- [Interview Questions](#-interview-questions)
- [Cheatsheet](#-cheatsheet)

---

## 🎯 What We're Building

```
Local App (Next.js)
      │
      ▼
Dockerfile (multi-stage build)
      │
      ▼
Docker Image → Docker Hub (codinggaurav/infralens:latest)
      │
      ▼
EC2 (Ubuntu) → KIND Cluster
      │
      ├── Namespace: infralens
      ├── Deployment: 3 replicas
      └── Service: NodePort → App live on port 3000
```

This day was different from Day 1. Instead of deploying a pre-built image, we:
1. Took a **real Next.js project** (InfraLens)
2. **Dockerized** it ourselves with a multi-stage Dockerfile
3. **Scanned it for vulnerabilities** before pushing
4. **Deployed it to Kubernetes** on an EC2 instance

---

## 🔭 The App – InfraLens

InfraLens is a **pre-deployment infrastructure safety tool**. It analyzes Terraform plans inside pull requests, scores the risk level, and explains the business impact in plain English — before the change is merged.

```
Features:
✅ Comments directly on PRs with risk level and affected services
✅ Maps cloud resource IDs to service tags (Business Context)
✅ Async processing — results in under 2 minutes
✅ Risk scoring out of 100
```

> **Why deploy a real app?**
> Static HTML pages are fine for learning basics.
> Real production apps have build steps, environment vars, and multi-stage Dockerfiles.
> This is what you'll face on the job.

---

## 🐳 Practical 1 – Dockerizing the App

### What is a Multi-Stage Docker Build?

```
Single-stage (naive):                  Multi-stage (production):
──────────────────────                 ─────────────────────────
FROM node:20                           FROM node:20-alpine AS deps
COPY . .                                 COPY package*.json ./
RUN npm install                          RUN npm ci
RUN npm run build
                                       FROM node:20-alpine AS builder
Final image contains:                    COPY --from=deps node_modules ./
  node_modules  (dev deps)               COPY . .
  source code   (not needed)             RUN npm run build
  build cache   (not needed)
  → 1.5 GB image ❌                   FROM node:20-alpine AS runner
                                         COPY --from=builder .next ./.next
                                         COPY --from=builder public ./public
                                         → ~200 MB image ✅
```

### The .dockerignore

Before building, always set up `.dockerignore` to exclude unnecessary files.

> 📸 **Screenshot — `.dockerignore` + docker build running:**
>
> <!-- Image 1: cat .dockerignore + docker build -t infralens . steps 1/18 to 9/18 -->

```
node_modules    ← biggest win, don't copy local deps
.next           ← build output, rebuilt inside container
.git
.gitignore
*.md
.env
.env.local
Dockerfile
```

### The Dockerfile (Multi-Stage)

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build the application
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production runner
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=deps /app/node_modules ./node_modules

EXPOSE 3000
CMD ["npm", "start"]
```

### Build & Push Commands

```bash
# Build the image
docker build -t infralens .

# Tag for Docker Hub
docker tag infralens codinggaurav/infralens:latest

# Push to registry
docker push codinggaurav/infralens:latest

# Verify locally before pushing
docker run -p 3000:3000 infralens
```

### Build Output Explained

```
Step 1/18 : FROM node:20-alpine AS deps     ← Start stage 1
Step 3/18 : COPY package*.json ./           ← Copy manifest only (cache trick)
Step 4/18 : RUN npm ci                      ← Install exact locked deps
Step 5/18 : FROM node:20-alpine AS builder  ← Start stage 2 (fresh layer)
Step 8/18 : COPY . .                        ← Source code goes in here
Step 9/18 : RUN npm run build               ← Next.js compilation
...
---> Using cache                            ← Cache hit = faster builds ✅
```

> **Why `npm ci` over `npm install`?**
> `npm ci` uses `package-lock.json` exactly. Same result every time.
> `npm install` can silently upgrade packages. Bad for reproducibility.

---

## 🔍 Practical 2 – Security Scanning with Trivy

### What is Trivy?

```
Trivy = Open-source vulnerability scanner by Aqua Security

Scans:
  ├── Container images
  ├── File systems (your source code)
  ├── Git repositories
  └── Kubernetes cluster configs

Detects:
  ├── CVEs in dependencies (npm, pip, go, etc.)
  ├── Misconfigurations (Dockerfile, K8s YAML)
  └── Secrets (API keys, tokens accidentally committed)
```

### Running the Scan

```bash
# Scan the filesystem before building
trivy fs .

# Scan the built image
trivy image codinggaurav/infralens:latest

# Scan only for HIGH and CRITICAL (filter out noise)
trivy image --severity HIGH,CRITICAL codinggaurav/infralens:latest
```

### Scan Results

> 📸 **Screenshot — Trivy scan output with vulnerability table:**
>
> <!-- Image 2: trivy fs . showing 14 vulnerabilities in package-lock.json -->

```
Target: package-lock.json (npm)
Total: 14 (LOW: 2, MEDIUM: 5, HIGH: 7, CRITICAL: 0)

Library  │ CVE             │ Severity │ Status │ Fixed Version
─────────┼─────────────────┼──────────┼────────┼──────────────
next     │ CVE-2026-44573  │ HIGH     │ fixed  │ 15.5.16, 16.2.5
next     │ CVE-2026-44574  │ HIGH     │ fixed  │ (same)
next     │ CVE-2026-44575  │ HIGH     │ fixed  │ (same)
```

**What these CVEs mean:**
```
CVE-2026-44573: Middleware/Proxy bypass in Pages Router
CVE-2026-44574: Middleware bypass via dynamic route parameter injection
CVE-2026-44575: Middleware/Proxy bypass in App Router

Root cause: next@16.2.4 is installed, but fix is in 16.2.5
Fix: bump next version in package.json, then rebuild
```

### Vulnerability Severity Guide

| Severity | Action | Example |
|---|---|---|
| **CRITICAL** | Block deployment immediately | Remote code execution |
| **HIGH** | Fix before next release | Auth bypass (like above) |
| **MEDIUM** | Fix in next sprint | Info disclosure |
| **LOW** | Track and fix eventually | Minor edge cases |

### The DevSecOps Mindset

```
Old way:           New way (DevSecOps):
─────────          ───────────────────
Code → Build       Code → Scan → Build → Scan Image → Deploy
         ↓                 ↑                    ↑
       Deploy       Catch early          Catch in image
         ↓          (cheaper fix)        (before prod)
    Security audit
    (expensive fix)
```

---

## 🖥️ Practical 3 – SSH & EC2 Setup

### Setting Up SSH Between Machines

> 📸 **Screenshot — SSH setup, port verification, scp file transfer:**
>
> <!-- Image 5: systemctl start ssh, ss -tulnp showing ports 22/80/8080, scp hello.txt transfer -->

```bash
# Start SSH service on remote machine
sudo systemctl start ssh

# Verify SSH is listening on port 22
ss -tulnp | grep 22

# Output
tcp   LISTEN  0  4096  0.0.0.0:22  0.0.0.0:*
tcp   LISTEN  0  4096  [::]:22     [::]:*
```

### What is `ss -tulnp`?

```
ss   = socket statistics (replacement for netstat)
-t   = TCP sockets
-u   = UDP sockets
-l   = listening sockets only
-n   = show port numbers (not service names)
-p   = show process using the socket

Output from the scan shows ports open:
:22    → SSH ✅
:80    → HTTP (app)
:8080  → alternate HTTP
:3000  → Next.js app (InfraLens)
```

### SCP – Copying Files to Remote

```bash
# Copy file from local to remote EC2
scp hello.txt gaurav@172.24.148.85:/tmp/.

# Output
The authenticity of host '172.24.148.85' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue? yes

hello.txt     100%    9    7.1KB/s   00:00
```

```
scp = Secure Copy Protocol
      Uses SSH tunnel to transfer files

Syntax: scp <local-file> <user>@<host>:<remote-path>

Common uses:
  scp file.txt user@host:/tmp/         → local to remote
  scp user@host:/var/log/app.log ./    → remote to local
  scp -r ./folder user@host:/home/     → copy entire folder
```

---

## 📝 Practical 4 – Writing Kubernetes YAMLs

### The Three Files

> 📸 **Screenshot — `cat deployment.yml`, `cat ns.yml`, `cat service.yml`:**
>
> <!-- Image 4: terminal showing all three YAML files cat'd in sequence -->

```
infralens/k8s/
├── ns.yml          ← Namespace first, always
├── deployment.yml  ← The app logic
└── service.yml     ← How the world reaches it
```

### ns.yml – Namespace

```yaml
kind: Namespace
apiVersion: v1

metadata:
  name: infralens
```

```
Why a dedicated namespace?
  nginx-ns   → Day 1 experiments
  infralens  → This real app
  payment    → Future payment service

Namespaces = separation of concerns at the cluster level
```

### deployment.yml – The App

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: infralens-deployment
  namespace: infralens
  labels:
    app: infralens

spec:
  replicas: 3

  selector:
    matchLabels:
      app: infralens

  template:
    metadata:
      labels:
        app: infralens

    spec:
      containers:
        - name: infralens
          image: codinggaurav/infralens:latest
          ports:
            - containerPort: 80
```

### service.yml – The Entry Point

```yaml
apiVersion: v1
kind: Service

metadata:
  name: infralens-service
  namespace: infralens

spec:
  selector:
    app: infralens

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080

  type: NodePort
```

### How Labels Wire Everything Together

```
Deployment creates pods with:          Service routes traffic to pods where:
  labels:                                selector:
    app: infralens          ←──────────    app: infralens

If labels don't match → Service has no endpoints → Traffic goes nowhere ❌
If labels match      → Service finds all pods   → Traffic routed correctly ✅
```

---

## 🚀 Practical 5 – Deploying to Kubernetes

### Apply Order Matters

```bash
# Step 1: Namespace must exist first
kubectl apply -f ns.yml

# Step 2: Deploy the app
kubectl apply -f deployment.yml

# Step 3: Expose it
kubectl apply -f service.yml

# Or apply everything at once (K8s handles order)
kubectl apply -f .
```

### Verify Everything

```bash
# Check all resources in namespace
kubectl get all -n infralens

# Expected output
NAME                                        READY   STATUS    RESTARTS   AGE
pod/infralens-deployment-7d9f4b6c5-abc12    1/1     Running   0          2m
pod/infralens-deployment-7d9f4b6c5-def34    1/1     Running   0          2m
pod/infralens-deployment-7d9f4b6c5-ghi56    1/1     Running   0          2m

NAME                       TYPE       CLUSTER-IP      PORT(S)        AGE
service/infralens-service   NodePort   10.96.112.44   80:30080/TCP   1m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/infralens-deployment   3/3     3            3           2m
```

### Access the App

```bash
# Option 1: Port forward for local testing
kubectl port-forward service/infralens-service 3000:80 -n infralens

# Option 2: Access via EC2 public IP + nodePort
http://<ec2-public-ip>:30080

# Option 3: Port forward if nodePort not working on KIND
kubectl port-forward service/infralens-service 9090:80 -n infralens
```

---

## ⚠️ Mistakes I Made

### Mistake 1 – containerPort mismatch

```yaml
# ❌ Wrong: App runs on 3000, not 80
containers:
  - name: infralens
    image: codinggaurav/infralens:latest
    ports:
      - containerPort: 80   # Next.js runs on 3000!

# ✅ Fix: Match the actual port the app uses
    ports:
      - containerPort: 3000
```

### Mistake 2 – Forgot imagePullPolicy

```yaml
# Problem: Kubernetes cached the old image, didn't pull new one
# Fix: Add this to always pull fresh
imagePullPolicy: Always
```

### Mistake 3 – Wrong namespace in commands

```bash
# ❌ Looked in default namespace, found nothing
kubectl get pods

# ✅ Always specify the namespace
kubectl get pods -n infralens
```

### Mistake 4 – Port 8080 already in use

```bash
# Error during port-forward
Error: bind: address already in use

# Fix: Use a different local port
kubectl port-forward service/infralens-service 9090:80 -n infralens
```

---

## 💡 Key Concepts

### Multi-Stage vs Single-Stage Builds

```
Single stage:   All tools + source + build artifacts in final image
                Image size: 1.5 GB+
                Security risk: build tools exposed ❌

Multi-stage:    Only production artifacts copied to final image
                Image size: 150-250 MB
                No build tools, no source code in prod ✅
```

### Trivy in CI/CD Pipeline

```yaml
# GitHub Actions example
- name: Scan for vulnerabilities
  run: |
    trivy image --exit-code 1 \
                --severity CRITICAL \
                codinggaurav/infralens:latest
    # exit-code 1 = fail the pipeline on CRITICAL vulns
```

### Port Mapping in Kubernetes

```
Request flow for NodePort:

  Browser
     │
     │ :30080 (nodePort on EC2)
     ▼
  EC2 Node
     │
     │ :80 (service port)
     ▼
  Service (infralens-service)
     │
     │ :3000 (targetPort, what the container listens on)
     ▼
  Container (Next.js app)
```

---

## 🎯 Interview Questions

**Q1. What is a multi-stage Docker build and why use it?**
```
Multi-stage builds use multiple FROM instructions in a single Dockerfile.
Each stage can copy artifacts from the previous one.
The final image only contains what the production app needs —
no build tools, no dev dependencies, no source code.
Result: smaller image, faster pulls, smaller attack surface.
```

**Q2. What does Trivy scan for?**
```
Trivy scans:
  1. CVEs in language dependencies (npm, pip, go modules)
  2. OS package vulnerabilities
  3. Secrets accidentally committed (API keys, tokens)
  4. Misconfigurations in Dockerfile and K8s YAML

It works on: filesystem, container images, git repos, K8s clusters.
```

**Q3. What's the difference between port, targetPort, and nodePort in a Service?**
```
port:       The port the Service object itself listens on (inside cluster)
targetPort: The port on the Pod/container to forward traffic to
nodePort:   The port exposed on every Node for external access (30000-32767)

Traffic: Browser → nodePort → port → targetPort → Container
```

**Q4. Why use `npm ci` instead of `npm install` in Dockerfiles?**
```
npm ci:
  - Reads package-lock.json exactly
  - Fails if lock file is missing or mismatched
  - Reproducible: same result every build
  - Faster (skips package resolution)

npm install:
  - Can update lock file during install
  - Non-deterministic: may install different versions
  - Never use in CI/CD or Dockerfiles
```

**Q5. What is the purpose of `.dockerignore`?**
```
.dockerignore excludes files from the Docker build context.
The build context is everything sent to the Docker daemon before building.

Without it:
  - node_modules (hundreds of MB) gets copied needlessly
  - .env files with secrets get included in the image
  - Build is slower, image may contain sensitive data

With it:
  - Only essential files reach the daemon
  - Faster builds, cleaner images, no accidental secret leaks
```

**Q6. What happens if the label in a Deployment doesn't match the selector in a Service?**
```
The Service will have no Endpoints.
kubectl get endpoints -n <ns> will show: <none>

Traffic sent to the Service will fail — no pods to forward to.
Always verify: selector in Service matches labels on Pod template in Deployment.
```

---

## 📋 Cheatsheet

### Docker

```bash
docker build -t <image> .                     # Build image
docker build -t <image> --no-cache .          # Force fresh build
docker tag <image> <registry>/<image>:tag     # Tag for registry
docker push <registry>/<image>:tag            # Push to Docker Hub
docker run -p 3000:3000 <image>               # Run locally
docker images                                 # List images
docker rmi <image>                            # Remove image
```

### Trivy

```bash
trivy fs .                                    # Scan filesystem
trivy image <image>                           # Scan image
trivy image --severity HIGH,CRITICAL <image>  # Filter severity
trivy image --exit-code 1 <image>             # Fail on vuln (CI use)
trivy k8s --report summary cluster           # Scan whole cluster
```

### SSH & SCP

```bash
sudo systemctl start ssh                      # Start SSH daemon
ss -tulnp | grep 22                           # Verify SSH listening
scp file.txt user@host:/path/                 # Copy to remote
scp user@host:/path/file.txt ./               # Copy from remote
scp -r ./folder user@host:/path/              # Copy folder
ssh user@host                                 # Connect to remote
```

### Kubernetes

```bash
kubectl apply -f .                            # Apply all YAMLs
kubectl get all -n <ns>                       # Everything in ns
kubectl get pods -n <ns> --watch              # Watch live
kubectl logs <pod> -n <ns>                    # Pod logs
kubectl describe pod <pod> -n <ns>            # Debug pod
kubectl exec -it <pod> -n <ns> -- sh          # Enter pod (alpine)
kubectl port-forward svc/<name> 9090:80 -n <ns>  # Forward port
kubectl get endpoints -n <ns>                 # Check service wiring
```

---

## 📁 File Structure

```
DEVOPS-PRACTICAL/
│
├── InfraLens/                   ← The app source
│   ├── .dockerignore
│   ├── Dockerfile               ← Multi-stage
│   ├── package.json
│   └── src/
│
└── k8s/
    ├── ns.yml
    ├── deployment.yml
    └── service.yml
```

---

## 🔗 Resources

- [Trivy Documentation](https://trivy.dev/latest/)
- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Kubernetes Service Types](https://kubernetes.io/docs/concepts/services-networking/service/)
- [npm ci vs npm install](https://docs.npmjs.com/cli/v10/commands/npm-ci)

---

<div align="center">

⭐ Star this repo if it helped you

🔔 Follow for daily DevOps content

Made with 💙 during #90DaysOfCloudDevOps

</div>