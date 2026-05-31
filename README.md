# Fault-Tolerant Web Hosting on Kubernetes

> Build a web application that **heals itself when it crashes**, **scales up under load**, and **routes traffic through a single entry point** — all handled by Kubernetes, with zero manual intervention.
>
> Built with **React** + **Docker** + **Kubernetes (kind)** + **NGINX Ingress Controller**. Runs entirely on a local cluster.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![React](https://img.shields.io/badge/React-17-61DAFB?logo=react&logoColor=black)](https://reactjs.org/)
[![NGINX Ingress](https://img.shields.io/badge/NGINX-Ingress-009639?logo=nginx&logoColor=white)](https://kubernetes.github.io/ingress-nginx/)

---

## What This Is

A single container running a web app is fragile — if it crashes, your site goes down. This project demonstrates the three pillars that make Kubernetes the industry standard for production hosting: **self-healing** (crashed pods restart automatically), **horizontal scaling** (spin up more replicas with one command), and **ingress routing** (a single entry point that load-balances across all replicas).

The app itself is intentionally simple — a React page that says "Hello from Mars!" — because the interesting part isn't the app. It's what happens when you **kill a pod and watch Kubernetes bring it back to life in seconds**, or **scale from 3 to 7 replicas with a single command** and see traffic spread across all of them.

---

## Architecture

```
                External Traffic (HTTP)
                         │
                         ▼
              ┌─────────────────────┐
              │   NGINX Ingress     │   ← single entry point
              │   Controller        │     routes path "/" to kubernetes-svc
              └─────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   kubernetes-svc    │   ← NodePort Service (31111 → 3000)
              │   load balances     │     distributes traffic across replicas
              └─────────────────────┘
                    │    │    │
            ┌───────┘    │    └───────┐
            ▼            ▼            ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │  Pod 1   │ │  Pod 2   │ │  Pod 3   │  ...up to N replicas
      │  React   │ │  React   │ │  React   │
      │  :3000   │ │  :3000   │ │  :3000   │
      └──────────┘ └──────────┘ └──────────┘
           │             │             │
           └──── restartPolicy: Always ┘
                 (self-healing)
```

Kill any pod — Kubernetes immediately spins up a replacement. Scale to 7 replicas — the Service automatically routes traffic across all of them. No config changes, no downtime.

---

## Project Structure

```
fault-tolerant-web-hosting-on-kubernetess/
│
├── Application/                  # The React web app
│   ├── Dockerfile                # node:latest → npm install → npm start
│   ├── package.json              # "hello-mars" React app (port 3000)
│   ├── src/
│   │   ├── App.js                # "Hello from Mars!" UI
│   │   └── index.css             # Styling
│   └── public/                   # Static assets
│
├── deployment.yaml               # Deployment: 5 replicas, restartPolicy: Always
├── service.yaml                  # NodePort Service: 31111 → 3000
└── ingress.yaml                  # NGINX Ingress: path "/" → kubernetes-svc
```

---

## How It Works

### 1. Self-Healing — `restartPolicy: Always`

```yaml
# deployment.yaml
spec:
  restartPolicy: Always
```

This single line is the difference between "site goes down at 3 AM" and "site fixes itself at 3 AM." Think of it like a night watchman stationed next to every server — the moment one goes dark, the watchman flips it back on, no phone call needed. Kubernetes monitors every pod and restarts any that crash, enter an error state, or get killed. You can verify this yourself: delete a pod and watch the replica count snap right back.

### 2. Horizontal Scaling — Replicas

```yaml
# deployment.yaml
spec:
  replicas: 5
```

Five identical copies of your app, running simultaneously. It's like a grocery store opening 5 checkout lanes instead of 1 — if one lane closes (a pod dies), customers (traffic) flow to the other 4 with zero disruption. And you can resize on the fly:

```bash
kubectl scale --replicas=7 -f deployment.yaml   # scale up to 7
kubectl scale --replicas=3 -f deployment.yaml   # scale back down to 3
```

The Service automatically discovers new pods and starts routing traffic to them. No config changes, no restart.

### 3. Ingress — A Single Front Door

```yaml
# ingress.yaml
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-svc
                port:
                  number: 31111
```

Without an Ingress, every service needs its own external port — messy and hard to manage. The NGINX Ingress Controller acts like a hotel reception desk: all guests (HTTP requests) arrive at one door, and reception knows exactly which room (service) to route them to based on the URL path. For this project there's one rule — everything at `/` goes to `kubernetes-svc` — but in production you'd add paths like `/api`, `/admin`, `/docs`, each routing to a different backend service.

---

## Quickstart

### Prerequisites

- [Docker](https://www.docker.com/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kind](https://kind.sigs.k8s.io/) (or minikube / Docker Desktop)

### 1. Create a cluster

```bash
kind create cluster
```

### 2. Build and push the app image

```bash
cd Application
docker build -t <your-dockerhub-username>/hello-mars:latest .
docker login -u <your-dockerhub-username>
docker push <your-dockerhub-username>/hello-mars:latest
```

Then update the `image:` field in `deployment.yaml` to match your image name. The manifest currently references `keerthangowda09907/sample`.

### 3. Deploy the application

```bash
# Apply the deployment (5 replicas)
kubectl apply -f deployment.yaml

# Expose it via a Service
kubectl apply -f service.yaml

# Verify pods and service
kubectl get pods
kubectl get service
```

### 4. Access the app (before Ingress)

```bash
kubectl port-forward svc/kubernetes-svc --address 0.0.0.0 31111:3000
```

Open **http://localhost:31111** — you'll see "Hello from Mars!"

### 5. Deploy the NGINX Ingress Controller

```bash
# Install the NGINX Ingress Controller (v1.12.3)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.3/deploy/static/provider/cloud/deploy.yaml

# Wait for it to be ready
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx

# Apply your Ingress rule
kubectl apply -f ingress.yaml
```

Traffic to `/` now routes through the Ingress Controller → Service → any of the running pods.

---

## Fault Tolerance in Action

These are the experiments that prove the system works. Run them yourself:

```bash
# 1. See all 5 replicas running
kubectl get pods

# 2. Kill a pod — watch Kubernetes immediately replace it
kubectl delete pod <any-pod-name>
kubectl get pods    # still 5 pods — the deleted one was replaced

# 3. Scale up to 7
kubectl scale --replicas=7 -f deployment.yaml
kubectl get pods    # 7 pods running

# 4. Scale down to 3
kubectl scale --replicas=3 -f deployment.yaml
kubectl get pods    # 3 pods, extras gracefully terminated

# 5. Watch pods in real-time while you kill them
kubectl get pods -w &
kubectl delete pod <pod-name>
# You'll see: old pod Terminating → new pod ContainerCreating → Running
```

---

## Tech Stack

| Component | Choice |
|---|---|
| Web App | React 17 ("Hello from Mars!") |
| Container | Docker (`node:latest`, port 3000) |
| Orchestration | Kubernetes |
| Local Cluster | kind |
| Fault Tolerance | `restartPolicy: Always` + replica sets |
| Service | NodePort (`31111` → `3000`) |
| Ingress | NGINX Ingress Controller (v1.12.3) |
| Image Registry | Docker Hub (`keerthangowda09907/sample`) |

---

## What I Learned

- **Self-healing** with `restartPolicy: Always` — Kubernetes restarts crashed pods without human intervention
- **Horizontal scaling** with `replicas` and `kubectl scale` — adding and removing instances on the fly
- **Service-level load balancing** — a NodePort Service automatically distributes traffic across all matching pods
- **Ingress routing** — using the NGINX Ingress Controller as a single external entry point that routes to backend services by URL path
- The full lifecycle: **containerize → push to registry → deploy → scale → expose via Ingress**

---

## What's Next

- Add **liveness and readiness probes** so Kubernetes can detect not just crashes but also hung processes
- Configure **resource requests and limits** (CPU/memory) to prevent a single pod from starving the node
- Set up a **Horizontal Pod Autoscaler (HPA)** to scale replicas automatically based on CPU usage — no manual `kubectl scale` needed
- Add multiple services with **path-based routing** (`/api`, `/admin`) through the Ingress to demonstrate real microservices traffic management

---

*Built as part of a hands-on Kubernetes learning portfolio — proving fault tolerance, scaling, and ingress routing with a live cluster you can break and watch heal itself.*
