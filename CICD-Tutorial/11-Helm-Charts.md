# 11 — Helm Charts: Kubernetes Package Management

> **Goal:** Understand Helm — how to package, version, and deploy Kubernetes applications like a pro.

---

## 🎯 What is Helm?

**Helm** is the **package manager for Kubernetes** — think of it as `apt` or `npm` for K8s. It bundles Kubernetes manifests into reusable, configurable packages called **Charts**.

### The Problem Helm Solves

```
Without Helm:
  To deploy payment-service, you need:
  - 1 Deployment YAML
  - 1 Service YAML
  - 1 Ingress YAML
  - 1 ConfigMap YAML
  - 1 Secret YAML
  - 1 HPA YAML
  - 1 NetworkPolicy YAML
  = 7 files to manage manually
  
  Different environments (dev, staging, prod)?
  = 21 files to maintain!
  
  Update one image tag across all environments?
  = Edit 3 files manually (error-prone!)

With Helm:
  helm install payment ./payment-chart -f values-prod.yaml
  = ONE command deploys everything
  = ONE chart, multiple value files for environments
  = ONE place to update image tags
```

---

## 📦 Helm Chart Structure

```
payment-chart/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default configuration values
├── values-dev.yaml     # Dev environment overrides
├── values-staging.yaml # Staging environment overrides
├── values-prod.yaml    # Production environment overrides
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── networkpolicy.yaml
│   └── _helpers.tpl    # Template helpers
└── .helmignore
```

### Chart.yaml
```yaml
apiVersion: v2
name: payment-service
description: Helm chart for the bank's payment processing service
type: application
version: 1.0.0      # Chart version
appVersion: "2.3.1"  # Application version
maintainers:
  - name: DevOps Team
    email: devops@bank.com
keywords:
  - banking
  - payment
  - upi
```

### values.yaml (Default Values)
```yaml
replicaCount: 2

image:
  repository: registry.bank.com/banking/payment-service
  pullPolicy: IfNotPresent
  tag: "2.3.1"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: api.bank.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### values-prod.yaml (Production Overrides)
```yaml
replicaCount: 6

image:
  tag: "2.3.1"

ingress:
  enabled: true
  host: api.bank.com
  tls:
    - secretName: bank-tls
      hosts:
        - api.bank.com

resources:
  limits:
    cpu: "1"
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 6
  maxReplicas: 30
  targetCPUUtilizationPercentage: 70

db:
  host: postgres.prod.bank.com
  port: 5432
```

### Template: deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payment.fullname" . }}
  labels:
    {{- include "payment.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "payment.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "payment.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: {{ .Values.db.host | quote }}
            - name: DB_PORT
              value: {{ .Values.db.port | quote }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 🔧 Essential Helm Commands

```bash
# Install a chart
helm install payment ./payment-chart -f values-prod.yaml -n production

# Upgrade to new version
helm upgrade payment ./payment-chart -f values-prod.yaml -n production

# Rollback to previous version
helm rollback payment 1 -n production

# List all releases
helm list -n production
# NAME      NAMESPACE   REVISION  STATUS     CHART               APP VERSION
# payment   production  3         deployed   payment-service-1.0  2.3.1

# View history
helm history payment -n production
# REVISION  STATUS     DESCRIPTION
# 1         superseded Initial install
# 2         superseded Upgrade to v2.3.0
# 3         deployed   Upgrade to v2.3.1

# Uninstall
helm uninstall payment -n production
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Environment-Specific Deployments
**Context:** Deploy the same payment service to dev, staging, and production with different configurations.

```bash
# Development
helm install payment ./payment-chart \
  -f values-dev.yaml \
  -n dev \
  --set image.tag=latest \
  --set replicaCount=1

# Staging
helm install payment ./payment-chart \
  -f values-staging.yaml \
  -n staging \
  --set image.tag=v2.3.0-rc1 \
  --set replicaCount=2

# Production
helm install payment ./payment-chart \
  -f values-prod.yaml \
  -n production \
  --set image.tag=v2.3.1 \
  --set replicaCount=6
```

**Same chart, different configurations:**
| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| Replicas | 1 | 2 | 6 |
| CPU Limit | 250m | 500m | 1 core |
| Memory Limit | 256Mi | 512Mi | 1Gi |
| Autoscaling | No | No | Yes (6-30) |
| Ingress | No | Yes | Yes (with TLS) |
| DB Host | local | staging-db | prod-db |

### Scenario 2: Canary Release with Helm
**Context:** Gradually roll out new version using Helm with Argo Rollouts.

```yaml
# values-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: {{ include "payment.fullname" . }}
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: {{ include "payment.fullname" . }}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
```

```bash
# Deploy canary
helm upgrade payment ./payment-chart \
  -f values-canary.yaml \
  --set image.tag=v2.4.0 \
  -n production

# Argo Rollouts manages the canary:
# 10% → Wait 5min → Check success rate → 
# 25% → Wait 10min → 50% → Wait 10min → 100%
```

### Scenario 3: Helm Chart Versioning for Compliance
**Context:** Regulators require exact version tracking of deployed software.

```bash
# Every deployment is tracked
$ helm history payment -n production
# REVISION  STATUS     DESCRIPTION                      CHART VERSION  APP VERSION
# 1         superseded Initial install                   1.0.0          2.2.0
# 2         superseded Add transaction limits           1.1.0          2.3.0
# 3         superseded Fix UPI timeout                  1.1.1          2.3.1
# 4         superseded Security patch CVE-2026-1234     1.1.2          2.3.2
# 5         deployed   Add GST calculation              1.2.0          2.4.0

# Audit query: "What was deployed on 2026-08-15?"
# Answer: Revision 4, App v2.3.2, Chart v1.1.2

# Rollback to specific revision if needed
$ helm rollback payment 3 -n production
# Rolled back to revision 3 (v2.3.1)
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Helm Chart for Payment Service Across Environments

**Context:** Deploy the same payment service to dev, staging, and production with different configurations.

```bash
# Chart structure
payment-chart/
├── Chart.yaml
├── values.yaml (defaults)
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    └── networkpolicy.yaml

# Deploy to Development
$ helm install payment-dev ./payment-chart \
    -f values-dev.yaml \
    -n dev \
    --set image.tag=latest
# NAME: payment-dev
# LAST DEPLOYED: Wed Sep  4 10:00:00 2026
# NAMESPACE: dev
# STATUS: deployed

# Deploy to Staging
$ helm install payment-staging ./payment-chart \
    -f values-staging.yaml \
    -n staging \
    --set image.tag=v2.4.0-rc1
# NAME: payment-staging
# LAST DEPLOYED: Wed Sep  4 11:00:00 2026
# NAMESPACE: staging
# STATUS: deployed

# Deploy to Production
$ helm install payment-prod ./payment-chart \
    -f values-prod.yaml \
    -n production \
    --set image.tag=v2.4.0
# NAME: payment-prod
# LAST DEPLOYED: Wed Sep  4 14:00:00 2026
# NAMESPACE: production
# STATUS: deployed

# Verify all environments
$ helm list --all-namespaces
# NAME              NAMESPACE    REVISION  STATUS     CHART               APP VERSION
# payment-dev       dev          1         deployed   payment-service-1.0  v2.4.0
# payment-staging   staging      1         deployed   payment-service-1.0  v2.4.0-rc1
# payment-prod      production   1         deployed   payment-service-1.0  v2.4.0

# Compare configurations
$ helm get values payment-dev -n dev
# image.tag: latest
# replicaCount: 1
# autoscaling.enabled: false

$ helm get values payment-prod -n production
# image.tag: v2.4.0
# replicaCount: 6
# autoscaling.enabled: true
```

### E2E Example 2: Helm Chart Rollback in Production

**Context:** New version causes errors; rollback to previous version.

```bash
# Deploy v2.4.0 (has bug)
$ helm upgrade --install payment ./payment-chart \
    -f values-prod.yaml \
    --set image.tag=v2.4.0 \
    -n production
# Release "payment" upgraded.

# Monitor: Error rate spikes to 2%
$ kubectl logs -l app=payment -n production --tail=50 | grep ERROR
# ERROR: NullPointerException in TransferService.java:142

# Rollback to v2.3.1 (previous working version)
$ helm rollback payment 1 -n production
# Rollback was a success! Happy hacking!

# Verify rollback
$ helm history payment -n production
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade to v2.4.0
# 3         deployed    Rollback to v2.3.1

# Monitor: Error rate returns to 0.001%
$ kubectl logs -l app=payment -n production --tail=50 | grep -c ERROR
# 0 errors ✅

# Total rollback time: 30 seconds
# Customer impact: 5 minutes (error spike duration)
# Transactions affected: ~250 (0.001% of daily volume)
```

### E2E Example 3: Helm Chart Versioning for Audit Trail

**Context:** Regulators require exact version tracking of all deployments.

```bash
# Complete deployment history
$ helm history payment -n production
# REVISION  STATUS      DESCRIPTION                      CHART VERSION  APP VERSION
# 1         superseded  Initial install                   1.0.0          2.2.0
# 2         superseded  Add transaction limits           1.1.0          2.3.0
# 3         superseded  Fix UPI timeout                  1.1.1          2.3.1
# 4         superseded  Security patch CVE-2026-1234     1.1.2          2.3.2
# 5         superseded  Add GST calculation              1.2.0          2.4.0
# 6         superseded  Fix memory leak                  1.2.1          2.4.1
# 7         deployed    Add instant UPI                  1.3.0          2.5.0

# Audit query: "What was deployed on 2026-08-15?"
$ helm history payment -n production --output json | jq '.[] | select(.status=="superseded")'
# Revision 4 was active on 2026-08-15
# App Version: 2.3.2
# Chart Version: 1.1.2
# Description: Security patch CVE-2026-1234

# Generate compliance report
$ helm list -n production -o json | jq '.[] | {name, revision, status, chart, appVersion, lastDeployed}'
# {
#   "name": "payment",
#   "revision": 7,
#   "status": "deployed",
#   "chart": "payment-service-1.3.0",
#   "appVersion": "2.5.0",
#   "lastDeployed": "2026-09-04T14:00:00Z"
# }
```

---

## 📋 Interview Questions

### Q1: What is the difference between Helm and kubectl?
**Answer:** 

`kubectl` applies raw Kubernetes manifests. Helm is a package manager that templates manifests, manages releases, handles rollbacks, and supports environment-specific configurations. 

Think of `kubectl apply` as installing software manually, while Helm is like using `apt` or `npm`. In banking, Helm is essential for managing multiple environments and tracking deployment history.

### Q2: How do Helm hooks work and when would you use them?
**Answer:** Helm hooks are Kubernetes resources that run at specific points in the release lifecycle. 

Common hooks: 

`pre-install` (run before install), 

`post-install` (run after install), 

`pre-upgrade` (run before upgrade). 

Use cases: 

(1) Database migrations — run as `pre-upgrade` hook. 

(2) Schema validation — run as `pre-install` hook. 

(3) Cleanup — run as `post-delete` hook.

### Q3: What is a Helm repository and how do you manage one?
**Answer:** A Helm repository is a collection of Helm charts, accessed via HTTP. Banks use private repositories (ChartMuseum, Harbor, or cloud-hosted). 

Management: 

(1) `helm repo add` — add a repository. 

(2) `helm search repo` — search for charts. 

(3) `helm pull` — download charts. 

(4) Version pinning — always specify exact chart versions in production.

### Q4: How do you handle secrets in Helm charts?
**Answer:** Never store secrets in `values.yaml` or chart templates. 

Use: (1) **External Secrets Operator** — sync from Vault/AWS Secrets Manager. 

(2) **Kubernetes Secrets** — reference existing secrets via `existingSecret`. 

(3) **Helm secrets plugin** — encrypt values files with SOPS. 

(4) **CI/CD variables** — inject secrets at deploy time. 

Example: `helm install payment chart --set db.password=$DB_PASSWORD` (from CI variable).

### Q5: What is the difference between Helm 2 and Helm 3?
**Answer:** Helm 2 required Tiller (a server-side component in the cluster) with cluster-admin privileges — a security risk. Helm 3 removed Tiller, making it client-side only. 

Key improvements: 

(1) No Tiller — simpler, more secure. 

(2) Native K8s auth — uses your kubeconfig permissions. 

(3) Secrets stored as K8s Secrets. 

(4) Improved rollbacks. 

All banking deployments should use Helm 3.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Helm | Package manager for Kubernetes |
| Chart | Bundled K8s manifests with templates |
| Values | Environment-specific configuration |
| Release | A deployed instance of a chart |
| Hooks | Run tasks at lifecycle points |
| Banking Relevance | Environment management, version tracking |

**Next:** [12-Infrastructure-as-Code.md](./12-Infrastructure-as-Code.md) — Learn Terraform and Ansible.
