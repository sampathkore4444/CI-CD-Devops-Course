# 01 — CI/CD Basics: From Zero to Understanding

> **Goal:** Understand what CI/CD is, why it exists, and how it transforms software delivery.

---

## 🧱 What is CI/CD?

**CI/CD** stands for **Continuous Integration / Continuous Delivery (or Deployment)**. It is a set of practices that automates the process of getting code from a developer's laptop into production safely and quickly.

### Breaking It Down

| Term | Full Name | What It Means |
|------|-----------|---------------|
| **CI** | Continuous Integration | Developers merge code changes frequently; each merge triggers automated builds and tests |
| **CD** | Continuous Delivery | Code is always in a deployable state; releases happen on demand with a manual approval gate |
| **CD** | Continuous Deployment | Every change that passes tests is automatically deployed to production with zero human intervention |

### The Simple Analogy

Think of CI/CD like a **car factory assembly line**:

1. **Raw materials** = Source code (your developers write this)
2. **Assembly robot** = CI pipeline (builds, compiles, tests)
3. **Quality inspection station** = Automated tests (catches defects)
4. **Paint & finish** = Artifact creation (Docker images, binaries)
5. **Final inspection** = Staging environment testing
6. **Shipping to dealer** = Deployment to production

Without CI/CD, imagine hand-building every car from scratch — slow, error-prone, and expensive.

---

## 📜 The Evolution: How We Got Here

### Era 1: Manual Deployment (Pre-2000s)
```
Developer writes code → Hands CD to ops team → Ops installs on server → Pray it works
```
- Deployments happened **monthly or quarterly**
- "Works on my machine" was a real problem
- Rollbacks meant manually undoing changes

### Era 2: Scripted Deployment (2000s)
```
Developer writes code → Shell scripts deploy → Sometimes works → Usually breaks on Fridays
```
- Introduced basic automation
- Scripts were fragile and undocumented
- No standardized testing in the pipeline

### Era 3: CI/CD Revolution (2010s–Now)
```
Developer commits code → Pipeline auto-builds → Auto-tests → Auto-deploys → Monitors in production
```
- Every commit triggers the pipeline
- Automated quality gates prevent bad code from reaching production
- Deployments happen **multiple times per day**

---

## 🔄 The CI/CD Pipeline — Visual Overview

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  SOURCE   │───▶│  BUILD   │───▶│  TEST    │───▶│  STAGE   │───▶│ DEPLOY   │
│  CODE     │    │          │    │          │    │          │    │          │
│ (Git Push)│    │ (Compile)│    │ (Auto)   │    │ (Mirror  │    │(Prod)    │
│           │    │ (Docker) │    │ (Unit,   │    │  of Prod)│    │          │
│           │    │          │    │  Integ,  │    │          │    │          │
│           │    │          │    │  E2E)    │    │          │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                             ┌──────────┐
                                                             │ MONITOR  │
                                                             │(Observe) │
                                                             └──────────┘
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Online Banking Login Fix
**Context:** A bank's mobile app has a login bug — customers cannot sign in during peak hours.

**Without CI/CD:**
- Developer fixes the bug locally
- Sends a zip file to the ops team (via email!)
- Ops deploys on the next maintenance window (3 days later)
- **Impact:** 3 days of customer complaints, potential regulatory fine

**With CI/CD:**
- Developer pushes the fix to Git at 9:00 AM
- Pipeline auto-builds, runs unit tests, integration tests
- Code reviewed and merged by 10:30 AM
- Auto-deployed to production by 11:00 AM
- **Impact:** 2-hour turnaround, minimal customer impact

**Example Pipeline Output:**
```bash
$ git push origin hotfix/login-fix
[Pipeline] Starting: Build Stage
[Pipeline] Compiling Java application...
[Pipeline] Building Docker image: bank-app:v2.3.1-hotfix
[Pipeline] Running unit tests... 247/247 passed ✅
[Pipeline] Running integration tests... 89/89 passed ✅
[Pipeline] Deploying to production cluster...
[Pipeline] Health check: 200 OK ✅
[Pipeline] Deployment complete in 14 minutes
```

### Scenario 2: Regulatory Compliance Update
**Context:** New RBI (Reserve Bank of India) guidelines require adding a "Transaction Limit" feature to NEFT transfers within 30 days.

**Without CI/CD:**
- Development takes 3 weeks
- Testing takes 2 weeks (manual QA)
- Deployment takes 1 week (with approval chains)
- **Result:** Missed the deadline, regulatory penalty of ₹5 crores

**With CI/CD:**
- Feature developed in 2 weeks with automated testing
- Each sprint delivers tested, working increments
- Weekly deployments to staging for compliance team review
- Final deployment on day 25, 5 days ahead of deadline
- **Result:** On-time delivery, zero penalties

**Example Timeline:**
```
Week 1-2: Feature development (50+ commits, 200+ automated tests)
Week 3:   Integration testing on staging, compliance review
Week 4:   Performance testing, security audit (automated)
Day 25:   Production deployment ✅
Day 26-30: Buffer for monitoring and hotfixes
```

### Scenario 3: Multi-Region ATM Software Rollout
**Context:** A bank with 10,000 ATMs across 50 countries needs to update ATM software with new authentication features.

**Without CI/CD:**
- USB drives sent to each branch
- Manual installation at each ATM
- Some ATMs incompatible with new software
- **Result:** 6-month rollout, inconsistent software versions

**With CI/CD:**
- Software packaged as containerized artifact
- Deployed to regional staging clusters first
- Canary release: 5% of ATMs → 25% → 50% → 100%
- Automated rollback if error rate exceeds threshold
- **Result:** 2-week global rollout, consistent versions everywhere

**Example Canary Deployment:**
```yaml
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atm-software-canary
spec:
  replicas: 500  # 5% of 10,000 ATMs
  strategy:
    canary:
      steps:
        - setWeight: 5     # 500 ATMs
        - pause: {duration: 2h}
        - setWeight: 25    # 2,500 ATMs
        - pause: {duration: 4h}
        - setWeight: 50    # 5,000 ATMs
        - pause: {duration: 4h}
        - setWeight: 100   # All ATMs
```

---

## 🎯 Key Benefits of CI/CD

| Benefit | Without CI/CD | With CI/CD |
|---------|---------------|------------|
| **Deployment Frequency** | Monthly | Multiple times/day |
| **Lead Time** | Weeks to months | Hours to days |
| **Change Failure Rate** | 30-50% | 5-15% |
| **Recovery Time** | Days | Minutes to hours |
| **Manual Errors** | Frequent | Rare (automated) |
| **Regulatory Compliance** | Manual audits | Automated evidence |

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Building a NEFT Transfer Service from Scratch

**Business Requirement:** Bank needs a new NEFT (National Electronic Funds Transfer) service that complies with RBI guidelines.

**Step-by-Step CI/CD Journey:**

```
Day 1: Developer creates Git repo
  → git init + .gitignore + README.md
  → Push to GitLab

Day 2-5: Feature development
  → Write Java Spring Boot application
  → Add unit tests (50+ tests)
  → Each commit triggers CI pipeline
  → CI runs: build → test → security scan

Day 6: Code review & merge
  → Create Pull Request
  → 2 reviewers approve
  → CI confirms all 200+ tests pass
  → Merge to main branch

Day 7: Docker containerization
  → Write Dockerfile (multi-stage build)
  → Build Docker image: bank/neft-service:v1.0.0
  → Push to Harbor registry
  → Trivy scan: 0 vulnerabilities ✅

Day 8: Deploy to staging
  → Helm chart deploys to staging cluster
  → Integration tests run against staging
  → Compliance team reviews

Day 9: Production deployment
  → Release manager approves
  → Blue-green deployment to production
  → Traffic switched from Blue → Green
  → Monitoring confirms: 0 errors, latency < 200ms

Day 10: Go-live
  → NEFT service live for customers
  → 24/7 monitoring active
  → On-call team alerted if issues
```

**Key Metrics:**
- Time to market: 10 days (vs 6 months traditional)
- Test coverage: 87%
- Pipeline duration: 8 minutes
- Deployment frequency: Daily

### E2E Example 2: Multi-Region UPI Gateway Deployment

**Business Requirement:** Deploy UPI payment gateway across Mumbai and Singapore regions with disaster recovery.

**Architecture:**
```
┌─────────────────────────────────────────────────────────────┐
│                    GLOBAL LOAD BALANCER                      │
│                   (Route 53 / Cloudflare)                   │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐        ┌─────────────────────┐
│   Mumbai Region     │        │  Singapore Region   │
│   (Primary)         │◄──────▶│  (DR)               │
│                     │  Sync  │                     │
│  ┌───────────────┐  │        │  ┌───────────────┐  │
│  │ K8s Cluster   │  │        │  │ K8s Cluster   │  │
│  │ 6 nodes       │  │        │  │ 4 nodes       │  │
│  │ 3 AZs         │  │        │  │ 2 AZs         │  │
│  └───────────────┘  │        │  └───────────────┘  │
│  ┌───────────────┐  │        │  ┌───────────────┐  │
│  │ PostgreSQL    │  │        │  │ PostgreSQL    │  │
│  │ Primary +     │  │        │  │ Standby       │  │
│  │ 2 Replicas    │  │        │  │ (async)       │  │
│  └───────────────┘  │        │  └───────────────┘  │
└─────────────────────┘        └─────────────────────┘
```

**CI/CD Pipeline:**
```bash
# 1. Developer pushes UPI gateway code
$ git push origin feature/upi-gateway

# 2. CI Pipeline (3 minutes)
[Build] ✅ Maven compile
[Unit Tests] ✅ 456 tests passed
[SAST] ✅ SonarQube: 0 vulnerabilities
[Dependency Check] ✅ 0 CVEs
[Docker Build] ✅ Image: upi-gateway:v1.0.0
[Image Scan] ✅ Trivy: 0 critical/high

# 3. Deploy to Mumbai (Primary)
$ helm upgrade --install upi-gateway ./helm/upi-chart \
    -f ./helm/values-mumbai.yaml \
    -f ./helm/values-prod.yaml
$ kubectl rollout status deployment/upi-gateway -n production
# ✅ Mumbai: 6 replicas ready

# 4. Deploy to Singapore (DR)
$ helm upgrade --install upi-gateway ./helm/upi-chart \
    -f ./helm/values-singapore.yaml \
    -f ./helm/values-prod.yaml
$ kubectl rollout status deployment/upi-gateway -n production
# ✅ Singapore: 4 replicas ready

# 5. Verify cross-region connectivity
$ curl https://mumbai-upi.bank.com/health
# {"status": "UP", "region": "mumbai", "latency": "45ms"}
$ curl https://sg-upi.bank.com/health
# {"status": "UP", "region": "singapore", "latency": "38ms"}

# 6. Failover test
$ kubectl delete namespace production -n mumbai-cluster
# Singapore takes over traffic automatically
# Zero customer impact ✅

# 7. Recovery
$ helm upgrade --install upi-gateway ./helm/upi-chart \
    -f ./helm/values-mumbai.yaml -n production
# Mumbai back online, traffic restored
```

### E2E Example 3: Real-Time Fraud Detection System

**Business Requirement:** Deploy ML-based fraud detection that analyzes every transaction in real-time (< 100ms).

**System Architecture:**
```
Transaction Flow:

Customer App → API Gateway → Payment Service → Fraud Detection → Bank Core
                    │              │                  │              │
                    │              │                  │              │
                    │              │                  ▼              │
                    │              │          ┌──────────────┐      │
                    │              │          │ ML Model     │      │
                    │              │          │ (TensorFlow) │      │
                    │              │          │              │      │
                    │              │          │ Rules:       │      │
                    │              │          │ - Velocity   │      │
                    │              │          │ - Geo-fence  │      │
                    │              │          │ - Amount     │      │
                    │              │          │ - Device     │      │
                    │              │          └──────────────┘      │
                    │              │                  │              │
                    │              │          ┌───────┴───────┐      │
                    │              │          │ Decision      │      │
                    │              │          │ ALLOW/BLOCK   │      │
                    │              │          │ Review/Flag   │      │
                    │              │          └───────────────┘      │
```

**CI/CD Pipeline with ML:**
```yaml
# fraud-detection-pipeline.yml
stages:
  - train-model
  - validate-model
  - build-service
  - deploy

train-model:
  stage: train-model
  script:
    - python train.py --data /data/transactions.csv --output model/
    - python evaluate.py --model model/ --threshold 0.95
    # Model accuracy: 97.3% ✅ (threshold: 95%)
    # False positive rate: 0.8% ✅ (threshold: 1%)

validate-model:
  stage: validate-model
  script:
    - python validate_model.py --model model/
    # Bias check: gender平等 ✅
    # Bias check: age平等 ✅
    # Regulatory check: explainability ✅
    # Performance: P99 latency < 50ms ✅

build-service:
  stage: build-service
  script:
    - docker build --build-arg MODEL_PATH=model/ -t fraud-detection:v1.0.0 .
    - docker push registry.bank.com/fraud-detection:v1.0.0

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/fraud-detection \
        fraud-detection=registry.bank.com/fraud-detection:v1.0.0
    # Canary: 5% → 25% → 50% → 100%
    # Monitoring: fraud detection rate, false positives, latency
```

**Production Metrics:**
```
✅ Transaction analyzed in 45ms (threshold: 100ms)
✅ Fraud detection rate: 99.2%
✅ False positive rate: 0.3%
✅ 10 million transactions/day
✅ 500 fraudulent transactions blocked/day
✅ ₹50 crore fraud prevented/month
```

---

## 📋 Interview Questions

### Q1: What is the difference between Continuous Delivery and Continuous Deployment?
**Answer:** 

Continuous Delivery ensures code is always in a releasable state, but requires **manual approval** before deploying to production. 

Continuous Deployment goes one step further — **every change** that passes all automated tests is deployed to production automatically, with no human intervention. 

In banking, Continuous Delivery is often preferred because regulatory requirements may demand a human sign-off before production changes.

### Q2: Why should a bank adopt CI/CD instead of manual deployments?
**Answer:** Banks face three critical pressures: 

(1) **Regulatory compliance** — auditors require traceability of every change, which CI/CD provides through pipeline logs. 

(2) **Speed** — customers expect real-time fixes and features. 

(3) **Risk reduction** — automated testing catches issues before production, reducing the chance of outages that could affect millions of transactions.

### Q3: What are the stages of a typical CI/CD pipeline?
**Answer:** The standard stages are: 

(1) **Source** — code committed to version control, 

(2) **Build** — compile code, create artifacts/Docker images, 

(3) **Test** — unit tests, integration tests, security scans, 

(4) **Stage** — deploy to a staging environment mirroring production, 

(5) **Deploy** — push to production, 

(6) **Monitor** — observe performance, log errors, alert on issues.

### Q4: What is a "pipeline as code" and why is it important?
**Answer:** Pipeline as code means defining the CI/CD pipeline in a version-controlled file (like Jenkinsfile, .gitlab-ci.yml, or GitHub Actions workflow). This ensures the pipeline itself is auditable, reviewable, repeatable, and can be rolled back if changed — exactly what regulators want to see in banking environments.

### Q5: How does CI/CD handle rollbacks when something goes wrong?
**Answer:** There are several strategies: 

(1) **Blue-Green deployment** — keep the old version running; switch traffic back instantly. 

(2) **Canary release** — gradually route traffic; if errors spike, stop and roll back. 

(3) **Immutable artifacts** — previous Docker images are kept in the registry; Kubernetes can revert to the previous image tag. 

In banking, blue-green is most common because rollback must be instantaneous for zero-downtime requirements.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| CI | Merge code frequently, test automatically |
| CD (Delivery) | Always be deployable, manual approval gate |
| CD (Deployment) | Fully automated, zero-touch production releases |
| Pipeline | A sequence of automated stages from code to production |
| Banking Relevance | Speed, compliance, and risk reduction |

**Next:** [02-Version-Control-Git.md](./02-Version-Control-Git.md) — Learn Git, the foundation of every CI/CD pipeline.
