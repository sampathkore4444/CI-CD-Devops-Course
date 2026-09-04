# 23 — API Gateway: Managing Banking APIs

> **Goal:** Understand API Gateway — how banks manage, secure, and deploy APIs in CI/CD.

---

## 📑 Table of Contents

- [🔍 What is an API Gateway?](#-what-is-an-api-gateway)
- [🔧 API Gateway Features](#-api-gateway-features)
- [📋 API Gateway Configuration](#-api-gateway-configuration)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: API Versioning Strategy](#e2e-example-1-api-versioning-strategy)
  - [E2E Example 2: Rate Limiting for API Abuse Prevention](#e2e-example-2-rate-limiting-for-api-abuse-prevention)
  - [E2E Example 3: API Monitoring Dashboard](#e2e-example-3-api-monitoring-dashboard)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is an API Gateway?

An **API Gateway** is the single entry point for all client requests. It handles authentication, rate limiting, routing, and monitoring.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY ARCHITECTURE                          │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │ Mobile App  │  │ Web App     │  │ Partner API │               │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│         │                │                │                        │
│         └────────────────┼────────────────┘                        │
│                          │                                         │
│                  ┌───────┴───────┐                                 │
│                  │  API Gateway  │                                 │
│                  │  (Kong/NGINX) │                                 │
│                  └───────┬───────┘                                 │
│                          │                                         │
│    ┌─────────────────────┼─────────────────────┐                  │
│    │                     │                     │                  │
│    ▼                     ▼                     ▼                  │
│ ┌──────────┐      ┌──────────┐      ┌──────────┐                 │
│ │ Payment  │      │ Account  │      │ Loan     │                 │
│ │ Service  │      │ Service  │      │ Service  │                 │
│ └──────────┘      └──────────┘      └──────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 API Gateway Features

| Feature | Description | Banking Use Case |
|---------|-------------|-----------------|
| **Authentication** | Verify user identity | JWT/OAuth2 for mobile app |
| **Rate Limiting** | Throttle excessive requests | Prevent DDoS, API abuse |
| **Routing** | Direct to correct service | /payments → Payment Service |
| **Transformation** | Request/response modification | Mask card numbers |
| **Caching** | Store frequent responses | Account balance cache |
| **Logging** | Record all API calls | Audit trail for compliance |
| **Versioning** | Multiple API versions | v1, v2, v3 coexistence |

---

## 📋 API Gateway Configuration

```yaml
# Kong API Gateway configuration
# File: kong.yml

_format_version: "3.0"

services:
  - name: payment-service
    url: http://payment-service:8080
    routes:
      - name: payment-routes
        paths:
          - /api/v1/payments
        methods:
          - POST
          - GET
        strip_path: false
    plugins:
      - name: jwt
        config:
          claims_to_verify:
            - exp
            - iss
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis.bank.com
      - name: cors
        config:
          origins:
            - https://app.bank.com
          methods:
            - GET
            - POST
          headers:
            - Authorization
            - Content-Type
      - name: request-transformer
        config:
          add:
            headers:
              - X-Request-ID:$(uuid)
              - X-Timestamp:$(now)

  - name: account-service
    url: http://account-service:8080
    routes:
      - name: account-routes
        paths:
          - /api/v1/accounts
        methods:
          - GET
          - PUT
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 60
      - name: response-transformer
        config:
          add:
            headers:
              - X-Content-Type-Options:nosniff
              - X-Frame-Options:DENY
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: API Versioning Strategy

**Context:** Deploy new API version without breaking existing clients.

```yaml
# Kong service configuration for API versions
services:
  - name: payment-api-v1
    url: http://payment-service:8080/v1
    routes:
      - name: payment-v1
        paths:
          - /api/v1/payments
        plugins:
          - name: request-transformer
            config:
              add:
                headers:
                  - X-API-Version:1
          
  - name: payment-api-v2
    url: http://payment-service:8080/v2
    routes:
      - name: payment-v2
        paths:
          - /api/v2/payments
        plugins:
          - name: request-transformer
            config:
              add:
                headers:
                  - X-API-Version:2
          # New features in v2:
          # - Instant UPI support
          # - Multi-currency support
          # - Enhanced error responses
```

```bash
# Deploy v2 alongside v1
$ kubectl apply -f payment-service-v2.yaml -n production

# Monitor both versions
$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(istio_requests_total{destination_version="v2"}[5m])'
# ~50 req/s (25% of traffic - early adopters)

$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(istio_requests_total{destination_version="v1"}[5m])'
# ~150 req/s (75% of traffic - existing clients)

# Gradually shift traffic to v2
$ kubectl apply -f virtualservice-75-v2.yaml

# After 30 days, deprecate v1
$ curl -s -H "X-API-Version: 1" https://api.bank.com/api/v1/payments
# {
#   "error": "API v1 deprecated",
#   "message": "Please migrate to v2",
#   "deprecated_date": "2026-10-04",
#   "migration_guide": "https://docs.bank.com/api/v2/migration"
# }
```

### E2E Example 2: Rate Limiting for API Abuse Prevention

**Context:** Prevent DDoS and API abuse for banking APIs.

```yaml
# Rate limiting configuration per endpoint
plugins:
  - name: rate-limiting
    config:
      # Global rate limit
      minute: 1000
      hour: 10000
      policy: redis
      
  - name: rate-limiting
    config:
      # Per-user rate limit
      minute: 100
      hour: 1000
      policy: redis
      limit_by: consumer
```

```bash
# Test rate limiting
$ for i in {1..150}; do
    curl -s -o /dev/null -w "%{http_code}\n" \
        -H "Authorization: Bearer $TOKEN" \
        https://api.bank.com/api/v1/payments
done | sort | uniq -c
# 100 200  (successful)
#  50 429  (rate limited) ✅

# Rate limit response
$ curl -s -H "Authorization: Bearer $TOKEN" https://api.bank.com/api/v1/payments
# {
#   "error": "rate_limit_exceeded",
#   "message": "Rate limit exceeded. Try again in 30 seconds.",
#   "retry_after": 30
# }
# HTTP Status: 429 Too Many Requests
# Headers:
#   X-RateLimit-Limit: 100
#   X-RateLimit-Remaining: 0
#   X-RateLimit-Reset: 1693819260
```

### E2E Example 3: API Monitoring Dashboard

**Context:** Real-time API monitoring for banking operations.

```yaml
# Prometheus metrics from API Gateway
# File: grafana/api-dashboard.json

{
  "dashboard": {
    "title": "Banking API Dashboard",
    "panels": [
      {
        "title": "Request Rate (req/s)",
        "targets": [
          {
            "expr": "sum(rate(kong_http_requests_total[5m])) by (service)"
          }
        ]
      },
      {
        "title": "Error Rate (%)",
        "targets": [
          {
            "expr": "sum(rate(kong_http_requests_total{status=~\"5..\"}[5m])) / sum(rate(kong_http_requests_total[5m])) * 100"
          }
        ]
      },
      {
        "title": "Latency P99 (ms)",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(kong_http_request_duration_seconds_bucket[5m])) by (le, service)) * 1000"
          }
        ]
      },
      {
        "title": "Rate Limited Requests",
        "targets": [
          {
            "expr": "sum(rate(kong_http_requests_total{status=\"429\"}[5m])) by (consumer)"
          }
        ]
      }
    ]
  }
}
```

```bash
# Query API metrics
$ curl -s 'http://prometheus:9090/api/v1/query?query=sum(rate(kong_http_requests_total[5m])) by (service)'
# payment-service: 1247.5
# account-service: 892.3
# loan-service: 234.1
# gateway-service: 2156.7

# Check for suspicious activity
$ curl -s 'http://prometheus:9090/api/v1/query?query=topk(5, sum(rate(kong_http_requests_total{status="429"}[1h])) by (consumer))'
# consumer_app_123: 450 (potential abuse)
# consumer_app_456: 120
# consumer_app_789: 89
# consumer_app_012: 45
# consumer_app_345: 23
```

---

## 📋 Interview Questions

### Q1: What is the difference between API Gateway and Load Balancer?
**Answer:** **Load Balancer** distributes traffic across servers (Layer 4/7). **API Gateway** adds application-level features: authentication, rate limiting, request transformation, caching, monitoring. Load Balancer is infrastructure; API Gateway is application layer. Banks use both: Load Balancer for high availability, API Gateway for API management and security.

### Q2: How do you implement API versioning in banking?
**Answer:** Three strategies: (1) **URL versioning** — `/api/v1/payments`, `/api/v2/payments` (simple, recommended). (2) **Header versioning** — `Accept: application/vnd.bank.v2+json` (clean URLs). (3) **Query parameter** — `/api/payments?version=2` (easy but messy). Banks prefer URL versioning because: (1) Easy to route at gateway. (2) Clear for developers. (3) Easy to deprecate old versions. Always support at least 2 versions simultaneously.

### Q3: How do you handle API rate limiting for different customer tiers?
**Answer:** (1) **Premium customers** — 1000 requests/hour (higher limits). (2) **Regular customers** — 500 requests/hour. (3) **Partners** — 10000 requests/hour (B2B). (4) **Public APIs** — 100 requests/hour (free tier). Implementation: Kong/NGINX rate limiting with consumer groups. Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

### Q4: How do you secure banking APIs against common attacks?
**Answer:** (1) **Authentication** — JWT/OAuth2 with short-lived tokens. (2) **Rate limiting** — prevent brute force and DDoS. (3) **Input validation** — sanitize all inputs (SQL injection, XSS). (4) **CORS** — restrict origins to bank domains. (5) **Helmet headers** — security headers (X-Content-Type-Options, X-Frame-Options). (6) **IP whitelisting** — restrict API access to known IPs. (7) **API key rotation** — periodic key rotation for partners.

### Q5: How do you implement API monitoring and alerting?
**Answer:** (1) **Metrics** — request rate, error rate, latency (Prometheus). (2) **Logs** — all API calls logged with request ID (ELK). (3) **Traces** — distributed tracing across services (Jaeger). (4) **Alerts** — error rate > 1%, latency > 2s, rate limit exceeded (PagerDuty). (5) **Dashboards** — real-time API health (Grafana). (6) **SLA monitoring** — track API availability vs SLA targets. Banks require 99.99% API availability.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| API Gateway | Single entry point for all APIs |
| Authentication | JWT/OAuth2 for user verification |
| Rate Limiting | Prevent abuse and DDoS |
| Versioning | Support multiple API versions |
| Monitoring | Real-time API health |
| Banking Relevance | Security, compliance, availability |

**Next:** [24-Incident-Management.md](./24-Incident-Management.md) — Learn incident response for banking.
