# Blue-Green Deployment — Zero to Hands-On Guide

A complete, beginner-friendly walkthrough of Blue-Green Deployment: what it is, why it matters, and how to actually build one on your own laptop using Docker, KIND (Kubernetes in Docker), and NGINX Ingress. Written so a junior DevOps/Cloud engineer can follow every step without getting lost.

---

## Table of Contents

1. [The Problem We're Solving](#1-the-problem-were-solving)
2. [What is Blue-Green Deployment?](#2-what-is-blue-green-deployment)
3. [How It Works (The Process)](#3-how-it-works-the-process)
4. [Other Deployment Strategies (So You Know the Alternatives)](#4-other-deployment-strategies-so-you-know-the-alternatives)
5. [Architecture Diagram](#5-architecture-diagram)
6. [Prerequisites](#6-prerequisites)
7. [Hands-On Lab: Build It Yourself](#7-hands-on-lab-build-it-yourself)
   - [Step 1: Install Docker](#step-1-install-docker)
   - [Step 2: Install kubectl](#step-2-install-kubectl)
   - [Step 3: Install KIND](#step-3-install-kind)
   - [Step 4: Create a KIND Cluster](#step-4-create-a-kind-cluster)
   - [Step 5: Install NGINX Ingress Controller](#step-5-install-nginx-ingress-controller)
   - [Step 6: Deploy the "Blue" Environment](#step-6-deploy-the-blue-environment)
   - [Step 7: Deploy the "Green" Environment](#step-7-deploy-the-green-environment)
   - [Step 8: Test Green Before Going Live](#step-8-test-green-before-going-live)
   - [Step 9: Switch Traffic from Blue to Green](#step-9-switch-traffic-from-blue-to-green)
   - [Step 10: Roll Back (If Something Goes Wrong)](#step-10-roll-back-if-something-goes-wrong)
   - [Step 11: Clean Up](#step-11-clean-up)
8. [Handling the Database](#8-handling-the-database)
9. [Best Practices](#9-best-practices)
10. [Troubleshooting Tips](#10-troubleshooting-tips)
11. [Summary Table](#11-summary-table)
12. [References](#12-references)

---

## 1. The Problem We're Solving

Every time you deploy a new version of an app, you risk:

- **Downtime** — users can't use the app while you deploy.
- **Bugs in production** — a broken release affects real users immediately.
- **Hard rollbacks** — if something breaks, going back to the old version is slow and stressful.

**Blue-Green Deployment** fixes all three by giving you:

- ✅ Zero-downtime deployments
- ✅ A safe place to test the new version before real users see it
- ✅ An instant "undo button" if something goes wrong

### Goals of this strategy
1. Minimize downtime during deployments.
2. Reduce risk by keeping the new version isolated from live traffic until it's proven.
3. Let you test new features in an environment identical to production.
4. Give you instant rollback to a known-good version.
5. Work well with cloud-native tools like AWS, Kubernetes, and Terraform.

### Who does this job?
A **DevOps / Cloud Engineer** doing Blue-Green deployments is typically responsible for:
- Creating two parallel environments (Blue = current live, Green = new version)
- Routing traffic using something like an AWS ALB, Route 53, or a Kubernetes Ingress
- Handling the database safely (shared or replicated) during the switch
- Making sure shared storage (like S3 buckets) is versioned and accessible to both environments
- Automating the whole thing with Terraform/Infrastructure-as-Code
- Monitoring the new deployment with tools like CloudWatch, Prometheus, or Grafana

---

## 2. What is Blue-Green Deployment?

> **Blue-Green Deployment** is a release strategy where you run **two identical environments** — only one of which is live at any time — so you can switch between them instantly.

- **Blue** = your current, stable production environment (what users are using right now).
- **Green** = the new version you want to release.

Here's the flow in plain English:

1. Users are currently being served by **Blue**.
2. You deploy the new version into **Green**, a full copy of production, but it isn't receiving real user traffic yet.
3. You test Green thoroughly.
4. Once you're confident, you flip a switch (a load balancer or router setting) so all traffic now goes to **Green**.
5. **Blue** stays running, untouched, in case anything goes wrong.
6. If Green has issues, you flip traffic back to Blue **instantly** — that's your rollback.

**Key benefit: instant rollback**, because the old version never gets deleted until you're sure the new one works.

---

## 3. How It Works (The Process)

Think of this as 7 stages:

### Stage 1 — Duplicate the Environment
- Spin up a **Green** environment that is an exact copy of **Blue** (same app code — new version, same infrastructure shape: same number of pods/nodes, same configuration pattern).
- Blue keeps serving live traffic the entire time — nothing changes for users yet.

### Stage 2 — Handle the Database
You have two options:

| Option | How it works | Trade-off |
|---|---|---|
| **Shared Database** | Both Blue and Green point to the same RDS/DynamoDB instance | Simple, but **all schema changes must be backward-compatible** (Blue's old code must still work with any new schema) |
| **Separate Databases** | Green gets its own database | Safer isolation, but you need to keep data in sync (replication) |

### Stage 3 — Shared Storage (S3 / Static Assets)
- Both environments usually share the **same S3 bucket** for static files (images, CSS, JS, uploads).
- Turn on **S3 bucket versioning** so you can roll back a bad file upload just like you roll back code.

### Stage 4 — Test Green
- Run smoke tests, integration tests, and load tests against Green **before it gets real traffic**.
- Watch logs, metrics, and alerts (CloudWatch, Prometheus, Grafana).
- Confirm Green behaves correctly without touching Blue at all.

### Stage 5 — Switch Traffic
- Update your router so traffic goes from Blue → Green. In AWS/Kubernetes this usually means:
  - **ALB target groups**, or
  - **Route 53 weighted routing**, or
  - A **Kubernetes Ingress/Service** selector change
- Green is now live. Blue is idle, but still running and ready.

### Stage 6 — Monitor & Rollback
- Watch Green closely for errors, latency spikes, or anything unusual.
- If something's wrong, switch traffic back to Blue **immediately** — this is your safety net.

### Stage 7 — Clean Up / Promote
- Once Green has proven stable for a while, you can shut down Blue or repurpose it as the "next Green" for the following release.

---

## 4. Other Deployment Strategies (So You Know the Alternatives)

Blue-Green isn't the only option. Here's how it compares:

### Rolling Deployment
Update the app **one server/pod at a time**, gradually replacing the old version.
- ✅ No full downtime
- ⚠️ Risk: users might briefly hit a mix of old and new versions
- 🟢 Best for: Kubernetes, microservices

### Canary Deployment
Release to a **small percentage of users first** (e.g., 5% → 20% → 100%), watching metrics as you go.
- 🟢 Best for: high-risk releases where you want early warning signs
- ⚠️ Costs more in monitoring/setup complexity

### Recreate Deployment
**Stop the old version completely, then start the new one.**
- ✅ Simple and cheap
- ❌ Causes downtime
- 🟢 Best for: dev/test environments only — not production

### A/B Deployment
Run **two versions with different behavior** and split users between them, usually to compare features or UX (not just to release safely).
- 🟢 Best for: product experiments and UX decisions
- ⚠️ Needs more complex traffic-routing logic

### Quick Comparison

| Strategy | Downtime | Risk | Popular With |
|---|---|---|---|
| Blue-Green | ❌ None | Low | Enterprises |
| Rolling | ❌ None | Medium | Kubernetes teams |
| Canary | ❌ None | Very Low | DevOps teams |
| Recreate | ✅ Yes | High | Small apps |
| A/B | ❌ None | Medium | Product teams |

---

## 5. Architecture Diagram

```
                         [ Users ]
                             |
                             v
                 [ ALB / Route 53 / Ingress ]
                     |                |
                     v                v
              [ Blue (live) ]   [ Green (new) ]
                     |                |
                     +-------+--------+
                             |
                     [ Shared RDS / DynamoDB ]
                     [ Shared S3 Bucket ]
                     [ Same VPC / Same Subnets ]
                             |
                       [ Terraform ]
              (provisions & switches Blue/Green)
```

- The router (ALB, Route 53, or Kubernetes Ingress) decides whether traffic goes to Blue or Green.
- Both environments typically sit inside the same network (VPC) and can share the database and S3 bucket.
- Terraform (or any Infrastructure-as-Code tool) automates creating both environments and flipping the switch, so deployments are repeatable and not done by hand.

---

## 6. Prerequisites

Before starting the hands-on lab, make sure you have:

- A Linux machine (Ubuntu recommended) or WSL2 on Windows
- `sudo` access
- Basic comfort with the terminal
- Internet access to download packages

You do **not** need AWS for this lab — we'll simulate the whole Blue-Green setup locally using **KIND** (Kubernetes IN Docker), which is free and runs entirely on your machine. The same concepts map directly onto AWS ALB / Route 53 / EKS later.

---

## 7. Hands-On Lab: Build It Yourself

We're going to build a real, working Blue-Green setup on your laptop, step by step.

### Step 1: Install Docker

Docker is required because KIND runs Kubernetes nodes as Docker containers.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

> 💡 `newgrp docker` refreshes your group membership so you can run `docker` without `sudo`. If it doesn't work, just log out and log back in.

---

### Step 2: Install kubectl

`kubectl` is the command-line tool you use to talk to any Kubernetes cluster.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

### Step 3: Install KIND

KIND lets you run a full Kubernetes cluster inside Docker containers on your own machine — perfect for practicing without needing a cloud account.

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

---

### Step 4: Create a KIND Cluster

We'll create a cluster with:
- 1 control-plane node
- 2 worker nodes
- Ports 80 and 443 mapped to your machine, so we can access apps via a browser like a real deployment

Create a file called `kind-config.yml`:

```yaml
# kind-config.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  # Control plane node
  - role: control-plane
    image: kindest/node:v1.28.0
    extraPortMappings:
      - containerPort: 80   # HTTP
        hostPort: 80
        protocol: TCP
      - containerPort: 443  # HTTPS
        hostPort: 443
        protocol: TCP
  # Worker node 1
  - role: worker
    image: kindest/node:v1.28.0
  # Worker node 2
  - role: worker
    image: kindest/node:v1.28.0
```

Now create the cluster:

```bash
kind create cluster --name bluegreen-demo --config=kind-config.yml
kubectl cluster-info
kubectl get nodes
```

You should see 3 nodes: 1 control-plane and 2 workers, all `Ready`.

---

### Step 5: Install NGINX Ingress Controller

The Ingress controller is what will route traffic to Blue or Green, similar to how an AWS ALB works.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

This creates:
- The `ingress-nginx` namespace
- The controller Deployment
- RBAC roles and permissions
- Admission webhooks
- A LoadBalancer Service

Verify it's running (this can take a minute or two):

```bash
kubectl get pods -n ingress-nginx
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

### Step 6: Deploy the "Blue" Environment

This represents your **current, live production version**.

Create `blue-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: myapp
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: blue-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: blue-content
data:
  index.html: |
    <html><body style="background:#3498db;color:white;text-align:center;font-family:sans-serif;">
    <h1>🔵 BLUE — Version 1.0 (Current Production)</h1>
    </body></html>
---
apiVersion: v1
kind: Service
metadata:
  name: app-blue-svc
spec:
  selector:
    app: myapp
    version: blue
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f blue-deployment.yaml
kubectl get pods -l version=blue
```

Now create the **live Service** that Ingress will point to. This is the Service whose selector we'll flip later:

```yaml
# live-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-live-svc
spec:
  selector:
    app: myapp
    version: blue     # 👈 currently pointing at Blue
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-live-svc
                port:
                  number: 80
```

```bash
kubectl apply -f live-service.yaml
```

Test it in your browser or with curl:

```bash
curl http://localhost
```

You should see the **BLUE** page. 🎉 Your "production" is live.

---

### Step 7: Deploy the "Green" Environment

This is the **new version** you want to release — deployed in parallel, but **not yet receiving live traffic**.

Create `green-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: myapp
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: green-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: green-content
data:
  index.html: |
    <html><body style="background:#2ecc71;color:white;text-align:center;font-family:sans-serif;">
    <h1>🟢 GREEN — Version 2.0 (New Release)</h1>
    </body></html>
---
apiVersion: v1
kind: Service
metadata:
  name: app-green-svc
spec:
  selector:
    app: myapp
    version: green
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f green-deployment.yaml
kubectl get pods -l version=green
```

Notice: **Green is running, but `app-live-svc` still points to Blue.** Real users still only see Blue. This is exactly the safety property we want.

---

### Step 8: Test Green Before Going Live

Test Green **directly**, without touching live traffic, by temporarily port-forwarding to its own Service:

```bash
kubectl port-forward svc/app-green-svc 8080:80
```

Then in another terminal:

```bash
curl http://localhost:8080
```

You should see the **GREEN** page. Run whatever smoke tests, health checks, or load tests you need here. Stop the port-forward (`Ctrl+C`) once you're satisfied.

---

### Step 9: Switch Traffic from Blue to Green

This is the moment of truth — the actual "switch." All we do is change the selector on `app-live-svc` from `blue` to `green`:

```bash
kubectl patch service app-live-svc -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'
```

Verify the switch:

```bash
curl http://localhost
```

You should now see the **GREEN** page — instantly, with **zero downtime**. Blue's pods are still running in the background, untouched.

> 🧠 **In AWS**, this same idea is done by shifting the ALB target group or updating Route 53 weighted routing from 100% Blue / 0% Green to 0% Blue / 100% Green.

---

### Step 10: Roll Back (If Something Goes Wrong)

If Green has a problem after going live, rollback is just as instant — flip the selector back:

```bash
kubectl patch service app-live-svc -p '{"spec":{"selector":{"app":"myapp","version":"blue"}}}'
curl http://localhost
```

You're back on Blue immediately. This is the whole point of Blue-Green: **the "undo" is a one-line command, not a redeploy.**

---

### Step 11: Clean Up

Once Green has proven stable for a while, you can retire Blue (or keep it as the base for the next release):

```bash
kubectl delete deployment app-blue
kubectl delete configmap blue-content
kubectl delete service app-blue-svc
```

To tear down the whole practice cluster when you're done with the lab:

```bash
kind delete cluster --name bluegreen-demo
```

---

## 8. Handling the Database

The trickiest part of Blue-Green in a real app is the database, because unlike stateless app pods, you can't just duplicate data safely.

- **Shared database (simplest):** Both Blue and Green use the same RDS/DynamoDB instance. The rule you must follow: **any schema migration has to be backward-compatible**, meaning the *old* app code (Blue) must still work correctly even after the new schema change is applied. This usually means:
  - Add new columns as nullable (don't drop or rename existing ones during the switch)
  - Deploy schema changes *before* deploying new app code
  - Remove old/unused columns only after Blue is fully retired
- **Separate databases (safer, more work):** Green gets its own database copy, kept in sync with replication until the switch happens. More isolation, but more operational complexity.

For a junior engineer: **start with a shared database and backward-compatible migrations** — it's simpler and covers most real-world cases.

---

## 9. Best Practices

1. **Immutable deployments** — always deploy *new* pods for a new version rather than modifying existing ones in place.
2. **Automate with Terraform / IaC** — don't create Blue/Green environments by hand; script it so it's repeatable and less error-prone.
3. **Backward-compatible database migrations** — never break Blue while Green is being tested.
4. **Monitoring & alerts** — wire up CloudWatch, Prometheus, and Grafana (or equivalents) so you actually notice problems in Green before/after the switch.
5. **Controlled traffic switch** — use ALB target groups or Route 53 weighted routing (or a Kubernetes Service/Ingress change, as we did above) for a clean, fast switch.
6. **Keep Blue around** for a while after switching — don't delete it immediately; it's your safety net.

---

## 10. Troubleshooting Tips

| Problem | Likely Cause | Fix |
|---|---|---|
| `docker: permission denied` | Your user isn't in the `docker` group yet | Run `newgrp docker` or log out/in again |
| `kind create cluster` hangs or fails | Docker isn't running | `sudo systemctl start docker` |
| `curl http://localhost` returns nothing | Ingress controller pods aren't ready yet | `kubectl get pods -n ingress-nginx` and wait for `Running` |
| Switching the Service selector doesn't change the response | Browser/curl cache | Try `curl -v http://localhost` or a hard refresh; DNS/cache isn't usually an issue with a raw IP but worth ruling out |
| Green pods stuck in `Pending` | Not enough resources on your machine | Reduce `replicas` to 1, or free up Docker resources |

---

## 11. Summary Table

| Step | Action |
|---|---|
| 1 | Duplicate environment (Blue + Green) |
| 2 | Handle databases (shared or replicated) |
| 3 | Share S3/storage assets with versioning |
| 4 | Test Green environment |
| 5 | Switch traffic from Blue → Green |
| 6 | Monitor & roll back if needed |
| 7 | Clean up / promote Green |

### Key Benefits
- ✅ Zero downtime deployments
- ✅ Instant rollback on failure
- ✅ Safe testing in a production-like environment
- ✅ Works with AWS, Kubernetes, and Terraform-based workflows

---

## 12. References

- Reference video: https://youtu.be/_T3vfkl-3Vk?si=dCii6CijT4MpXDW5
- Example repository: https://github.com/LondheShubham153/kubestarter/tree/main/Deployment_Strategies
- KIND docs: https://kind.sigs.k8s.io/
- NGINX Ingress Controller: https://kubernetes.github.io/ingress-nginx/
- Kubernetes docs: https://kubernetes.io/docs/home/

---

*This guide turns the original Blue-Green Deployment notes into a full, runnable lab so a junior engineer can go from zero knowledge to an actual working Blue-Green setup on their own machine, then map the same concepts onto AWS.*
