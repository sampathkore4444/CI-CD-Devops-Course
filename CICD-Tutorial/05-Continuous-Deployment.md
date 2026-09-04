# 05 — Continuous Deployment: Zero-Touch Production Releases

> **Goal:** Understand Continuous Deployment — the fully automated path from code commit to production.

---

## 📑 Table of Contents

- [🔍 What is Continuous Deployment?](#-what-is-continuous-deployment)
- [🏗️ CD Pipeline Architecture](#-cd-pipeline-architecture)
- [🔄 CD in Kubernetes (Your Notes.md Flow)](#-cd-in-kubernetes-your-notesmd-flow)
- [📋 Complete Pipeline Example](#-complete-pipeline-example)
- [🏦 Real-World Banking Scenarios](#-real-world-banking-scenarios)
  - [Scenario 1: Microservice Auto-Deployment](#scenario-1-microservice-auto-deployment)
  - [Scenario 2: Feature Flag-Based Deployment](#scenario-2-feature-flag-based-deployment)
  - [Scenario 3: Automated Rollback on Failure](#scenario-3-automated-rollback-on-failure)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Real-Time Payment Microservice Deployment](#e2e-example-1-real-time-payment-microservice-deployment)
  - [E2E Example 2: Feature Flag Rollout for New UPI Feature](#e2e-example-2-feature-flag-rollout-for-new-upi-feature)
  - [E2E Example 3: Automated Rollback for Payment Gateway](#e2e-example-3-automated-rollback-for-payment-gateway)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is Continuous Deployment?

**Continuous Deployment** takes Continuous Delivery one step further: **every change** that passes all automated tests and quality gates is **automatically deployed to production** — with no human intervention.

```
Continuous Delivery:   Code → Build → Test → Stage → [APPROVAL] → Production
Continuous Deployment: Code → Build → Test → Stage → Production (automatic)
```

### When Does CD Make Sense?

| Scenario | Best Approach |
|----------|---------------|
| Banking core system | Continuous Delivery (manual approval) |
| Customer-facing mobile app | Continuous Delivery (app store review) |
| Internal developer tools | Continuous Deployment (fast iteration) |
| Microservices | Continuous Deployment (small, independent services) |
| API gateway configuration | Continuous Deployment (config changes) |

---

## 🏗️ CD Pipeline Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Git     │──▶│  Build  │──▶│  Test   │──▶│ Security│──▶│ Stage   │──▶│ Prod    │
│  Push    │   │  & Tag  │   │  Suite  │   │  Scan   │   │ Deploy  │   │ Deploy  │
│          │   │         │   │         │   │         │   │         │   │         │
│  Auto    │   │  Auto   │   │  Auto   │   │  Auto   │   │  Auto   │   │  Auto   │
│  trigger │   │  build  │   │  verify │   │  check  │   │  test   │   │  ship   │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
                                                                               │
                    ┌──────────────────────────────────────────────────────────┘
                    ▼
              ┌─────────┐   ┌─────────┐
              │ Monitor │──▶│ Auto    │
              │ & Alert │   │ Rollback│
              │         │   │ if bad  │
              └─────────┘   └─────────┘
```

---

## 🔄 CD in Kubernetes (Your Notes.md Flow)

This is the exact flow described in your Notes.md:

```
┌──────────────────┐
│ 1. Git Source    │  Developer pushes code
│    Code Repo     │  to the repository
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Trigger       │  Webhook fires when code is pushed
│    Pipeline      │  (Jenkins, GitLab CI, ArgoCD)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Build Artifact│  Compile code, run tests, build
│    & Push Image  │  Docker image, push to registry
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Docker        │  Private registry stores the image
│    Registry      │  (Harbor, ECR, GCR, ACR)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Pull Docker   │  K8s worker nodes pull the new
│    Image         │  image from the registry
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. Deploy        │  kubectl apply applies new manifests
│    (kubectl)     │  to the cluster
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 7. Monitor       │  Prometheus/Grafana monitors health
│    & Validate    │  Auto-rollback if metrics degrade
└──────────────────┘
```

---

## 📋 Complete Pipeline Example

```yaml
# .gitlab-ci.yml - Full Continuous Deployment Pipeline
stages:
  - build
  - test
  - security
  - stage
  - production
  - monitor

variables:
  DOCKER_REGISTRY: registry.bank.com
  APP_NAME: payment-service
  K8S_NAMESPACE: banking

# Stage 1: Build
build:
  stage: build
  script:
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - echo "Image: $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA"
  artifacts:
    reports:
      dotenv: build.env

# Stage 2: Unit Tests
unit-tests:
  stage: test
  script:
    - mvn test
    - mvn jacoco:report
  coverage: '/Total.*?(\d+%)/'
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

# Stage 3: Integration Tests
integration-tests:
  stage: test
  script:
    - docker-compose -f docker-compose.test.yml up -d
    - mvn verify -P integration-tests
    - docker-compose -f docker-compose.test.yml down
  services:
    - docker:dind

# Stage 4: Security Scan
security-scan:
  stage: security
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - sonar-scanner -Dsonar.projectKey=$APP_NAME
  allow_failure: false

# Stage 5: Deploy to Staging
deploy-staging:
  stage: stage
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/$APP_NAME -n staging --timeout=300s
  environment:
    name: staging
    url: https://staging.bank.com

# Stage 6: Deploy to Production (Automatic)
deploy-production:
  stage: production
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/$APP_NAME -n production --timeout=300s
  environment:
    name: production
    url: https://api.bank.com
  when: on_success
  only:
    - main

# Stage 7: Post-Deploy Monitoring
post-deploy-monitor:
  stage: monitor
  script:
    - python scripts/validate-deployment.py --duration=5m --error-threshold=0.01
    - python scripts/check-health-endpoints.py
  when: on_success
  allow_failure: false
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Microservice Auto-Deployment
**Context:** A bank has 50+ microservices. Each service is independently deployable and follows continuous deployment.

**Service Architecture:**
```
┌─────────────────────────────────────────────────┐
│                 API Gateway                      │
│              (rate limiting, auth)               │
└──────────┬──────────┬──────────┬────────────────┘
           │          │          │
     ┌─────┴───┐ ┌───┴─────┐ ┌─┴──────────┐
     │ Account │ │Payment  │ │Notification│
     │ Service │ │Service  │ │ Service    │
     │ (Go)    │ │ (Java)  │ │ (Node.js)  │
     └─────────┘ └─────────┘ └────────────┘
           │          │          │
     ┌─────┴───┐ ┌───┴─────┐ ┌─┴──────────┐
     │ Account │ │Payment  │ │Notification│
     │ DB      │ │DB       │ │ Queue      │
     │(Postgres)│ │(Oracle) │ │ (RabbitMQ) │
     └─────────┘ └─────────┘ └────────────┘
```

**Auto-Deployment Flow:**
```bash
# Developer pushes to notification-service
$ git push origin feature/sms-via-twilio

# Pipeline runs in 3 minutes (small service)
[Pipeline] Build Docker image ✅ (45s)
[Pipeline] Run 89 unit tests ✅ (30s)
[Pipeline] Security scan ✅ (15s)
[Pipeline] Deploy to staging ✅ (30s)
[Pipeline] Run smoke tests ✅ (1m)
[Pipeline] Deploy to production ✅ (30s)
[Pipeline] Monitor for anomalies ✅ (5m)

# Only notification-service was affected
# Account and Payment services: UNCHANGED, ZERO RISK
```

### Scenario 2: Feature Flag-Based Deployment
**Context:** Deploy code to production but keep new features hidden until ready to enable them.

**Implementation:**
```java
@Service
public class TransferService {
    
    @Autowired
    private FeatureFlagService featureFlags;
    
    public Transaction initiateTransfer(TransferRequest request) {
        // Feature 1: Standard transfer (always enabled)
        Transaction tx = processStandardTransfer(request);
        
        // Feature 2: Instant UPI (feature flag controlled)
        if (featureFlags.isEnabled("instant-upi-transfer")) {
            processInstantUPI(tx);
        }
        
        // Feature 3: AI fraud check (gradual rollout)
        if (featureFlags.isEnabled("ai-fraud-check", request.getUserId())) {
            runAIFraudCheck(tx);
        }
        
        return tx;
    }
}
```

**Feature Flag Configuration:**
```json
{
  "instant-upi-transfer": {
    "enabled": true,
    "rollout_percentage": 25,
    "allowed_user_groups": ["premium", "business"]
  },
  "ai-fraud-check": {
    "enabled": true,
    "rollout_percentage": 10,
    "allowed_regions": ["Mumbai", "Delhi"]
  }
}
```

**Result:** Code is deployed but features are invisible until toggled. No rollback needed — just disable the flag.

### Scenario 3: Automated Rollback on Failure
**Context:** Deploy a new version, but if error rate exceeds 0.1% within 5 minutes, automatically rollback.

**Monitoring & Auto-Rollback:**
```python
# scripts/validate-deployment.py
import time
import requests
import sys

def monitor_deployment(duration_minutes=5, error_threshold=0.001):
    """Monitor deployment and auto-rollback if errors exceed threshold"""
    
    start_time = time.time()
    end_time = start_time + (duration_minutes * 60)
    total_requests = 0
    error_count = 0
    
    while time.time() < end_time:
        # Query Prometheus for error metrics
        metrics = requests.get(
            "http://prometheus:9090/api/v1/query",
            params={
                "query": f'sum(rate(http_requests_total{{status=~"5..",service="payment"}}[1m]))'
            }
        ).json()
        
        error_rate = float(metrics['data']['result'][0]['value'][1])
        
        if error_rate > error_threshold:
            print(f"❌ ERROR RATE {error_rate:.4%} EXCEEDS THRESHOLD {error_threshold:.4%}")
            print("🔄 Initiating automatic rollback...")
            
            # Rollback to previous version
            import subprocess
            subprocess.run([
                "kubectl", "rollout", "undo", 
                "deployment/payment-service", 
                "-n", "production"
            ])
            
            print("✅ Rollback complete")
            sys.exit(1)
        
        print(f"✅ Error rate: {error_rate:.4%} (threshold: {error_threshold:.4%})")
        time.sleep(30)
    
    print("✅ Deployment validation passed - no rollback needed")

if __name__ == "__main__":
    monitor_deployment()
```

**Example Output:**
```
$ python scripts/validate-deployment.py
[00:00] Error rate: 0.001% (threshold: 0.1%) ✅
[00:30] Error rate: 0.002% (threshold: 0.1%) ✅
[01:00] Error rate: 0.001% (threshold: 0.1%) ✅
[01:30] Error rate: 0.003% (threshold: 0.1%) ✅
[02:00] Error rate: 0.001% (threshold: 0.1%) ✅
[02:30] Error rate: 0.002% (threshold: 0.1%) ✅
[03:00] Error rate: 0.001% (threshold: 0.1%) ✅
[03:30] Error rate: 0.002% (threshold: 0.1%) ✅
[04:00] Error rate: 0.001% (threshold: 0.1%) ✅
[04:30] Error rate: 0.002% (threshold: 0.1%) ✅
[05:00] ✅ Deployment validation passed - no rollback needed
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Real-Time Payment Microservice Deployment

**Context:** Deploy 5 microservices independently with continuous deployment.

```bash
# Service Architecture:
# 1. payment-gateway (Go) - Handles UPI/NEFT/RTGS
# 2. account-service (Java) - Account management
# 3. fraud-detection (Python) - ML-based fraud
# 4. notification-service (Node.js) - SMS/Email/Push
# 5. reconciliation-service (Java) - Daily reconciliation

# Service 1: payment-gateway (Deployed 3 times today)
$ git push origin feature/rtgs-enhancement
[Pipeline] Build: 45s
[Pipeline] Tests: 156/156 passed
[Pipeline] Deploy to production: 30s
[Pipeline] Auto-rollback: Not triggered (error rate 0.001%)

# Service 2: fraud-detection (Deployed 1 time today)
$ git push origin feat/new-ml-model
[Pipeline] Train model: 2h
[Pipeline] Validate model: 97.3% accuracy ✅
[Pipeline] Deploy canary: 5% → 25% → 100%
[Pipeline] Auto-rollback: Not triggered (fraud rate 0.3%)

# Service 3: notification-service (Deployed 5 times today)
$ git push origin fix/sms-timeout
[Pipeline] Build: 30s
[Pipeline] Tests: 89/89 passed
[Pipeline] Deploy: 20s

# Service 4: account-service (Deployed 2 times today)
$ git push origin feat/balance-inquiry
[Pipeline] Build: 1m 15s
[Pipeline] Tests: 423/423 passed
[Pipeline] Deploy: 45s

# Service 5: reconciliation-service (Deployed 1 time today)
$ git push origin feat/t+0-reconciliation
[Pipeline] Build: 2m
[Pipeline] Tests: 312/312 passed
[Pipeline] Deploy: 1m

# Summary:
# Total deployments today: 12
# Successful: 12
# Auto-rollbacks: 0
# Average deploy time: 45 seconds
# Zero customer impact ✅
```

### E2E Example 2: Feature Flag Rollout for New UPI Feature

**Context:** Deploy "UPI Lite" feature (offline payments) to 1 million users gradually.

```java
// Feature flag implementation
@Service
public class UPILiteService {
    
    @Autowired
    private FeatureFlagService featureFlags;
    
    public PaymentResult processUPILitePayment(UPILiteRequest request) {
        // Check if UPI Lite is enabled for this user
        if (!featureFlags.isEnabled("upi-lite", request.getUserId())) {
            throw new FeatureNotAvailableException("UPI Lite not enabled for this user");
        }
        
        // Check wallet balance (UPI Lite uses wallet)
        BigDecimal walletBalance = walletService.getBalance(request.getUserId());
        if (walletBalance.compareTo(request.getAmount()) < 0) {
            throw new InsufficientBalanceException("UPI Lite wallet balance insufficient");
        }
        
        // Process offline payment
        return processOfflinePayment(request);
    }
}
```

```yaml
# Feature flag configuration
# File: feature-flags.yaml
features:
  upi-lite:
    enabled: true
    rollout_percentage: 25
    target_segments:
      - premium_users
      - beta_testers
      - users_in_mumbai  # Geographic rollout
    metrics:
      - success_rate > 0.99
      - latency_p99 < 500ms
      - error_rate < 0.01
    auto_disable:
      enabled: true
      threshold: error_rate > 0.01
```

**Rollout Timeline:**
```
Day 1: Deploy code with flag OFF (0% users)
Day 2: Enable for beta testers (1,000 users)
Day 3: Monitor: Success rate 99.98% ✅
Day 4: Enable for Mumbai (100,000 users)
Day 5: Monitor: Success rate 99.99% ✅
Day 6: Enable for premium users (500,000 users)
Day 7: Monitor: Success rate 99.97% ✅
Day 8: Enable for all users (1,000,000 users)
Day 9: Monitor: Success rate 99.99% ✅
Day 10: Feature fully rolled out ✅

If error rate > 0.01% at any step:
→ Auto-disable flag
→ Zero rollback needed
→ Users see old behavior instantly
```

### E2E Example 3: Automated Rollback for Payment Gateway

**Context:** Deploy new payment gateway version; auto-rollback if error rate exceeds threshold.

```bash
# 10:00 AM - Deploy new version
$ kubectl set image deployment/payment-gateway \
    gateway=registry.bank.com/gateway:v2.5.0 \
    -n production
$ kubectl rollout status deployment/payment-gateway -n production
# deployment "payment-gateway" successfully rolled out

# 10:01 AM - Monitoring script starts
$ python scripts/monitor-deployment.py --duration 10m

[10:01:00] Error rate: 0.002% (threshold: 0.1%) ✅
[10:01:30] Error rate: 0.003% (threshold: 0.1%) ✅
[10:02:00] Error rate: 0.001% (threshold: 0.1%) ✅
[10:02:30] Error rate: 0.002% (threshold: 0.1%) ✅
[10:03:00] Error rate: 0.001% (threshold: 0.1%) ✅

# 10:05 AM - Sudden spike detected!
[10:05:00] Error rate: 0.15% (threshold: 0.1%) ❌
[10:05:01] 🔄 AUTO-ROLLBACK INITIATED
[10:05:02] kubectl rollout undo deployment/payment-gateway -n production
[10:05:05] Rollback complete: gateway:v2.4.9
[10:05:10] Error rate: 0.001% (normalized) ✅
[10:05:11] Alert: "Auto-rollback completed for payment-gateway"
[10:05:12] Notification: Slack #incidents channel

# Post-mortem:
# Root cause: v2.5.0 had memory leak in connection pooling
# Impact: 50 requests affected over 5 minutes
# Customer impact: Minimal (auto-rollback in 5 seconds)
# Fix: v2.5.1 released next day
```

---

## 📋 Interview Questions

### Q1: What is the difference between continuous delivery and continuous deployment?
**Answer:** The only difference is the **approval gate**. 

Continuous Delivery requires a human to approve before production deployment. 

Continuous Deployment deploys automatically. 

In banking, most core systems use Continuous Delivery (regulatory requirement), while some internal tools and non-critical microservices may use Continuous Deployment for faster iteration.

### Q2: How do you implement automated rollbacks in Kubernetes?
**Answer:** Kubernetes provides built-in rollback via `kubectl rollout undo`. 

To automate: 

(1) Deploy the new version. 

(2) Monitor error rates, latency, and health checks for a defined window. 

(3) If metrics exceed thresholds, run `kubectl rollout undo deployment/<name>`. 

(4) This automatically reverts to the previous ReplicaSet. 

For more sophistication, use Argo Rollouts with automated analysis steps.

### Q3: What are feature flags and how do they decouple deployment from release?
**Answer:** Feature flags are configuration toggles that control whether a feature is visible to users. Code can be deployed to production with a flag set to `false`, meaning the feature exists but is hidden. When ready, toggle the flag to `true` to "release" the feature. This decouples **deployment** (getting code to production) from **release** (making it available to users). In banking, this allows gradual rollout to specific customer segments.

### Q4: What is the "12-Factor App" methodology and why does it matter for CD?
**Answer:** The 12-Factor App is a set of best practices for building modern, scalable applications: 

(1) Codebase tracked in Git, 

(2) Dependencies declared explicitly, 

(3) Config stored in environment variables, 

(4) Backing services as attached resources, 

(5) Strict separation of build and run stages, 

(6) Stateless processes, 

(7) Disposability (fast startup/shutdown), 

(8) Dev/prod parity, 

(9) Logs as event streams, 

(10) Admin processes as one-off tasks. 

These principles make applications easier to deploy, scale, and manage in CD pipelines.

### Q5: How do you handle secret rotation in a continuous deployment pipeline?
**Answer:** 

(1) **Never store secrets in Git** — use Vault, AWS Secrets Manager, or K8s Secrets. 

(2) **Automated rotation** — set up periodic rotation (e.g., database passwords every 90 days). 

(3) **Zero-downtime rotation** — use techniques like dual-write during rotation window. 

(4) **K8s Secrets** — mount secrets as volumes; pods reload on next restart. 

(5) **External Secrets Operator** — syncs secrets from Vault to K8s automatically.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| CD (Deployment) | Fully automated, no human intervention |
| Pipeline Flow | Git → Build → Test → Security → Stage → Prod |
| Feature Flags | Decouple deployment from release |
| Auto-Rollback | Monitor and revert if metrics degrade |
| Kubernetes | kubectl apply for deployment, rollout undo for rollback |
| Banking Relevance | Fast iteration for non-critical, controlled for core |

**Next:** [06-Docker-Fundamentals.md](./06-Docker-Fundamentals.md) — Learn containers, the building blocks of modern CI/CD.
