# 08 — Kubernetes Basics: Container Orchestration

> **Goal:** Understand Kubernetes — how it manages containers at scale in banking environments.

---

## 🔍 What is Kubernetes?

**Kubernetes** (K8s) is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.

### The Problem Kubernetes Solves

```
Without Kubernetes:
  "Deploy payment-service to 50 servers"
  → Manual SSH to each server
  → Manual Docker run on each
  → Manual health checks
  → What if a server crashes?
  → Manual failover
  → Scaling? Add more servers manually
  → This takes DAYS

With Kubernetes:
  kubectl apply -f payment-deployment.yaml
  → Automatically deployed to 50 servers
  → Automatic health checks
  → Automatic restart if container crashes
  → Automatic failover
  → Scale to 100 replicas: kubectl scale --replicas=100
  → This takes SECONDS
```

---

## 🏗️ Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     CONTROL PLANE                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │ API      │  │ etcd     │  │ Scheduler│  │Controller│   │   │
│  │  │ Server   │  │(Database)│  │          │  │ Manager  │   │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌───────────────────────┐  ┌───────────────────────┐              │
│  │    WORKER NODE 1      │  │    WORKER NODE 2      │              │
│  │  ┌─────────────────┐  │  │  ┌─────────────────┐  │              │
│  │  │ Payment Pod     │  │  │  │ Payment Pod     │  │              │
│  │  │ ┌─────────────┐ │  │  │  │ ┌─────────────┐ │  │              │
│  │  │ │   Container │ │  │  │  │ │   Container │ │  │              │
│  │  │ └─────────────┘ │  │  │  │ └─────────────┘ │  │              │
│  │  └─────────────────┘  │  │  └─────────────────┘  │              │
│  │  ┌─────────────────┐  │  │  ┌─────────────────┐  │              │
│  │  │ Account Pod     │  │  │  │ Notify Pod      │  │              │
│  │  │ ┌─────────────┐ │  │  │  │ ┌─────────────┐ │  │              │
│  │  │ │   Container │ │  │  │  │ │   Container │ │  │              │
│  │  │ └─────────────┘ │  │  │  │ └─────────────┘ │  │              │
│  │  └─────────────────┘  │  │  └─────────────────┘  │              │
│  │  ┌─────────────────┐  │  │  ┌─────────────────┐  │              │
│  │  │ kubelet         │  │  │  │ kubelet         │  │              │
│  │  │ kube-proxy      │  │  │  │ kube-proxy      │  │              │
│  │  └─────────────────┘  │  │  └─────────────────┘  │              │
│  └───────────────────────┘  └───────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Analogy |
|-----------|------|---------|
| **API Server** | Entry point for all commands | Receptionist |
| **etcd** | Distributed key-value store (cluster state) | Filing cabinet |
| **Scheduler** | Assigns pods to nodes | Job assignment manager |
| **Controller Manager** | Maintains desired state | Quality inspector |
| **kubelet** | Agent on each node, manages pods | Floor supervisor |
| **kube-proxy** | Network routing on each node | Traffic controller |

---

## 📋 Core Kubernetes Objects

### 1. Pod — The Smallest Unit
```yaml
# payment-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service-pod
  labels:
    app: payment-service
spec:
  containers:
    - name: payment
      image: registry.bank.com/banking/payment-service:v2.3.1
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

### 2. Deployment — Manages Pods
```yaml
# payment-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: banking
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment
          image: registry.bank.com/banking/payment-service:v2.3.1
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
```

### 3. Service — Expose Pods
```yaml
# payment-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: banking
spec:
  selector:
    app: payment-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### 4. Ingress — External Access
```yaml
# payment-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ingress
  namespace: banking
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
    - hosts:
        - api.bank.com
      secretName: bank-tls-cert
  rules:
    - host: api.bank.com
      http:
        paths:
          - path: /api/v1/payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

---

## 🔧 Essential kubectl Commands

```bash
# View cluster info
kubectl cluster-info

# List all pods
kubectl get pods -n banking
# NAME                              READY   STATUS    AGE
# payment-service-abc123-def456     1/1     Running   5d
# payment-service-ghi789-jkl012     1/1     Running   5d
# payment-service-mno345-pqr678     1/1     Running   5d

# View pod details
kubectl describe pod payment-service-abc123-def456 -n banking

# View logs
kubectl logs -f payment-service-abc123-def456 -n banking

# Deploy application
kubectl apply -f payment-deployment.yaml

# Scale deployment
kubectl scale deployment payment-service --replicas=5 -n banking

# Rollout status
kubectl rollout status deployment/payment-service -n banking

# Rollback
kubectl rollout undo deployment/payment-service -n banking

# Delete
kubectl delete -f payment-deployment.yaml
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Zero-Downtime Deployment
**Context:** Deploy new payment service version without any interruption to customer transactions.

**Rolling Update in Action:**
```bash
# Current state: 3 replicas running v2.3.0
$ kubectl get pods -l app=payment-service -n banking
NAME                              READY   STATUS    AGE
payment-service-abc123            1/1     Running   30d    # v2.3.0
payment-service-def456            1/1     Running   30d    # v2.3.0
payment-service-ghi789            1/1     Running   30d    # v2.3.0

# Deploy v2.3.1 with rolling update
$ kubectl set image deployment/payment-service \
    payment=registry.bank.com/banking/payment-service:v2.3.1 \
    -n banking

# Kubernetes automatically:
# 1. Creates new pod with v2.3.1
# 2. Waits for it to be ready (health check passes)
# 3. Terminates one old pod
# 4. Creates another new pod
# 5. Repeats until all pods are v2.3.1

# During transition (mixed versions):
NAME                              READY   STATUS
payment-service-abc123            1/1     Running    # v2.3.0 (old)
payment-service-xyz789            1/1     Running    # v2.3.1 (new)
payment-service-ghi789            1/1     Running    # v2.3.0 (old)

# After completion:
NAME                              READY   STATUS
payment-service-xyz789            1/1     Running    # v2.3.1
payment-service-jkl012            1/1     Running    # v2.3.1
payment-service-mno345            1/1     Running    # v2.3.1

# Zero downtime, zero customer impact ✅
```

### Scenario 2: Auto-Scaling During Peak Hours
**Context:** Payment service needs to handle 10x load during salary day (1st of month).

**Horizontal Pod Autoscaler (HPA):**
```yaml
# payment-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: banking
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

**Scaling Timeline:**
```
00:00 - Normal traffic: 3 replicas
01:00 - Salary processing begins: 3 → 8 replicas (auto-scale up)
02:00 - Peak load: 8 → 25 replicas (auto-scale up)
03:00 - Load decreases: 25 → 15 replicas (auto-scale down)
04:00 - Normal traffic: 15 → 3 replicas (auto-scale down)
```

### Scenario 3: Self-Healing and Health Checks
**Context:** A payment service pod crashes due to memory leak. Kubernetes automatically restarts it.

**Health Check Configuration:**
```yaml
# Liveness: Is the container alive?
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
  # If fails 3 times → restart container

# Readiness: Can the container serve traffic?
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3
  # If fails 3 times → remove from service endpoints
```

**Self-Healing Flow:**
```bash
# Pod crashes at 02:15 AM
# Kubernetes detects via liveness probe failure

[02:15:00] Liveness probe failed (attempt 1/3)
[02:15:10] Liveness probe failed (attempt 2/3)
[02:15:20] Liveness probe failed (attempt 3/3)
[02:15:20] Container terminated (OOMKilled)
[02:15:21] kubelet starting new container...
[02:15:25] New container running
[02:15:40] Liveness probe: healthy ✅
[02:15:45] Readiness probe: ready ✅
[02:15:45] Pod added back to service

# Total downtime: 25 seconds (vs hours if manual intervention needed)
# No human woke up at 2 AM ✅
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Payment Service Kubernetes Deployment

**Context:** Deploy payment service to production Kubernetes cluster with high availability.

```yaml
# Complete deployment manifest
# File: payment-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  labels:
    app: payment-service
    version: v2.4.0
    team: payments
spec:
  replicas: 6
  selector:
    matchLabels:
      app: payment-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payment-service
        version: v2.4.0
    spec:
      containers:
        - name: payment
          image: registry.bank.com/prod/payment:v2.4.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: db-password
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    app: payment-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
    - hosts:
        - api.bank.com
      secretName: bank-tls
  rules:
    - host: api.bank.com
      http:
        paths:
          - path: /api/v1/payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

```bash
# Deploy to production
$ kubectl apply -f payment-deployment.yaml -n production
# deployment.apps/payment-service created
# service/payment-service created
# ingress.networking.k8s.io/payment-ingress created

# Verify deployment
$ kubectl get pods -l app=payment-service -n production
# NAME                          READY   STATUS    AGE
# payment-service-abc123-xyz1   1/1     Running   30s
# payment-service-def456-xyz2   1/1     Running   30s
# payment-service-ghi789-xyz3   1/1     Running   30s
# payment-service-jkl012-xyz4   1/1     Running   30s
# payment-service-mno345-xyz5   1/1     Running   30s
# payment-service-pqr678-xyz6   1/1     Running   30s

# Test service
$ kubectl exec -it payment-service-abc123-xyz1 -n production -- curl -s http://localhost:8080/actuator/health
# {"status":"UP","components":{"db":{"status":"UP"},"redis":{"status":"UP"}}}

# Check ingress
$ curl -k https://api.bank.com/api/v1/payments/health
# {"status":"healthy","version":"2.4.0"}
```

### E2E Example 2: Auto-Scaling Payment Service During Salary Day

**Context:** Payment service must handle 10x load on salary day (1st of month).

```yaml
# HPA configuration
# File: payment-hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 6
  maxReplicas: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
          name: memory
          target:
            type: Utilization
            averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 10
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
```

```bash
# Salary day morning: Load increases
$ kubectl get hpa payment-hpa -n production
# NAME            REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS
# payment-hpa     Deployment/payment-service    65%/70%, 72%/80%   6         60        6

# 10:00 AM: Peak load starts
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    85%/70%, 88%/80%   6         60        12
# Pods scaling up...

# 10:05 AM: High load continues
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    78%/70%, 82%/80%   6         60        25

# 10:15 AM: Peak load
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    68%/70%, 75%/80%   6         60        45

# 2:00 PM: Load decreases
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    45%/70%, 52%/80%   6         60        30

# 6:00 PM: Normal load
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    35%/70%, 42%/80%   6         60        8

# 10:00 PM: Low load
$ kubectl get hpa payment-hpa -n production
# payment-hpa     Deployment/payment-service    25%/70%, 30%/80%   6         60        6
```

### E2E Example 3: Zero-Downtime Deployment with Health Checks

**Context:** Deploy new payment version without any transaction failure.

```bash
# Current state: 6 pods running v2.3.0
$ kubectl get pods -l app=payment-service -n production
# NAME                          READY   STATUS    VERSION
# payment-service-abc123        1/1     Running   v2.3.0
# payment-service-def456        1/1     Running   v2.3.0
# payment-service-ghi789        1/1     Running   v2.3.0
# payment-service-jkl012        1/1     Running   v2.3.0
# payment-service-mno345        1/1     Running   v2.3.0
# payment-service-pqr678        1/1     Running   v2.3.0

# Start deployment
$ kubectl set image deployment/payment-service \
    payment=registry.bank.com/prod/payment:v2.4.0 \
    -n production

# Watch rolling update
$ kubectl rollout status deployment/payment-service -n production
# Waiting for rollout to finish: 2 out of 6 new replicas have been updated...
# payment-service-abc123: Ready (new v2.4.0 pod created)
# payment-service-def456: Terminating (old v2.3.0)
# payment-service-stu901: Pending (new v2.4.0)
# ...
# deployment "payment-service" successfully rolled out

# Final state: All 6 pods running v2.4.0
$ kubectl get pods -l app=payment-service -n production
# NAME                          READY   STATUS    VERSION
# payment-service-stu901        1/1     Running   v2.4.0
# payment-service-vwx234        1/1     Running   v2.4.0
# payment-service-yza567        1/1     Running   v2.4.0
# payment-service-bcd890        1/1     Running   v2.4.0
# payment-service-efg123        1/1     Running   v2.4.0
# payment-service-hij456        1/1     Running   v2.4.0

# Verify zero downtime
$ curl -w "\nHTTP Code: %{http_code}\nTime: %{time_total}s\n" \
    https://api.bank.com/api/v1/payments/health
# {"status":"healthy","version":"2.4.0"}
# HTTP Code: 200
# Time: 0.045s

# Transaction monitoring during deployment
$ kubectl logs -l app=payment-service -n production --tail=100 | grep -c "Transaction completed"
# 15,847 transactions completed during deployment period
# 0 failed transactions
# 0 timeout errors
```

---

## 📋 Interview Questions

### Q1: What is the difference between a Pod and a Container?
**Answer:** A Container is a standalone runtime environment. A Pod is the smallest deployable unit in Kubernetes — it wraps one or more containers with shared storage and network. In banking, a Pod typically runs one main container (the application) and optionally sidecar containers (logging, monitoring). Pods provide the abstraction that allows Kubernetes to manage containers.

### Q2: How does Kubernetes handle zero-downtime deployments?
**Answer:** Kubernetes uses **Rolling Updates** by default. It creates new pods with the updated image, waits for them to pass health checks, then terminates old pods. Key settings: `maxSurge` (how many extra pods can be created) and `maxUnavailable` (how many pods can be down). With `maxSurge: 1, maxUnavailable: 0`, there's always at least the desired number of healthy pods running.

### Q3: What are liveness and readiness probes and why are they critical?
**Answer:** 

**Liveness probes** check if a container is alive. If it fails, Kubernetes restarts the container. 

**Readiness probes** check if a container is ready to serve traffic. If it fails, the pod is removed from service endpoints (no traffic sent). In banking, these are critical because: 

(1) Liveness probes detect deadlocks and memory leaks. 

(2) Readiness probes prevent sending traffic to pods that aren't ready (e.g., still initializing database connections).

### Q4: What is the difference between Deployment and StatefulSet?
**Answer:** 

Deployments are for **stateless** applications (each pod is identical, interchangeable). 

StatefulSets are for **stateful** applications (pods have stable identities, persistent storage). 

In banking: payment service → Deployment (stateless), database → StatefulSet (stateful, needs persistent storage and stable network identity).

### Q5: How do you manage application configuration in Kubernetes?
**Answer:** Three mechanisms: 

(1) **ConfigMaps** — non-sensitive configuration (feature flags, API URLs). 

(2) **Secrets** — sensitive data (passwords, API keys, certificates). 

(3) **Environment variables or volume mounts** — inject ConfigMaps/Secrets into pods. 

Best practice: use External Secrets Operator to sync from HashiCorp Vault, and never commit secrets to Git.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Pod | Smallest deployable unit, wraps containers |
| Deployment | Manages replicas, rolling updates, rollbacks |
| Service | Stable network endpoint for pods |
| Ingress | External HTTP/HTTPS access |
| HPA | Auto-scale based on metrics |
| Self-Healing | Automatic restart on failure |
| Banking Relevance | Zero downtime, auto-scaling, self-healing |

**Next:** [09-Kubernetes-Advanced.md](./09-Kubernetes-Advanced.md) — Advanced K8s concepts for banking.
