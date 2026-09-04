# 09 — Kubernetes Advanced: Production-Grade Banking Infrastructure

> **Goal:** Master advanced K8s concepts essential for banking CI/CD — namespaces, secrets, ConfigMaps, and resource management.

---

## 📑 Table of Contents

- [🏗️ Namespaces: Logical Isolation](#-namespaces-logical-isolation)
- [🔐 Secrets: Managing Sensitive Data](#-secrets-managing-sensitive-data)
- [📋 ConfigMaps: Application Configuration](#-configmaps-application-configuration)
- [💾 Persistent Volumes: Stateful Data](#-persistent-volumes-stateful-data)
- [🏦 Real-World Banking Scenarios](#-real-world-banking-scenarios)
  - [Scenario 1: Multi-Tenant Namespace Isolation](#scenario-1-multi-tenant-namespace-isolation)
  - [Scenario 2: Secret Rotation Without Downtime](#scenario-2-secret-rotation-without-downtime)
  - [Scenario 3: Resource Quotas and LimitRanges](#scenario-3-resource-quotas-and-limitranges)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Multi-Tenant Banking Namespace Setup](#e2e-example-1-multi-tenant-banking-namespace-setup)
  - [E2E Example 2: Secret Rotation Without Downtime](#e2e-example-2-secret-rotation-without-downtime)
  - [E2E Example 3: Resource Quota Enforcement](#e2e-example-3-resource-quota-enforcement)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🏗️ Namespaces: Logical Isolation

Namespaces **divide** a cluster into virtual sub-clusters for organization, access control, and resource isolation.

```
┌─────────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                     │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  dev         │  │  staging     │  │  production  │    │
│  │  namespace   │  │  namespace   │  │  namespace   │    │
│  │             │  │             │  │             │    │
│  │ payment:dev │  │ payment:stg │  │ payment:v2.3│    │
│  │ 2 replicas  │  │ 2 replicas  │  │ 6 replicas  │    │
│  │ 256Mi RAM   │  │ 512Mi RAM   │  │ 1Gi RAM     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐                       │
│  │  compliance  │  │  monitoring  │                       │
│  │  namespace   │  │  namespace   │                       │
│  │             │  │             │                       │
│  │ audit-svc   │  │ prometheus  │                       │
│  │ report-svc  │  │ grafana     │                       │
│  └─────────────┘  └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Deploy to specific namespace
kubectl apply -f payment-deployment.yaml -n production

# View resources by namespace
kubectl get pods --all-namespaces
# NAMESPACE     NAME                              STATUS
# dev           payment-service-abc123            Running
# staging       payment-service-def456            Running
# production    payment-service-ghi789            Running
```

---

## 🔐 Secrets: Managing Sensitive Data

```bash
# Create a secret from literal values
kubectl create secret generic db-secrets \
  --from-literal=username=bank_admin \
  --from-literal=password=S3cur3P@ssw0rd! \
  -n production

# Create a secret from a file
kubectl create secret tls bank-tls \
  --cert=certs/bank.com.crt \
  --key=certs/bank.com.key \
  -n production

# View secrets (encoded, not plain text)
kubectl get secrets -n production
# NAME              TYPE                AGE
# db-secrets        Opaque              5d
# bank-tls          kubernetes.io/tls   5d

# Decode a secret
kubectl get secret db-secrets -n production -o jsonpath='{.data.password}' | base64 -d
# S3cur3P@ssw0rd!
```

### Secret in Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        - name: payment
          env:
            # Reference secret as environment variable
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          volumeMounts:
            # Mount TLS certificate as file
            - name: tls-certs
              mountPath: /etc/ssl/certs
              readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: bank-tls
```

---

## 📋 ConfigMaps: Application Configuration

```bash
# Create ConfigMap
kubectl create configmap payment-config \
  --from-literal=DB_HOST=postgres.prod.bank.com \
  --from-literal=DB_PORT=5432 \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=FEATURE_FLAG_INSTANT_UPI=true \
  -n production
```

### ConfigMap in Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        - name: payment
          envFrom:
            - configMapRef:
                name: payment-config
          # All ConfigMap keys become environment variables
          # DB_HOST, DB_PORT, LOG_LEVEL, FEATURE_FLAG_INSTANT_UPI
```

---

## 💾 Persistent Volumes: Stateful Data

```yaml
# PersistentVolumeClaim - request storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: payment-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi

---
# StatefulSet - for stateful applications
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Multi-Tenant Namespace Isolation
**Context:** A bank operates in 3 countries (India, UAE, Singapore) with different regulations. Each country needs isolated resources.

```yaml
# Namespace per country
apiVersion: v1
kind: Namespace
metadata:
  name: india
  labels:
    region: india
    compliance: rbi
---
apiVersion: v1
kind: Namespace
metadata:
  name: uae
  labels:
    region: uae
    compliance: cbuae
---
apiVersion: v1
kind: Namespace
metadata:
  name: singapore
  labels:
    region: singapore
    compliance: mas
```

**Resource Quotas per Namespace:**
```yaml
# Resource quota for India (smaller deployment)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: india-quota
  namespace: india
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
---
# Resource quota for UAE (larger deployment)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: uae-quota
  namespace: uae
spec:
  hard:
    requests.cpu: "16"
    requests.memory: 32Gi
    limits.cpu: "32"
    limits.memory: 64Gi
    pods: "100"
    services: "40"
    persistentvolumeclaims: "20"
```

### Scenario 2: Secret Rotation Without Downtime
**Context:** Database password must be rotated every 90 days per PCI-DSS compliance.

**Automated Secret Rotation:**
```bash
# Step 1: Generate new password
$ NEW_PASSWORD=$(openssl rand -base64 32)

# Step 2: Update database with new password
$ psql -U admin -c "ALTER USER bank_user PASSWORD '$NEW_PASSWORD';"

# Step 3: Update Kubernetes secret
$ kubectl create secret generic db-secrets \
    --from-literal=username=bank_user \
    --from-literal=password="$NEW_PASSWORD" \
    --dry-run=client -o yaml | kubectl apply -f - -n production

# Step 4: Restart pods to pick up new secret
$ kubectl rollout restart deployment/payment-service -n production
$ kubectl rollout restart deployment/account-service -n production

# Step 5: Verify new secret is in use
$ kubectl exec payment-service-abc123 -- printenv DB_PASSWORD
# (new password) ✅
```

**Automated with External Secrets Operator:**
```yaml
# Syncs secrets from AWS Secrets Manager to K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secrets
  data:
    - secretKey: password
      remoteRef:
        key: bank/production/db-password
```

### Scenario 3: Resource Quotas and LimitRanges
**Context:** Prevent any single team from consuming all cluster resources.

```yaml
# LimitRange - default resource limits per container
apiVersion: v1
kind: LimitRange
metadata:
  name: banking-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      max:
        cpu: "2"
        memory: "4Gi"
      min:
        cpu: "100m"
        memory: "128Mi"
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Multi-Tenant Banking Namespace Setup

**Context:** Set up isolated namespaces for India, UAE, and Singapore banking operations.

```bash
# Create namespaces
$ kubectl create namespace india
$ kubectl create namespace uae
$ kubectl create namespace singapore

# Apply resource quotas
$ kubectl apply -f quotas-india.yaml -n india
$ kubectl apply -f quotas-uae.yaml -n uae
$ kubectl apply -f quotas-singapore.yaml -n singapore

# Deploy payment service to each region
$ helm upgrade --install payment ./helm/payment-chart \
    -f ./helm/values-india.yaml \
    -n india
$ helm upgrade --install payment ./helm/payment-chart \
    -f ./helm/values-uae.yaml \
    -n uae
$ helm upgrade --install payment ./helm/payment-chart \
    -f ./helm/values-singapore.yaml \
    -n singapore

# Verify isolation
$ kubectl get pods -n india -l app=payment
# NAME                    READY   STATUS    NODE
# payment-abc123          1/1     Running   node-india-01
# payment-def456          1/1     Running   node-india-02
# payment-ghi789          1/1     Running   node-india-03

$ kubectl get pods -n uae -l app=payment
# NAME                    READY   STATUS    NODE
# payment-jkl012          1/1     Running   node-uae-01
# payment-mno345          1/1     Running   node-uae-02

$ kubectl get pods -n singapore -l app=payment
# NAME                    READY   STATUS    NODE
# payment-pqr678          1/1     Running   node-sg-01
# payment-stu901          1/1     Running   node-sg-02

# Cross-namespace communication blocked by NetworkPolicy
$ kubectl exec -it payment-abc123 -n india -- curl -s payment-service.uae.svc.cluster.local
# Connection refused ✅ (Isolation working)
```

### E2E Example 2: Secret Rotation Without Downtime

**Context:** Rotate database credentials every 90 days per PCI-DSS.

```bash
# Step 1: Generate new credentials
$ NEW_PASSWORD=$(openssl rand -base64 32)
$ echo "New password generated: $(echo $NEW_PASSWORD | wc -c) bytes"

# Step 2: Update database
$ PGPASSWORD=$OLD_PASSWORD psql -h prod-db.bank.com -U admin -c \
    "ALTER USER payment_user PASSWORD '$NEW_PASSWORD';"
# ALTER ROLE

# Step 3: Update Kubernetes secret
$ kubectl create secret generic payment-secrets \
    --from-literal=db-password=$NEW_PASSWORD \
    --dry-run=client -o yaml | kubectl apply -f - -n production
# secret/payment-secrets configured

# Step 4: Rolling restart to pick up new secret
$ kubectl rollout restart deployment/payment-service -n production
# deployment.apps/payment-service restarted

# Step 5: Verify new secret is in use
$ kubectl get pods -l app=payment-service -n production
# NAME                          READY   STATUS    AGE
# payment-service-new-abc       1/1     Running   30s  (new pod)
# payment-service-old-def       1/1     Running   30s  (terminating)

$ kubectl exec -it payment-service-new-abc -n production -- printenv DB_PASSWORD
# (new password) ✅

# Step 6: Verify old password no longer works
$ PGPASSWORD=$OLD_PASSWORD psql -h prod-db.bank.com -U payment_user -c "SELECT 1;"
# psql: error: password authentication failed ✅

# Total downtime: 0 seconds
# Zero customer impact
# PCI-DSS compliance maintained
```

### E2E Example 3: Resource Quota Enforcement

**Context:** Prevent any team from consuming all cluster resources.

```bash
# Check current resource usage per namespace
$ kubectl describe resourcequota -n production
# Name: production-quota
# Resource                    Used    Hard
# --------                    ----    ----
# requests.cpu                8       16
# requests.memory             16Gi    32Gi
# limits.cpu                  12      32
# limits.memory               24Gi    64Gi
# pods                        18      50
# services                    12      20
# persistentvolumeclaims      5       10

# Try to deploy with excessive resources (should fail)
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: test
          image: nginx
          resources:
            requests:
              cpu: "20"  # Exceeds quota!
              memory: "40Gi"  # Exceeds quota!
EOF
# Error: exceeded quota: production-quota, requested: cpu=20,memory=40Gi,
#        used: cpu=8,memory=16Gi, limited: cpu=16,memory=32Gi
# Deployment blocked! ✅

# Apply with valid resources (should succeed)
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: test
          image: nginx
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
EOF
# deployment.apps/test-deployment created ✅
```

---

## 📋 Interview Questions

### Q1: What is the difference between a ConfigMap and a Secret?
**Answer:** ConfigMaps store **non-sensitive** configuration (feature flags, URLs, log levels). Secrets store **sensitive** data (passwords, API keys, certificates). Secrets are base64-encoded (not encrypted by default) and can be encrypted at rest using KMS. In banking, use ConfigMaps for application settings and Secrets (with encryption) for credentials.

### Q2: How do you implement network policies in Kubernetes for banking?
**Answer:** Network Policies control pod-to-pod communication. 

Default: all pods can talk to all pods. 

With Network Policies: 

(1) Payment service can only talk to database and Redis. 

(2) Database cannot initiate outbound connections. 

(3) Monitoring namespace can scrape all pods. 

This implements the **principle of least privilege** required by banking regulations.

### Q3: What is a DaemonSet and when would you use it in banking?
**Answer:** A DaemonSet ensures one pod runs on **every node** (or selected nodes). 

Use cases in banking: 

(1) **Log collection** — Fluentd/Filebeat on every node to collect logs. 

(2) **Security monitoring** — Falco for runtime threat detection. 

(3) **Node-level monitoring** — Prometheus node exporter. 

(4) **Compliance agents** — ensure every node has required security agents.

### Q4: How do you handle database migrations in Kubernetes?
**Answer:** Use a **Job** or **InitContainer** pattern: 

(1) Create a Kubernetes Job that runs the migration before the main application starts. 

(2) The migration Job uses the same database credentials as the application. 

(3) The Job runs Flyway or Liquibase to apply pending migrations. 

(4) If the migration fails, the Job fails and pods don't start. 

(5) For zero-downtime, use backward-compatible migrations (add column first, remove later).

### Q5: What is Pod Disruption Budget (PDB) and why is it important?
**Answer:** A PDB limits the number of pods that can be simultaneously disrupted (during voluntary disruptions like node drains, upgrades). 

Example: `minAvailable: 2` ensures at least 2 pods are always running. 

In banking, PDBs prevent: 

(1) All payment pods being evicted during cluster upgrade. 

(2) Database failover during maintenance. 

(3) Service degradation during node failures.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Namespaces | Logical isolation for environments/teams |
| Secrets | Sensitive data management with encryption |
| ConfigMaps | Non-sensitive application configuration |
| PersistentVolumes | Durable storage for stateful apps |
| Resource Quotas | Prevent resource abuse |
| Network Policies | Pod-to-pod communication control |
| Banking Relevance | Compliance isolation, secret management |

**Next:** [10-Pipeline-Tools.md](./10-Pipeline-Tools.md) — Learn Jenkins, GitLab CI, and ArgoCD.
