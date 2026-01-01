# Installation & Bootstrap Guide

This document describes the **end-to-end bootstrap flow** for deploying the GitOps
control plane and application platform using **Argo CD and ApplicationSets**.

It assumes a **pre-existing Kubernetes cluster** and focuses exclusively on the
**GitOps control plane and reconciliation model**.

---

## Prerequisites

- Kubernetes cluster (tested with **AWS EKS**)
- `kubectl` configured with **cluster-admin** access
- Git
- (Optional) `argocd` CLI for local inspection
- Valid DNS domain ownership and TLS certificate available (ACM or equivalent)

⚠️ **Mandatory precondition**

This repository contains **example domains and certificate references**
owned by the author.

Before proceeding, you **must** update all domain- and TLS-related values
to match your own environment.

See:
**Domain & TLS Configuration Guide**  
→ [`docs/domain-configuration.md`](docs/domain-configuration.md)

---

## 1. Clone the Repository

Clone the GitOps repository locally:

```bash
git clone https://github.com/LaurisNeimanis/aws-eks-gitops.git
cd aws-eks-gitops
```

This repository is the **single source of truth** for platform and workload state.

---

## 2. Install Argo CD (Bootstrap Layer)

Argo CD is treated as **control-plane tooling** and is intentionally
**not managed by this GitOps repository**.

It is installed once using a **Kustomize-based bootstrap**:

```bash
kubectl apply -k bootstrap/argocd
```

Verify that all Argo CD components are running:

```bash
kubectl get pods -n argocd
```

Wait until all pods are in `Running` state before proceeding.

---

## 3. (Optional) Access Argo CD UI & CLI

This step is optional and intended for **local inspection or demos**.

### Port-forward Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the UI at:

```
https://localhost:8080
```

### Login Credentials

**Username**
```
admin
```

**Retrieve initial admin password**
```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### (Optional) CLI Login

```bash
argocd login localhost:8080
```

---

## 4. Install Argo CD Projects (Required)

Before registering any applications, **Argo CD Projects must exist**.
They define security and ownership boundaries.

Apply the projects configuration:

```bash
kubectl apply -k argo/projects
```

This creates:
- `bootstrap` – root App-of-Apps
- `platform` – cluster-level services
- `workloads` – application workloads

This step is **mandatory**.

---

## 5. Bootstrap the GitOps Root Application

Apply the root application using the **App-of-Apps** pattern:

```bash
kubectl apply -f argo/root-application.yaml
```

From this point onward:

- Argo CD becomes the **authoritative reconciliation engine**
- Platform services and workloads are deployed automatically
- All managed namespaces are controlled via Git

No further manual `kubectl apply` is required.

---

## 6. Day-2 Operations Model

After bootstrap:

- All changes flow through **Git commits**
- Argo CD enforces:
  - automated sync
  - self-healing
  - deterministic ordering
- Direct `kubectl` changes in managed namespaces should be avoided

---

## Design Rationale

- **Clear separation** between control plane and GitOps payload
- **No circular dependencies**
- **Deterministic bootstrap**
- **Security-first** project boundaries
- **Portable GitOps structure** reusable across clusters

---

## Summary

After completing these steps:

- Argo CD operates as the **control plane**
- Platform and workloads are fully GitOps-managed
- Reconciliation is automated and self-healing
- All future changes are applied declaratively via Git

This bootstrap flow is intentionally minimal, explicit, and production-aligned.
