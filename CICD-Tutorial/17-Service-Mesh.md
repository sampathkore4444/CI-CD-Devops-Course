# 17 — Service Mesh: Istio for Banking Microservices

> **Goal:** Understand Service Mesh — how Istio secures, observes, and controls microservice communication.

---

## 📑 Table of Contents

- [🔍 What is a Service Mesh?](#-what-is-a-service-mesh)
- [🏗️ Istio Architecture](#️-istio-architecture)
- [🔒 Security Features](#-security-features)
- [📊 Observability Features](#-observability-features)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Secure Payment Service Communication](#e2e-example-1-secure-payment-service-communication)
  - [E2E Example 2: Traffic Management for Canary Release](#e2e-example-2-traffic-management-for-canary-release)
  - [E2E Example 3: Circuit Breaker for Fault Tolerance](#e2e-example-3-circuit-breaker-for-fault-tolerance)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is a Service Mesh?

A **Service Mesh** is an infrastructure layer that handles **service-to-service communication**. It provides security, observability, and traffic management without changing application code.

### The Problem It Solves

```
Without Service Mesh (50 microservices):
  Each service needs:
  - mTLS certificate management ❌
  - Retry logic ❌
  - Circuit breaker ❌
  - Distributed tracing ❌
  - Rate limiting ❌
  - Load balancing ❌
  
  = 50 services × 6 concerns = 300 custom implementations!
  = Inconsistent, error-prone, hard to maintain

With Service Mesh (Istio):
  All concerns handled by the mesh:
  ✅ mTLS automatic
  ✅ Retries automatic
  ✅ Circuit breakers automatic
  ✅ Tracing automatic
  ✅ Rate limiting automatic
  ✅ Load balancing automatic
  
  = Developers focus on business logic only
```

---

## 🏗️ Istio Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ISTIO ARCHITECTURE                                │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    CONTROL PLANE (istiod)                     │  │
│  │                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │   Pilot     │  │   Citadel   │  │  Telemetry  │         │  │
│  │  │ (Config     │  │ (Certificate│  │ (Metrics,   │         │  │
│  │  │  Distribution)│ │  Authority) │  │  Tracing)   │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    DATA PLANE                                 │  │
│  │                                                               │  │
│  │  ┌─────────────────┐         ┌─────────────────┐            │  │
│  │  │ Payment Pod     │         │ Account Pod     │            │  │
│  │  │ ┌─────────────┐ │         │ ┌─────────────┐ │            │  │
│  │  │ │ Payment     │ │ ◀─mTLS─▶│ │ Account     │ │            │  │
│  │  │ │ Service     │ │         │ │ Service     │ │            │  │
│  │  │ └─────────────┘ │         │ └─────────────┘ │            │  │
│  │  │ ┌─────────────┐ │         │ ┌─────────────┐ │            │  │
│  │  │ │ Envoy       │ │         │ │ Envoy       │ │            │  │
│  │  │ │ Sidecar     │ │         │ │ Sidecar     │ │            │  │
│  │  │ └─────────────┘ │         │ └─────────────┘ │            │  │
│  │  └─────────────────┘         └─────────────────┘            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Security Features

### 1. Automatic mTLS (Mutual TLS)
```yaml
# PeerAuthentication - Enforce mTLS for all services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # All service-to-service traffic encrypted
```

### 2. Authorization Policies
```yaml
# Allow only payment-service to call account-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: account-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: account-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/payment-service"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/v1/accounts/*"]
```

### 3. Rate Limiting
```yaml
# Rate limit: 100 requests per second per user
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit
  namespace: production
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 100
                tokens_per_fill: 100
                fill_interval: 1s
```

---

## 📊 Observability Features

```yaml
# Telemetry - Enable distributed tracing
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: banking-tracing
  namespace: production
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 100  # Trace all requests
  metrics:
    - providers:
        - name: prometheus
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Secure Payment Service Communication

**Context:** Enforce mTLS and authorization policies for all payment flows.

```bash
# Install Istio
$ istioctl install --set profile=default -y
# ✔ Istio installed successfully

# Enable sidecar injection for production namespace
$ kubectl label namespace production istio-injection=enabled
# namespace/production labeled

# Deploy payment service (Envoy sidecar injected automatically)
$ kubectl apply -f payment-deployment.yaml -n production
$ kubectl get pods -n production
# NAME                          READY   STATUS    RESTARTS
# payment-service-abc123        2/2     Running   0  # 2/2 = app + sidecar
# account-service-def456        2/2     Running   0
# notification-service-ghi789   2/2     Running   0

# Verify mTLS is working
$ istioctl authn tls-check payment-service-abc123 account-service.production.svc.cluster.local
# account-service.production.svc.cluster.local:443
#   Protocol: tcp
#   TLS: STRICT

# Test: Unencrypted traffic should fail
$ kubectl exec -it payment-service-abc123 -c payment -- curl http://account-service:8080/api/v1/accounts/12345
# curl: (56) Recv failure: Connection reset by peer ❌ (mTLS blocking plain HTTP)

# Test: Encrypted traffic works
$ kubectl exec -it payment-service-abc123 -c payment -- curl https://account-service:443/api/v1/accounts/12345
# {"accountId":"12345","balance":50000.00,"currency":"INR"} ✅
```

### E2E Example 2: Traffic Management for Canary Release

**Context:** Gradually shift traffic from v2.4 to v2.5 for payment service.

```yaml
# VirtualService for traffic splitting
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: stable
          weight: 90
        - destination:
            host: payment-service
            subset: canary
          weight: 10
---
# DestinationRule for subsets
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service
  subsets:
    - name: stable
      labels:
        version: v2.4.0
    - name: canary
      labels:
        version: v2.5.0
```

```bash
# Deploy canary version
$ kubectl apply -f payment-v2.5.0.yaml -n production

# Shift traffic gradually
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: stable
          weight: 75
        - destination:
            host: payment-service
            subset: canary
          weight: 25
EOF

# Monitor for 1 hour
$ istioctl proxy-status
# payment-service-abc123.synced  ✅

# Check metrics
$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(istio_requests_total{destination_version="v2.5.0"}[5m])'
# ~250 req/s (25% of 1000 total) ✅

# Continue shifting to 50% → 100%
```

### E2E Example 3: Circuit Breaker for Fault Tolerance

**Context:** Prevent cascade failures when downstream service is slow.

```yaml
# Circuit breaker configuration
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: account-service-circuit-breaker
  namespace: production
spec:
  host: account-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5      # Eject after 5 consecutive 5xx
      interval: 30s                 # Check every 30 seconds
      baseEjectionTime: 30s         # Eject for 30 seconds minimum
      maxEjectionPercent: 50        # Max 50% of pods can be ejected
```

```bash
# Simulate account-service failure
$ kubectl exec -it account-service-def456 -c account -- kill -9 1

# Payment service behavior with circuit breaker:
# Request 1-4: Forwarded to account-service (slow/failing)
# Request 5: 5th consecutive error → circuit opens
# Request 6+: Immediately returns 503 (no timeout, no cascade)
# After 30s: Circuit half-opens, tries 1 request
# If success: Circuit closes, normal traffic resumes
# If failure: Circuit opens again

# Monitoring
$ curl -s 'http://prometheus:9090/api/v1/query?query=istio_circuit_breaker_opening_total'
# {"value":[1693819200,"1"]}  # 1 circuit breaker opened
```

---

## 📋 Interview Questions

### Q1: What is the difference between a Service Mesh and an API Gateway?
**Answer:** **Service Mesh** handles **internal** service-to-service communication (east-west traffic). **API Gateway** handles **external** client-to-service communication (north-south traffic). In banking: API Gateway authenticates customers, rate limits external requests. Service Mesh secures internal microservice calls with mTLS, provides observability, and implements circuit breakers.

### Q2: How does Istio implement mTLS without code changes?
**Answer:** Istio uses **Envoy sidecar proxies** injected into each pod. When a pod starts, Istio: (1) Injects Envoy sidecar container. (2) Generates mTLS certificate via Citadel. (3) Envoy intercepts all inbound/outbound traffic. (4) Encrypts traffic between sidecars transparently. Application code doesn't know about mTLS — it's all handled at the infrastructure layer.

### Q3: What is circuit breaking and why is it critical for banking?
**Answer:** Circuit breaking prevents cascade failures. If Account Service is slow/down, Payment Service would wait for responses, exhausting connections and failing too. Circuit breaker: (1) Detects consecutive failures. (2) "Opens" the circuit — stops sending requests. (3) Returns fallback response immediately. (4) Periodically checks if downstream recovered. (5) "Closes" circuit when healthy. Critical for banking because one slow service shouldn't bring down the entire payment system.

### Q4: How do you implement rate limiting with Istio?
**Answer:** Three approaches: (1) **Local rate limiting** — each Envoy instance limits independently (fast, no central coordination). (2) **Global rate limiting** — central rate limit service coordinates across all instances (accurate, more complex). (3) **AuthorizationPolicy** — restrict which services can call which endpoints. For banking: use local rate limiting for DDoS protection, global rate limiting for API quota management.

### Q5: How does Service Mesh help with PCI-DSS compliance?
**Answer:** PCI-DSS requires: (1) **Encryption in transit** — Istio mTLS encrypts all internal traffic. (2) **Access control** — AuthorizationPolicy restricts service-to-service communication. (3) **Audit logging** — Istio logs all requests with source, destination, response code. (4) **Network segmentation** — VirtualService isolates payment card data flows. (5) **Certificate management** — Citadel automates certificate rotation (90-day cycle).

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Service Mesh | Infrastructure layer for service communication |
| Istio | Most popular service mesh for Kubernetes |
| mTLS | Automatic encryption between services |
| AuthorizationPolicy | Control which services can communicate |
| Circuit Breaker | Prevent cascade failures |
| Banking Relevance | PCI-DSS compliance, fault tolerance |

**Next:** [18-Chaos-Engineering.md](./18-Chaos-Engineering.md) — Learn how to test resilience through controlled failures.
