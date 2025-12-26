# 📘 Argo CD GitOps – Complete Hands-on Guide (Windows 11)

**Audience:** Solution Architects, Senior DevOps Engineers  
**Platform:** Windows 11 + WSL2 (Ubuntu)  
**Style:** Step-by-step, hands-on, production-oriented

---

## 🧭 Overview

This document is a **production-ready Argo CD GitOps handbook** designed for **Solution Architects and Platform Engineers**.

It goes beyond basic tutorials and focuses on:
- Enterprise GitOps design principles
- Scalable repository and environment strategies
- Security, governance, and auditability
- High availability and disaster recovery

This guide can be safely used as:
- Internal platform documentation
- Reference architecture
- Enablement material for DevOps teams

---


## 🧱 Target Architecture (Production-Aligned)

```
Git Provider (GitHub / Azure DevOps)
          │
          ▼
   Argo CD Control Plane (HA)
          │
   ┌──────┼─────────┐
   ▼      ▼         ▼
 Dev   QA / UAT    Prod Clusters
```

**Key Principles:**
- Git is the **single source of truth**
- Argo CD is the **continuous reconciler**, not a pipeline
- Kubernetes clusters are **immutable targets**

---


# 🔹 STEP 0 – Environment Setup

## 0.1 Install Required Tools (Windows 11)

### Install WSL2
```powershell
wsl --install
```
Restart and verify:
```powershell
wsl -l -v
```

### Update Ubuntu
```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker Desktop
- Enable **WSL2 backend**
- Enable **Ubuntu integration**

Verify inside WSL:
```bash
docker version
```

### Install kubectl
```bash
curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Install Kind
```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/kind
```

### Install Git
```bash
sudo apt install git -y
```

### Install Argo CD CLI
```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

---

## 0.2 Create Kubernetes Cluster

```bash
kind create cluster --name argocd-lab
kubectl get nodes
```

---

# 🔹 STEP 1 – Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify:
```bash
kubectl get pods -n argocd
```

Access UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

---

# 🔹 STEP 2 – First GitOps Application (Baseline Pattern)

## Purpose

This step establishes the **baseline GitOps contract**:
- No imperative deployments
- No environment-specific mutations
- All changes flow through Git

This pattern is mandatory before introducing Helm, Kustomize, or multi-environment setups.

## Git Repository Structure

```text
argocd-gitops-demo/
└── apps/
    └── nginx/
        ├── deployment.yaml
        └── service.yaml
```

### Design Notes
- One application per folder
- No templating at this stage
- Kubernetes-native YAML only

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Operational Expectations
- All deployments are reproducible
- Rollbacks are Git-based
- Drift is detectable

---


# 🔹 STEP 3 – Auto-Sync & Self-Healing

Enable in UI:
- Automated Sync
- Self Heal
- Prune

### Drift Test
```bash
kubectl scale deploy nginx --replicas=1
```

Argo CD restores desired state automatically.

---

# 🔹 STEP 4 – App-of-Apps Pattern (Enterprise Standard)

## Why App-of-Apps?

In production, **manual application creation does not scale**.

The App-of-Apps pattern ensures:
- Git-controlled onboarding
- Consistent environments
- Zero-click cluster recovery

## Repository Layout

```text
apps/
├── app-of-apps.yaml        # Root controller
├── dev/
│   └── nginx/
│       └── nginx-app.yaml
├── qa/
└── prod/
```

## Governance Rules
- Root app is created once
- All child apps are Git-managed
- No direct UI creation in production

---


# 🔹 STEP 5 – Helm & Kustomize (Environment Strategy)

## Decision Framework

| Requirement | Recommended Tool |
|------------|------------------|
Vendor / 3rd-party apps | Helm |
Internal microservices | Kustomize |
Strict env promotion | Kustomize |
Reusable packages | Helm |

---

## Helm with Argo CD

**Usage Guidelines:**
- Values must be environment-specific
- Charts must be version-pinned
- No Helm CLI usage in production

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: nginx
  targetRevision: 15.10.0
  helm:
    valueFiles:
      - values.yaml
```

---

## Kustomize with Argo CD

**Usage Guidelines:**
- Base contains no environment logic
- Overlays define replicas, resources, config
- No duplicated YAML

```text
base/
overlays/dev
overlays/prod
```

---


# 🔹 STEP 6 – Security, RBAC & SSO (Mandatory for Production)

## Security Model

```
Identity  →  SSO (OIDC / LDAP)
AuthZ     →  RBAC (Least Privilege)
Scope     →  Argo CD Projects
Secrets   →  External Secret Stores
```

## Non-Negotiable Rules
- Disable admin user
- No wildcard repo access in prod
- Read-only by default
- All access auditable

---

## Argo CD Projects

Projects enforce **blast-radius control**.

```yaml
kind: AppProject
spec:
  sourceRepos:
    - https://github.com/org/repo
  destinations:
    - namespace: dev
```

---

## RBAC Example

```yaml
p, role:dev-readonly, applications, get, dev/*, allow
p, role:dev-readonly, applications, sync, dev/*, deny
```

---

## SSO (OIDC)

Supported providers:
- Azure AD
- Okta
- Keycloak

Ingress + TLS is mandatory for SSO.

---


# 🔹 STEP 7 – Multi-Cluster, HA & DR (Platform Architecture)

## Multi-Cluster Strategy

- One Argo CD per organization
- Separate clusters per environment
- Centralized policy, decentralized execution

```bash
argocd cluster add <context>
```

---

## High Availability

HA is mandatory when:
- Argo CD controls production
- Compliance is required
- Downtime is unacceptable

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

---

## Disaster Recovery Model

| Component | Recovery Method |
|---------|-----------------|
Git | Native backup |
Argo CD | Reinstall + sync |
Secrets | External store |
Clusters | IaC rebuild |

---

## Promotion Strategy

Recommended:
- Folder-based environments
- PR-based promotion
- Mandatory reviews for prod

---


# 🏁 Final Architect Checklist (Production Readiness)

- [x] Git is the single source of truth
- [x] No kubectl apply in production
- [x] Auto-sync + self-heal enabled
- [x] App-of-Apps enforced
- [x] Helm/Kustomize strategy defined
- [x] RBAC & SSO enabled
- [x] Secrets externalized
- [x] Multi-cluster support
- [x] HA enabled for control plane
- [x] DR strategy documented

---

## 🏆 Conclusion

This document represents a **production-grade GitOps reference architecture** using Argo CD.

Teams following this guide can:
- Standardize deployments
- Eliminate configuration drift
- Improve auditability and compliance
- Scale safely across environments and clusters

**End of Production Handbook** 📕
