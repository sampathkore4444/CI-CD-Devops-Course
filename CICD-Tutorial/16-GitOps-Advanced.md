# 16 — Advanced GitOps: Declarative Infrastructure & Deployments

> **Goal:** Master GitOps beyond basics — pull-based deployments, reconciliation, and multi-cluster management.

---

## 🔍 What is GitOps (Advanced)?

GitOps is more than "use Git for deployments." It's a **complete operational framework** where Git is the single source of truth for everything — infrastructure, applications, and configuration.

### Core GitOps Principles

```
1. DECLARATIVE    → Describe desired state, not how to get there
2. VERSIONED      → Every change is in Git (immutable history)
3. AUTOMATED      → Agents automatically reconcile state
4. SELF-HEALING   → Detect drift and correct it automatically
```

### Push vs Pull Based Deployments

```
PUSH-BASED (Jenkins, GitLab CI):
┌─────────┐         ┌─────────────────┐
│  CI/CD  │ ──────▶ │ K8s Cluster     │
│  Server │  kubectl│                 │
│         │  apply  │  ┌───────────┐  │
│         │         │  │ Pods      │  │
└─────────┘         │  └───────────┘  │
                    └─────────────────┘
Problem: CI server needs cluster credentials
Risk: If CI server is compromised, attacker gets cluster access

PULL-BASED (ArgoCD, Flux):
┌─────────┐         ┌─────────────────┐
│  Git    │ ◀────── │ K8s Cluster     │
│  Repo   │  watch  │                 │
│         │         │  ┌───────────┐  │
│         │         │  │ ArgoCD    │  │
│         │         │  │ (inside   │  │
│         │         │  │  cluster) │  │
│         │         │  └───────────┘  │
└─────────┘         └─────────────────┘
Benefit: Cluster pulls from Git (no inbound access needed)
More secure for banking environments
```

---

## 🏗️ ArgoCD Architecture for Banking

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARGOCD ARCHITECTURE                                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    GIT REPOSITORY                             │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │  │
│  │  │ apps/         │  │ clusters/    │  │ policies/    │      │  │
│  │  │ payment/      │  │ mumbai/      │  │ network/     │      │  │
│  │  │ account/      │  │ singapore/   │  │ rbac/        │      │  │
│  │  │ gateway/      │  │ delhi/       │  │ compliance/  │      │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    ARGOCD SERVER                               │  │
│  │                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │ App Controller│  │ Repo Server │  │   API Server │         │  │
│  │  │ (watches Git)│  │ (clones)    │  │  (Web UI)    │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                    ┌─────────┴─────────┐                           │
│                    ▼                   ▼                           │
│  ┌─────────────────────┐  ┌─────────────────────┐                │
│  │   Mumbai Cluster     │  │  Singapore Cluster    │                │
│  │   (Production)       │  │  (DR)                 │                │
│  │                      │  │                       │                │
│  │  ┌───────────────┐  │  │  ┌───────────────┐   │                │
│  │  │ payment: v2.4 │  │  │  │ payment: v2.4 │   │                │
│  │  │ account: v1.8 │  │  │  │ account: v1.8 │   │                │
│  │  │ gateway: v3.1 │  │  │  │ gateway: v3.1 │   │                │
│  │  └───────────────┘  │  │  └───────────────┘   │                │
│  └─────────────────────┘  └─────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 GitOps Repository Structure

```
gitops-repo/
├── apps/                          # Application manifests
│   ├── payment-service/
│   │   ├── base/                  # Base manifests
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/              # Environment-specific
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/
│   │       ├── staging/
│   │       └── production/
│   ├── account-service/
│   └── gateway-service/
│
├── clusters/                      # Cluster-specific config
│   ├── mumbai/
│   │   ├── argocd-apps.yaml
│   │   ├── network-policies.yaml
│   │   └── resource-quotas.yaml
│   ├── singapore/
│   └── delhi/
│
├── policies/                      # Security & compliance
│   ├── rbac/
│   ├── network-policies/
│   └── pod-security-policies/
│
└── environments/                  # Environment definitions
    ├── dev.yaml
    ├── staging.yaml
    └── production.yaml
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Multi-Cluster GitOps Deployment

**Context:** Deploy payment service to 3 clusters using ArgoCD with different configurations.

```yaml
# ArgoCD Application for each cluster
# File: clusters/mumbai/payment-app.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-mumbai
  namespace: argocd
  labels:
    cluster: mumbai
    environment: production
    team: payments
spec:
  project: banking
  source:
    repoURL: https://git.bank.com/gitops/repo.git
    targetRevision: main
    path: apps/payment-service/overlays/production-mumbai
  destination:
    server: https://mumbai-k8s.bank.com
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Delete resources removed from Git
      selfHeal: true     # Revert manual changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  # Health checks
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Allow HPA to manage replicas
```

```bash
# Deploy to all clusters
$ kubectl apply -f clusters/mumbai/payment-app.yaml
$ kubectl apply -f clusters/singapore/payment-app.yaml
$ kubectl apply -f clusters/delhi/payment-app.yaml

# Verify sync status
$ argocd app list
# NAME                       CLUSTER                        STATUS    HEALTH
# payment-service-mumbai     https://mumbai-k8s.bank.com    Synced    Healthy
# payment-service-singapore  https://sg-k8s.bank.com         Synced    Healthy
# payment-service-delhi      https://delhi-k8s.bank.com     Synced    Healthy

# Update all clusters (just push to Git!)
$ cd gitops-repo
$ sed -i 's|payment:v2.4.0|payment:v2.5.0|' apps/payment-service/overlays/production-mumbai/kustomization.yaml
$ sed -i 's|payment:v2.4.0|payment:v2.5.0|' apps/payment-service/overlays/production-singapore/kustomization.yaml
$ sed -i 's|payment:v2.4.0|payment:v2.5.0|' apps/payment-service/overlays/production-delhi/kustomization.yaml
$ git commit -m "chore: update payment service to v2.5.0"
$ git push origin main

# ArgoCD detects change within 3 minutes (default polling)
# Auto-syncs to all 3 clusters
# Zero manual intervention
```

### E2E Example 2: GitOps Rollback

**Context:** New deployment causes errors; rollback using Git revert.

```bash
# 10:00 AM - Deploy v2.5.0 (has bug)
$ git commit -m "chore: update payment to v2.5.0"
$ git push origin main
# ArgoCD syncs to all clusters

# 10:15 AM - Errors detected
$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~"5..",service="payment"}[5m])'
# Error rate: 2.5% (threshold: 0.1%) ❌

# Rollback via Git (standard GitOps way)
$ git revert HEAD
$ git commit -m "revert: payment v2.5.0 due to error rate spike"
$ git push origin main

# ArgoCD detects revert
# Syncs v2.4.0 back to all clusters
# Error rate: 0.001% (normalized) ✅

# Alternative: ArgoCD CLI rollback (emergency)
$ argocd app rollback payment-service-mumbai 1
# Rolled back to revision 1 (v2.4.0)

# Post-mortem
$ git log --oneline -5
# a1b2c3d revert: payment v2.5.0 due to error rate spike
# d4e5f6g chore: update payment to v2.5.0
# h7i8j9k feat: add instant UPI feature
# l0m1n2o fix: resolve NEFT timeout
# p3q4r5s chore: update security certificates
```

### E2E Example 3: GitOps with Kustomize

**Context:** Use Kustomize for environment-specific configurations without Helm.

```yaml
# Base deployment
# File: apps/payment-service/base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment
          image: registry.bank.com/payment:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

```yaml
# Production overlay
# File: apps/payment-service/overlays/production/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: production

commonLabels:
  environment: production
  team: payments

patches:
  - target:
      kind: Deployment
      name: payment-service
    patch: |
      - op: replace
        path: /spec/replicas
        value: 6
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "512Mi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "500m"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "1Gi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu
        value: "1000m"

images:
  - name: registry.bank.com/payment
    newTag: v2.5.0
```

```bash
# Generate final manifest
$ kustomize build apps/payment-service/overlays/production
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   name: payment-service
#   namespace: production
#   labels:
#     environment: production
#     team: payments
# spec:
#   replicas: 6
#   ...
#     containers:
#       - name: payment
#         image: registry.bank.com/payment:v2.5.0
#         resources:
#           requests:
#             memory: "512Mi"
#             cpu: "500m"
#           limits:
#             memory: "1Gi"
#             cpu: "1000m"
```

---

## 📋 Interview Questions

### Q1: What is the difference between GitOps and traditional CI/CD?
**Answer:** Traditional CI/CD uses **push-based** deployment (CI server pushes to cluster). GitOps uses **pull-based** deployment (ArgoCD/Flux inside cluster pulls from Git). GitOps benefits: (1) **Security** — cluster doesn't expose credentials to CI server. (2) **Auditability** — Git is the single source of truth. (3) **Self-healing** — ArgoCD detects and corrects drift. (4) **Rollback** — revert Git commit, ArgoCD auto-syncs.

### Q2: How does ArgoCD handle drift detection and self-healing?
**Answer:** ArgoCD continuously compares desired state (Git) with actual state (cluster). If it detects drift (manual kubectl changes, HPA scaling), it automatically corrects it by syncing back to Git state. For banking, this ensures compliance — no unauthorized changes persist.例外: HPA-managed replica counts can be ignored via `ignoreDifferences`.

### Q3: What is the difference between Kustomize and Helm?
**Answer:** **Helm** uses templates with `{{ .Values.x }}` syntax. **Kustomize** uses overlays and patches without templates. Helm is better for: complex templating, reusable charts, versioned packages. Kustomize is better for: simple overlays, native K8s manifests, no template syntax. Banks often use both: Helm for third-party apps, Kustomize for internal services.

### Q4: How do you implement GitOps for disaster recovery?
**Answer:** (1) **Multi-cluster ArgoCD** — same Git repo, different cluster destinations. (2) **Cluster add-on management** — ArgoCD manages monitoring, logging, security tools on each cluster. (3) **Automated failover** — DNS switch + ArgoCD syncs DR cluster. (4) **Backup** — Velero backs up cluster state; ArgoCD restores from Git.

### Q5: What are GitOps best practices for regulated banking?
**Answer:** (1) **Branch protection** — main branch requires PR review. (2) **Signed commits** — verify author identity. (3) **Encrypted secrets** — SOPS or Vault for sensitive data. (4) **Multi-approval** — production changes need 2+ reviewers. (5) **Automated compliance** — policy enforcement via OPA/Gatekeeper. (6) **Immutable history** — no force pushes, no rebasing main.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| GitOps Principles | Declarative, versioned, automated, self-healing |
| Pull-based | More secure than push-based CI/CD |
| ArgoCD | Kubernetes-native GitOps controller |
| Kustomize | Template-free overlays for manifests |
| Drift Detection | Automatic correction of manual changes |
| Banking Relevance | Security, compliance, auditability |

**Next:** [17-Service-Mesh.md](./17-Service-Mesh.md) — Learn Istio for microservices security and observability.
