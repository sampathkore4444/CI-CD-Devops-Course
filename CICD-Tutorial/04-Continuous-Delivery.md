# 04 — Continuous Delivery (CD): Always Ready to Ship

> **Goal:** Understand Continuous Delivery — how to ensure your code is always production-ready.

---

## 📑 Table of Contents

- [🔍 What is Continuous Delivery?](#-what-is-continuous-delivery)
- [🏗️ Delivery Pipeline Architecture](#-delivery-pipeline-architecture)
- [🎭 Deployment Strategies](#-deployment-strategies)
- [🔄 The Approval Gate](#-the-approval-gate)
- [🏦 Real-World Banking Scenarios](#-real-world-banking-scenarios)
  - [Scenario 1: End-of-Day (EOD) Processing Update](#scenario-1-end-of-day-eod-processing-update)
  - [Scenario 2: Loan Disbursement System Upgrade](#scenario-2-loan-disbursement-system-upgrade)
  - [Scenario 3: Mobile Banking App Release](#scenario-3-mobile-banking-app-release)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Core Banking System Blue-Green Deployment](#e2e-example-1-core-banking-system-blue-green-deployment)
  - [E2E Example 2: Credit Card System Canary Deployment](#e2e-example-2-credit-card-system-canary-deployment)
  - [E2E Example 3: Mobile Banking App Release Pipeline](#e2e-example-3-mobile-banking-app-release-pipeline)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is Continuous Delivery?

**Continuous Delivery** means your code is **always in a deployable state**. Every change that passes the automated pipeline is ready to go to production at the push of a button (or approval of a gate).

### Key Difference: Delivery vs Deployment

```
Continuous Delivery:
  Code → Build → Test → Stage → [MANUAL APPROVAL] → Production
  
Continuous Deployment:
  Code → Build → Test → Stage → Production (automatic, no human touch)
```

In banking, **Continuous Delivery** is the standard because regulations often require a human sign-off before production changes.

---

## 🏗️ Delivery Pipeline Architecture

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  COMMIT   │──▶│  BUILD   │──▶│  TEST    │──▶│  STAGE   │──▶│ APPROVAL │──▶│ DEPLOY   │
│  to Git   │   │ Artifact │   │ All Tiers│   │ Mirror   │   │ Gate     │   │ Prod     │
│           │   │ Docker   │   │ Unit,E2E │   │ of Prod  │   │ Manual   │   │ Blue/    │
│           │   │ Image    │   │ Security │   │          │   │ Review   │   │ Green    │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## 🎭 Deployment Strategies

### Strategy 1: Blue-Green Deployment

```
BEFORE Deployment:
┌─────────────────┐     ┌─────────────────┐
│   BLUE (Live)    │     │   GREEN (Idle)   │
│   v2.3.0         │     │   v2.4.0         │
│   100% traffic   │     │   0% traffic     │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     │
              ┌──────┴──────┐
              │   Load      │
              │   Balancer  │
              └─────────────┘

AFTER Deployment (Instant Switch):
┌─────────────────┐     ┌─────────────────┐
│   BLUE (Idle)    │     │   GREEN (Live)   │
│   v2.3.0         │     │   v2.4.0         │
│   0% traffic     │     │   100% traffic   │
└─────────────────┘     └─────────────────┘

ROLLBACK: Switch traffic back to Blue (instant, zero downtime)
```

**Pros:** Instant rollback, zero downtime, simple
**Cons:** Requires 2x infrastructure (expensive)
**Best for:** Banks that need instant rollback capability

### Strategy 2: Canary Deployment

```
Step 1: Deploy to 5% of servers
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ v2.4 │ │ v2.3 │ │ v2.3 │ │ v2.3 │ │ v2.3 │
│  5%  │ │ 95%  │ │ 95%  │ │ 95%  │ │ 95%  │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘

Step 2: Monitor for 30 minutes, then increase to 25%
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ v2.4 │ │ v2.4 │ │ v2.3 │ │ v2.3 │ │ v2.3 │
│ 25%  │ │ 25%  │ │ 75%  │ │ 75%  │ │ 75%  │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘

Step 3: Monitor, then 50% → 100%

If error rate > threshold at ANY step → Automatic rollback
```

**Pros:** Gradual rollout, automatic rollback, minimal risk
**Cons:** Complex to implement, slow full rollout
**Best for:** Critical banking services where blast radius must be minimized

### Strategy 3: Rolling Update

```
Step 1: Replace 1 server at a time
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ v2.4 │ │ v2.3 │ │ v2.3 │ │ v2.3 │ │ v2.3 │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘
  Done

Step 2: Replace next server
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ v2.4 │ │ v2.4 │ │ v2.3 │ │ v2.3 │ │ v2.3 │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘
         Done

Continue until all servers are v2.4
```

**Pros:** Simple, no extra infrastructure
**Cons:** Slower rollback, mixed versions during deployment
**Best for:** Non-critical services, internal tools

---

## 🔄 The Approval Gate

In banking, the deployment gate is a **formal approval process**:

```
Pipeline reaches deployment stage
         │
         ▼
┌─────────────────────────────────┐
│         APPROVAL GATE           │
│                                 │
│ ✅ All tests passed             │
│ ✅ Security scan clean          │
│ ✅ Compliance checks passed     │
│ ✅ Performance benchmarks met   │
│                                 │
│ 👤 Requires approval from:      │
│    - Release Manager            │
│    - Security Team Lead         │
│    - Business Stakeholder       │
│                                 │
│ 📋 Click "Approve" to deploy    │
└─────────────────────────────────┘
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: End-of-Day (EOD) Processing Update
**Context:** The bank needs to update the End-of-Day batch processing system that calculates interest, processes standing instructions, and generates reports.

**Blue-Green Deployment:**
```bash
# Pre-deployment: Blue environment handles EOD processing
$ kubectl get pods -l env=blue
NAME                    READY   STATUS    AGE
eod-processor-blue-1    1/1     Running   30d
eod-processor-blue-2    1/1     Running   30d

# Deploy new version to Green environment
$ kubectl apply -f eod-processor-green-v2.4.yaml
$ kubectl get pods -l env=green
NAME                    READY   STATUS    AGE
eod-processor-green-1   1/1     Running   2m
eod-processor-green-2   1/1     Running   2m

# Run validation tests on Green
$ kubectl exec eod-processor-green-1 -- ./run-validation.sh
✅ Interest calculation: PASS
✅ Standing instructions: PASS
✅ Report generation: PASS
✅ Regulatory limits: PASS

# Switch traffic: Blue → Green
$ kubectl patch svc eod-service -p '{"spec":{"selector":{"env":"green"}}}'

# Monitor for 1 hour during EOD window
$ watch kubectl logs -l app=eod-processor,env=green --tail=50

# If any issue → instant rollback to Blue
$ kubectl patch svc eod-service -p '{"spec":{"selector":{"env":"blue"}}}'
# Rollback time: 3 seconds
```

### Scenario 2: Loan Disbursement System Upgrade
**Context:** Upgrade the loan disbursement system to support new government subsidy schemes. Must be deployed without any downtime.

**Canary Deployment:**
```yaml
# Step 1: Deploy canary (5% of traffic)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: loan-disbursement
spec:
  hosts:
    - loan-svc.internal.bank.com
  http:
    - route:
        - destination:
            host: loan-disbursement
            subset: stable
          weight: 95
        - destination:
            host: loan-disbursement
            subset: canary
          weight: 5

# Step 2: Monitor canary for 2 hours
# Metrics tracked:
# - Error rate: 0.001% (threshold: 0.01%) ✅
# - P99 latency: 450ms (threshold: 500ms) ✅
# - Throughput: 1,200 req/min (normal: 1,100) ✅

# Step 3: Increase canary to 25%
# Weight: stable=75, canary=25

# Step 4: Monitor for 2 more hours
# All metrics healthy ✅

# Step 5: Complete rollout
# Weight: stable=0, canary=100
# Rename canary → stable
```

### Scenario 3: Mobile Banking App Release
**Context:** Release a new version of the mobile banking app with UPI improvements.

**App Store Delivery Pipeline:**
```
Developer pushes code
         │
         ▼
┌─────────────────┐
│ Build iOS APK   │ ✅ Built in 8 minutes
│ Build Android   │ ✅ Built in 6 minutes
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Run E2E Tests   │ ✅ 45 test cases passed
│ (Appium/Selenium)│ ✅ On 3 device types
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Sign & Package  │ ✅ iOS: .ipa signed with Apple cert
│                 │ ✅ Android: .apk signed with keystore
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Upload to       │ ✅ Build uploaded to TestFlight
│ TestFlight/     │ ✅ Build uploaded to Play Console (internal)
│ Play Console    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ QA Team Tests   │ ✅ 5 business days testing
│ (Manual + Auto) │ ✅ 200+ test cases
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Compliance      │ ✅ RBI compliance check
│ Review          │ ✅ App store guidelines check
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Release to      │ ✅ Staged rollout: 10% → 50% → 100%
│ Production      │ ✅ Monitored for 7 days
└─────────────────┘
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Core Banking System Blue-Green Deployment

**Context:** Deploy new core banking version (v5.0.0) to 500+ branches with zero downtime.

```bash
# Pre-deployment checklist
$ kubectl get nodes
# node-prod-01: Ready (4 CPUs, 16GB RAM)
# node-prod-02: Ready (4 CPUs, 16GB RAM)
# node-prod-03: Ready (4 CPUs, 16GB RAM)

# Blue environment (current production)
$ kubectl get deployment core-banking-blue -n production
# NAME              READY   UP-TO-DATE   AVAILABLE
# core-banking-blue 6/6     6            6

# Green environment (new version)
$ kubectl apply -f core-banking-green-v5.0.0.yaml -n production
$ kubectl get deployment core-banking-green -n production
# NAME               READY   UP-TO-DATE   AVAILABLE
# core-banking-green  6/6     6            6

# Validation tests on Green
$ kubectl exec -it core-banking-green-abc123 -- ./validate.sh
✅ Account balance check: PASS
✅ Transaction history: PASS
✅ Interest calculation: PASS
✅ Regulatory limits: PASS
✅ Audit trail: PASS
✅ Encryption at rest: PASS
✅ Data masking: PASS

# Switch traffic: Blue → Green
$ kubectl patch svc core-banking -p '{"spec":{"selector":{"version":"green"}}}'

# Monitor for 2 hours
$ kubectl logs -l version=green -n production --tail=100 | grep -E "ERROR|WARN"
# No errors ✅

# Rollback plan (if needed)
$ kubectl patch svc core-banking -p '{"spec":{"selector":{"version":"blue"}}}'
# Rollback time: 3 seconds

# Final verification
$ curl https://api.bank.com/core/health
# {"status": "UP", "version": "5.0.0", "uptime": "7200s"}
```

### E2E Example 2: Credit Card System Canary Deployment

**Context:** Roll out new credit card authorization system to 10,000 merchants gradually.

```yaml
# Canary deployment configuration
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: credit-card-auth
  namespace: production
spec:
  replicas: 100
  strategy:
    canary:
      steps:
        # Step 1: 5% of traffic (500 merchants)
        - setWeight: 5
        - pause: {duration: 30m}
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: credit-card-auth
        
        # Step 2: 25% of traffic (2,500 merchants)
        - setWeight: 25
        - pause: {duration: 1h}
        - analysis:
            templates:
              - templateName: latency-check
        
        # Step 3: 50% of traffic (5,000 merchants)
        - setWeight: 50
        - pause: {duration: 2h}
        - analysis:
            templates:
              - templateName: fraud-rate
        
        # Step 4: 100% of traffic (all merchants)
        - setWeight: 100

---
# Analysis template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 5m
      count: 6
      successCondition: result[0] >= 0.999
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="credit-card-auth",status!~"5.."}[5m]))
            / sum(rate(http_requests_total{service="credit-card-auth"}[5m]))
```

**Canary Timeline:**
```
00:00 - Deploy canary (5%)
00:30 - Analysis: Success rate 99.99% ✅
00:31 - Increase to 25%
01:31 - Analysis: P99 latency 180ms ✅
01:32 - Increase to 50%
03:32 - Analysis: Fraud rate 0.001% ✅
03:33 - Increase to 100%
03:34 - Deployment complete ✅
```

### E2E Example 3: Mobile Banking App Release Pipeline

**Context:** Release new mobile banking app version to iOS and Android app stores.

```yaml
# Mobile App Delivery Pipeline
# File: .github/workflows/mobile-release.yml

name: Mobile Banking App Release
on:
  push:
    tags:
      - 'v*'  # Triggered by version tags

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Build Android APK
        run: |
          ./gradlew assembleRelease
          # Build time: 6 minutes
          # Output: app-release.apk (45MB)
      
      - name: Run Android Tests
        run: |
          ./gradlew test
          # Unit tests: 1,247 passed ✅
          # UI tests: 89 passed ✅
      
      - name: Sign APK
        run: |
          apksigner sign --ks release.keystore app-release.apk
          # Signed with bank's release key ✅
      
      - name: Upload to Play Console (Internal Testing)
        run: |
          fastlane android internal
          # Uploaded to Google Play Internal Testing track

  build-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
      
      - name: Build iOS IPA
        run: |
          xcodebuild -workspace BankingApp.xcworkspace \
            -scheme BankingApp \
            -archivePath build/BankingApp.xcarchive archive
          # Build time: 8 minutes
          # Output: BankingApp.ipa (52MB)
      
      - name: Run iOS Tests
        run: |
          xcodebuild test -workspace BankingApp.xcworkspace \
            -scheme BankingAppTests
          # Unit tests: 1,189 passed ✅
          # UI tests: 76 passed ✅
      
      - name: Upload to TestFlight
        run: |
          fastlane ios beta
          # Uploaded to Apple TestFlight

  qa-approval:
    needs: [build-android, build-ios]
    runs-on: ubuntu-latest
    steps:
      - name: Wait for QA Testing
        run: |
          echo "QA team has 5 business days to test"
          echo "Android: Internal Testing track"
          echo "iOS: TestFlight"
          # 200+ test cases must pass
          # RBI compliance verification
          # Performance benchmarking

  release-production:
    needs: qa-approval
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Release to Play Store (Staged)
        run: |
          fastlane android promote   # Internal → Production
          # Staged rollout: 10% → 50% → 100%
      
      - name: Release to App Store
        run: |
          fastlane ios release
          # Apple review: 24-48 hours
          # Staged rollout: 7 days
```

**Release Timeline:**
```
Day 1:  Build triggered by tag push
Day 2:  QA team starts testing
Day 6:  QA approval ✅
Day 7:  Android released (10% rollout)
Day 8:  Android 50% rollout
Day 9:  Android 100% rollout
Day 10: iOS submitted to App Store
Day 12: iOS approved by Apple
Day 13: iOS released (staged rollout)
Day 20: iOS 100% rollout
```

---

## 📋 Interview Questions

### Q1: Why do banks prefer Continuous Delivery over Continuous Deployment?
**Answer:** Banks are heavily regulated. Every production change requires: 

(1) **Business approval** — a product owner must sign off on the change. 

(2) **Compliance sign-off** — legal/compliance teams verify regulatory requirements. 

(3) **Audit trail** — regulators require evidence of who approved what. Continuous Delivery provides a **manual approval gate** between staging and production, while Continuous Deployment skips this gate entirely. 

Banks use CD (Delivery) to maintain control while still automating everything before the approval step.

### Q2: Compare Blue-Green and Canary deployments with real examples.
**Answer:** 

- **Blue-Green** is like switching power between two identical data centers — instant and complete. Example: Switching the ATM network from old to new software. If anything goes wrong, switch back in 3 seconds.

- **Canary** is like testing a new medicine on 100 patients before giving it to everyone. Example: Rolling out a new UPI feature to 5% of users first. If error rates spike, stop immediately and only 5% of users were affected.

### Q3: What is a staging environment and why is it critical?
**Answer:** A staging environment is an exact mirror of production — same infrastructure, same data (anonymized), same configurations. It's the final checkpoint before production. 

In banking, staging is critical because: 

(1) It catches issues that unit/integration tests miss (network latency, resource constraints). 

(2) It allows business users to validate features before customer-facing deployment. 

(3) It provides a safe space to test database migrations and performance under realistic load.

### Q4: How do you handle database schema changes in a delivery pipeline?
**Answer:** Database changes must be backward-compatible and follow a specific process: 

(1) **Additive changes first** — add new columns/tables (old code still works). 

(2) **Deploy code that works with both old and new schema.** 

(3) **Migrate data** if needed. 

(4) **Remove old columns** in a subsequent release. This "expand and contract" pattern ensures zero downtime. 

Tools like Flyway or Liquibase track and manage schema migrations in version control.

### Q5: What is a release train and when should banks use it?
**Answer:** A release train is a fixed schedule for deployments (e.g., every Thursday at 2 AM). All features that are ready join the train; those not ready wait for the next one. Banks use release trains because: 

(1) They provide predictability for operations teams. 

(2) They batch changes together, reducing the number of deployments. 

(3) They allow for comprehensive regression testing. 

(4) They align with maintenance windows. 

Example: "Train 45 deploys Thursday; features A, B, and C are onboard; feature D waits for Train 46."

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| CD (Delivery) | Always deployable, manual approval gate |
| Blue-Green | Two identical environments, instant switch |
| Canary | Gradual rollout, monitor at each step |
| Rolling Update | Replace servers one at a time |
| Approval Gate | Formal sign-off before production |
| Banking Relevance | Zero downtime, instant rollback, audit trail |

**Next:** [05-Continuous-Deployment.md](./05-Continuous-Deployment.md) — Learn about fully automated deployment.
