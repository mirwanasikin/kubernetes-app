<div align="center">

# 🎓 Kubernetes Student Registry App

![Kubernetes](https://img.shields.io/badge/kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![k3s](https://img.shields.io/badge/k3s-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)
![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)
![License](https://img.shields.io/github/license/mirwanasikin/kubernetes-app?style=for-the-badge)

**GitOps manifests for a full-stack student registry app, deployed on a self-managed k3s cluster on AWS EC2.**

[Infrastructure Repo](https://github.com/mirwanasikin/aws-ec2-server) · [App Source Repo](https://github.com/mirwanasikin/flask-registry-app)

</div>

---

## 🗺️ Architecture Overview

This project is part of a 3-repository GitOps system:

```
┌──────────────────────────────────────────────────────────────┐
│                        GitHub                                │
│                                                              │
│  ┌─────────────────┐      ┌──────────────────────────────┐  │
│  │ flask-registry- │ push │     kubernetes-app           │  │
│  │ app             │─────▶│  (this repo — GitOps source) │  │
│  │ (app source +   │image │                              │  │
│  │  CI pipeline)   │      └──────────────┬───────────────┘  │
│  └─────────────────┘                     │ ArgoCD polls      │
│                                          │                   │
│  ┌─────────────────┐                     │                   │
│  │ aws-ec2-server  │                     │                   │
│  │ (OpenTofu +     │                     │                   │
│  │  Ansible)       │                     │                   │
│  └────────┬────────┘                     │                   │
└───────────┼──────────────────────────────┼───────────────────┘
            │ provisions                   │ syncs
            ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     AWS EC2 (k3s Cluster)                   │
│                                                             │
│  Control Plane ──┬── Worker Node 1 ──┬── Worker Node 2     │
│                  │                   │                      │
│             ┌────▼───────────────────▼────┐                │
│             │   Namespace: registry-       │                │
│             │   mahasiswa                  │                │
│             │                              │                │
│             │  ┌─────────┐  ┌──────────┐  │                │
│             │  │ Nginx   │  │  Flask   │  │                │
│             │  │Frontend │─▶│ Backend  │  │                │
│             │  └────┬────┘  └────┬─────┘  │                │
│             │       │            │         │                │
│             │  ┌────▼────────────▼─────┐  │                │
│             │  │   PostgreSQL (StatefulSet)│                │
│             │  └───────────────────────┘  │                │
│             └─────────────────────────────┘                │
│                                                             │
│  ArgoCD (namespace: argocd) ◀── polls this repo            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧰 Tech Stack

| Layer          | Technology         | Notes                         |
| -------------- | ------------------ | ----------------------------- |
| Orchestration  | k3s                | Lightweight Kubernetes on EC2 |
| GitOps CD      | ArgoCD             | Polls this repo, auto-syncs   |
| Frontend       | Nginx              | Ingress via Traefik           |
| Backend        | Flask (Python)     | REST API                      |
| Database       | PostgreSQL         | StatefulSet with PVC          |
| Secrets        | Sealed Secrets     | Encrypted at rest in Git      |
| Infrastructure | OpenTofu + Ansible | Managed in `aws-ec2-server`   |

---

## 📁 Repository Structure

```
.
├── app/
│   ├── backend/
│   │   ├── configmap.yaml       # Non-sensitive env vars (DB host, port, etc.)
│   │   ├── deployment.yaml      # Flask backend deployment
│   │   └── service.yaml
│   ├── database/
│   │   ├── sealed-secret.yaml   # Encrypted DB credentials (safe to commit)
│   │   ├── service.yaml
│   │   └── statefulset.yaml     # PostgreSQL with persistent volume
│   ├── frontend/
│   │   ├── deployment.yaml      # Nginx frontend deployment
│   │   ├── ingress.yaml         # Traefik ingress rules
│   │   ├── middleware.yaml      # Traefik middleware config
│   │   └── service.yaml
│   └── namespace.yaml           # registry-mahasiswa namespace
├── argocd/
│   └── application.yaml         # ArgoCD Application manifest
├── LICENSE
└── README.md
```

> [!NOTE]
> `pub-cert.pem` and `secret.yaml` are excluded from this repo intentionally.
> Only the encrypted `sealed-secret.yaml` is committed — that's the whole point of Sealed Secrets.

---

## ⚙️ Prerequisites

Before applying any manifests, make sure:

- [ ] Infrastructure is provisioned via [`aws-ec2-server`](https://github.com/mirwanasikin/aws-ec2-server) (OpenTofu + Ansible)
- [ ] k3s cluster is running (1 control plane + 2 worker nodes)
- [ ] ArgoCD is installed on the cluster (`kubectl get pods -n argocd`)
- [ ] Sealed Secrets controller is installed (`kubeseal --version`)
- [ ] Your `kubeconfig` is pointing to the right cluster

---

## 🚀 Deployment Flow

### 1. Create the namespace

```bash
kubectl apply -f app/namespace.yaml
```

### 2. Apply Sealed Secrets

The database credentials are encrypted using Sealed Secrets. Apply them first so the StatefulSet can mount them.

```bash
kubectl apply -f app/database/sealed-secret.yaml
```

### 3. Apply the ArgoCD Application

This tells ArgoCD to start watching this repo and sync everything under `app/`.

```bash
kubectl apply -f argocd/application.yaml
```

ArgoCD will then automatically apply the rest — backend, frontend, database — in the correct order.

> [!TIP]
> Access the ArgoCD UI by port-forwarding to your local machine:
>
> ```bash
> kubectl port-forward svc/argocd-server -n argocd 8080:443
> ```
>
> Then open `https://localhost:8080`

---

## 🔐 Secrets Management

Credentials are never stored in plaintext. The workflow is:

```
plaintext secret.yaml
       │
       ▼  kubeseal --cert pub-cert.pem
sealed-secret.yaml  ──▶  committed to Git  ──▶  ArgoCD applies it
                                                       │
                                          Sealed Secrets controller decrypts
                                                       │
                                                 Kubernetes Secret
```

The encryption is tied to the specific Sealed Secrets controller instance in the cluster, so even if someone gets the `sealed-secret.yaml`, it's useless outside the cluster.

---

## 🔗 Related Repositories

| Repository                                                               | Role                                                                    |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| [aws-ec2-server](https://github.com/mirwanasikin/aws-ec2-server)         | Provisions the EC2 cluster with OpenTofu and configures it with Ansible |
| [flask-registry-app](https://github.com/mirwanasikin/flask-registry-app) | Flask + Nginx app source code, CI pipeline pushes images to ghcr.io     |
| **kubernetes-app** _(this repo)_                                         | GitOps manifests, ArgoCD reads from here                                |

---

<div align="center">
<sub>Built with too much caffeine and a 2010 Toshiba laptop running NixOS.</sub>
</div>
