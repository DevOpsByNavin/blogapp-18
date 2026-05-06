# blogapp-18 — Kubernetes CI/CD Version (based on blogapp-17)

This repository is the **Kubernetes (k8s)** version of my earlier project **[blogapp-17](https://github.com/DevOpsByNavin/blogapp-17)**.

In **blogapp-17**, the application is deployed using **Docker Compose** on a server (via Jenkins pipeline + Harbor).  
In **blogapp-18**, the goal is to achieve the same outcome—but deploy the application to a **Kubernetes cluster** (manual `kubeadm` setup) and manage application configuration using **HashiCorp Vault**.

---

## What this project does

### High-level workflow

1. Developer pushes code to GitHub.
2. Jenkins pipeline runs:
  - Builds Docker images for:
     - `backend1`
     - `backend2`
     - `nginx` (frontend build + reverse proxy)
  - Pushes images to a container registry (currently **DockerHub**).
3. Jenkins connects to the Kubernetes node through SSH and deploys the workloads using `kubectl`.
4. Application becomes accessible through an Ingress (Traefik).

---

## What’s different from blogapp-17?

✅ **blogapp-17**
- Deployment target: **Docker Compose**
- Runtime: Containers on a VM (EC2/VPS)
- Env/config: `.env` files stored on server
- Registry: **Harbor**
- Deployment method: `docker compose up -d` via SSH

✅ **blogapp-18**
- Deployment target: **Kubernetes**
- Runtime: Pods on a kubeadm cluster (control-plane + worker nodes)
- Env/config: pulled from **HashiCorp Vault**, then applied as Kubernetes ConfigMaps (initial approach)
- Registry: **DockerHub** (for now)
- Deployment method: `kubectl apply -f k8s/` via SSH
- Traffic: **Traefik Ingress Controller** + Kubernetes Ingress object

So you can think of **blogapp-18 as “blogapp-17 + Kubernetes + Vault”**.

---

## Repository structure

```bash
.
├── infra
│   ├── database
│   │   └── init.sql
│   └── nginx
│       ├── Dockerfile
│       └── nginx.conf
├── k8s
│   ├── namespace.yaml
│   ├── ingress.yaml
│   ├── backend1-deployment.yaml
│   ├── backend1-service.yaml
│   ├── backend2-deployment.yaml
│   ├── backend2-service.yaml
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   ├── postgres-pv.yaml
│   ├── postgres-service.yaml
│   └── postgres-statefulset.yaml
├── services
│   ├── backend1
│   ├── backend2
│   └── frontend
└── Jenkinsfile
```

---

## Tools / Tech used

- **Jenkins** (CI/CD)
- **Docker** (build and push images)
- **DockerHub** (container registry for now)
- **Kubernetes (kubeadm)** (deployment target)
- **Traefik** (Ingress Controller)
- **HashiCorp Vault** (source of application configs / environment values)
- **PostgreSQL** (database)

---

## Deployment notes (quick start)

This repo assumes you already have:

1. A working Kubernetes cluster (manual `kubeadm` setup is what I’m using)
2. Traefik installed as an ingress controller
3. Jenkins configured with:
   - Docker installed + permission to build/push images
   - Vault plugin configured
   - Credentials for:
     - DockerHub
     - SSH into Kubernetes node
     - Vault AppRole / Vault credentials

The Jenkins pipeline:
- builds and pushes images
- then copies the `k8s/` directory + database init script to the server
- creates configmaps from Vault secrets
- applies all Kubernetes manifests with `kubectl`

---

## Related project

- **blogapp-17 (Docker Compose version):**
  https://github.com/DevOpsByNavin/blogapp-17

If you want a more complete explanation of the original setup and flow, refer to the blogapp-17 README first, then come back here for the Kubernetes version.

---

## Status

This project is actively evolving. The goal is to reach feature-parity with **blogapp-17** while using Kubernetes-native deployment patterns.