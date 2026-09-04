# 03 — Continuous Integration (CI): Catch Bugs Before They Catch You

> **Goal:** Understand CI in depth — how automated builds and tests keep your codebase healthy.

---

## 🔍 What is Continuous Integration?

**Continuous Integration** is the practice where developers **frequently merge** their code changes into a shared repository. Each merge triggers an **automated build and test** process that validates the code works correctly.

### The Core Principle

```
"Integrate early, integrate often, and let the machine tell you if something broke."
```

### CI vs No CI

```
Without CI:
  Developer A: Works on feature for 3 weeks
  Developer B: Works on feature for 3 weeks
  Merge Day: "There are 47 conflicts and nothing compiles"
  Resolution: 2 more weeks of fixing merge conflicts

With CI:
  Developer A: Commits code every few hours
  Developer B: Commits code every few hours
  Every commit: Automated build + tests run in 5 minutes
  Result: Conflicts caught in minutes, not weeks
```

---

## 🏗️ How CI Works — Step by Step

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Developer   │────▶│  Git Push    │────▶│  CI Server   │────▶│  Results    │
│  commits     │     │  triggers    │     │  (Jenkins/   │     │  (Pass/Fail │
│  code        │     │  webhook     │     │  GitLab CI)  │     │  + Report)  │
└─────────────┘     └─────────────┘     └──────┬──────┘     └─────────────┘
                                               │
                    ┌──────────────────────────┼──────────────────────────┐
                    ▼                          ▼                          ▼
              ┌──────────┐              ┌──────────┐              ┌──────────┐
              │  BUILD   │              │  TEST    │              │  REPORT  │
              │ Compile  │              │ Unit     │              │ Email    │
              │ Package  │              │ Integr.  │              │ Slack    │
              │ Docker   │              │ E2E      │              │ Dashboard│
              └──────────┘              └──────────┘              └──────────┘
```

---

## 📋 CI Pipeline Stages in Detail

### Stage 1: Source (Code Checkout)
```yaml
# The CI server pulls the latest code
- checkout:
    repo: bank-payment-service
    branch: feature/new-payment
    commit: abc123def
```

### Stage 2: Build (Compile & Package)
```dockerfile
# Example: Building a Java application
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:resolve        # Download dependencies
COPY src ./src
RUN mvn package -DskipTests       # Compile and package

FROM openjdk:17-jre-slim
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Stage 3: Test (Automated Quality Gates)
```bash
# Unit Tests — Test individual functions
mvn test
# Result: 247/247 tests passed ✅ (12 seconds)

# Integration Tests — Test component interactions
mvn verify -P integration-tests
# Result: 89/89 tests passed ✅ (2 minutes 30 seconds)

# End-to-End Tests — Test complete user flows
mvn verify -P e2e-tests
# Result: 45/45 tests passed ✅ (8 minutes)

# Code Coverage
jacoco:report
# Result: 87% code coverage ✅ (above 80% threshold)
```

### Stage 4: Security Scan
```bash
# Dependency vulnerability scan
trivy image bank-app:v2.3.1
# Result: 0 critical, 0 high, 2 low vulnerabilities ✅

# Static Application Security Testing (SAST)
sonar-scanner
# Result: 0 bugs, 0 vulnerabilities, 3 code smells ⚠️

# Secret scanning
trufflehog filesystem --directory ./src
# Result: No secrets detected ✅
```

### Stage 5: Artifact Creation
```bash
# Push to Docker Registry
docker tag bank-app:v2.3.1 registry.bank.com/bank-app:v2.3.1
docker push registry.bank.com/bank-app:v2.3.1

# Tag the Git commit
git tag -a v2.3.1 -m "Release v2.3.1 - NEFT limit feature"
git push origin v2.3.1
```

---

## 🧪 Testing Pyramid

```
          ╱╲
         ╱  ╲
        ╱ E2E╲         Few (10-20 tests)
       ╱      ╲        Slow (minutes), Expensive
      ╱────────╲       Test complete user flows
     ╱ Integr.  ╲     Moderate (50-100 tests)
    ╱            ╲    Medium speed, Medium cost
   ╱──────────────╲   Test component interactions
  ╱    Unit Tests   ╲ Many (hundreds/thousands)
 ╱                    ╲ Fast (milliseconds), Cheap
╱──────────────────────╲  Test individual functions
```

### Example: Payment Service Test Suite
```java
// Unit Test — Fast, isolated, tests one function
@Test
public void testCalculateInterest() {
    double result = InterestCalculator.calculate(100000, 0.08, 365);
    assertEquals(8000.00, result, 0.01);
}

// Integration Test — Tests multiple components together
@Test
public void testPaymentEndToEnd() {
    Account source = accountService.createAccount("12345");
    Account target = accountService.createAccount("67890");
    
    Transaction tx = paymentService.initiateTransfer(
        source.getId(), target.getId(), 5000.00, "NEFT"
    );
    
    assertEquals("COMPLETED", tx.getStatus());
    assertEquals(95000.00, source.getBalance());
    assertEquals(105000.00, target.getBalance());
}

// E2E Test — Tests the entire application through the API
@Test
public void testNEFTTransferViaAPI() {
    Response response = given()
        .contentType("application/json")
        .body("{ \"from\": \"12345\", \"to\": \"67890\", \"amount\": 5000 }")
    .when()
        .post("/api/v1/transfer/neft")
    .then()
        .statusCode(200)
        .body("status", equalTo("COMPLETED"))
        .body("reference", notNullValue())
        .extract().response();
}
```

---

## 📊 Code Quality Metrics

| Metric | Threshold | What It Measures |
|--------|-----------|-----------------|
| **Test Coverage** | ≥ 80% | % of code covered by tests |
| **Code Duplication** | ≤ 3% | Duplicate code blocks |
| **Cyclomatic Complexity** | ≤ 10 per method | How complex a method is |
| **Technical Debt** | ≤ 5% | Time needed to fix all issues |
| **Build Time** | ≤ 10 minutes | How long CI pipeline takes |
| **Test Pass Rate** | 100% | All tests must pass |

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Preventing a Production Outage with CI
**Context:** A developer accidentally introduces a null pointer exception in the fund transfer service. Without CI, this would go to production and crash every transfer.

**The CI Save:**
```bash
# Developer pushes code at 3:00 PM
$ git push origin feature/transfer-enhancement

# CI Pipeline starts automatically:
[3:00:15] Pipeline #2847 started
[3:00:18] Building application... 
[3:00:45] Build successful ✅
[3:00:47] Running unit tests...
[3:01:15] ❌ TEST FAILED: testFundTransfer
         NullPointerException at TransferService.java:142
         "Cannot invoke getBalance() on null account"
[3:01:16] Pipeline FAILED ❌
[3:01:17] Notification sent to developer via Slack

# Developer sees the failure immediately
$ git log --oneline -1
abc123 feat: add transfer enhancement

# Developer fixes the bug
$ git commit -m "fix: handle null account in transfer validation"
$ git push origin feature/transfer-enhancement

# CI Pipeline runs again:
[3:15:00] Pipeline #2848 started
[3:15:30] All 247 tests passed ✅
[3:16:00] Pipeline SUCCESS ✅

# Total time from bug introduction to fix: 15 minutes
# Without CI: Bug would have reached production and affected customers
```

### Scenario 2: Automated Compliance Testing
**Context:** Every code change must pass regulatory compliance checks before deployment.

**CI Pipeline with Compliance Gates:**
```yaml
# .gitlab-ci.yml - Banking compliance pipeline
stages:
  - build
  - unit-test
  - compliance-test
  - security-scan
  - integration-test
  - staging-deploy

compliance-test:
  stage: compliance-test
  script:
    # RBI Transaction Limits
    - python tests/compliance/rbi_transaction_limits.py
    # Result: All transaction limits within regulatory bounds ✅
    
    # PCI-DSS Credit Card Data Handling
    - python tests/compliance/pci_dss_card_data.py
    # Result: Card data encrypted at rest and in transit ✅
    
    # GDPR Data Privacy
    - python tests/compliance/gdpr_data_privacy.py
    # Result: Customer PII properly masked ✅
    
    # SOX Financial Controls
    - python tests/compliance/sox_financial_controls.py
    # Result: All financial transactions have audit trail ✅
  artifacts:
    reports:
      compliance: compliance-report.json
```

### Scenario 3: Database Migration Safety
**Context:** A bank needs to add a new column to the transactions table for GST calculation.

**CI-Protected Migration:**
```sql
-- Migration file: V23__add_gst_column.sql
ALTER TABLE transactions 
ADD COLUMN gst_amount DECIMAL(10,2) DEFAULT 0.00;

ALTER TABLE transactions 
ADD COLUMN gst_rate DECIMAL(5,4) DEFAULT 0.0000;

-- Backfill existing transactions
UPDATE transactions 
SET gst_amount = amount * 0.18,
    gst_rate = 0.1800
WHERE transaction_date >= '2026-04-01'
  AND gst_amount = 0.00;
```

**CI Validation:**
```bash
# CI runs migration on a copy of production schema
[Stage: Migration Test]
✅ Migration applies successfully
✅ Rollback test: migration reverts cleanly
✅ Data integrity check: no orphaned records
✅ Performance test: query execution time < 50ms
✅ Schema validation: new columns exist with correct types

# Only then is the migration allowed to proceed
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: End-to-End Loan Application Processing Pipeline

**Context:** Build CI pipeline for a loan application system that handles personal, home, and business loans.

```yaml
# Complete CI Pipeline for Loan Service
# File: Jenkinsfile

pipeline {
    agent any
    
    stages {
        // Stage 1: Compile Java application
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        // Stage 2: Unit Tests (847 tests)
        stage('Unit Tests') {
            steps {
                sh 'mvn test -Dtest=LoanCalculationTest'
                sh 'mvn test -Dtest=InterestRateTest'
                sh 'mvn test -Dtest=EMICalculationTest'
                sh 'mvn test -Dtest=EligibilityCheckTest'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                }
            }
        }
        
        // Stage 3: Integration Tests (156 tests)
        stage('Integration Tests') {
            steps {
                sh 'docker-compose -f docker-compose.test.yml up -d db redis'
                sh 'mvn verify -P integration-tests'
                sh 'docker-compose -f docker-compose.test.yml down'
            }
        }
        
        // Stage 4: Compliance Tests (45 tests)
        stage('Compliance') {
            steps {
                // RBI Interest Rate Cap Check
                sh 'python tests/compliance/rbi_interest_rates.py'
                # Personal loan: Max 15% p.a.
                # Home loan: Max 12% p.a.
                # Business loan: Max 16% p.a.
                
                // EMI Calculation Validation
                sh 'python tests/compliance/emi_calculation.py'
                # EMI = P * r * (1+r)^n / ((1+r)^n - 1)
                # Verify against RBI formula
                
                // Foreclosure Charges Check
                sh 'python tests/compliance/foreclosure_charges.py'
                # Personal loan: Max 2% foreclosure
                # Home loan: Max 0% (RBI mandate)
            }
        }
        
        // Stage 5: Security Scan
        stage('Security') {
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE'
                sh 'sonar-scanner -Dsonar.qualitygate.wait=true'
            }
        }
        
        // Stage 6: Performance Test
        stage('Performance') {
            steps {
                sh 'jmeter -n -t tests/performance/loan-load-test.jmx \
                    -l results.jtl -e -o report/'
                # Results:
                # - Throughput: 500 req/sec
                # - P95 latency: 230ms
                # - Error rate: 0.001%
            }
        }
    }
}
```

**Test Results Summary:**
```
┌─────────────────────────────────────────────────────────────┐
│  LOAN SERVICE CI PIPELINE RESULTS                           │
├─────────────────────────────────────────────────────────────┤
│  Stage 1: Build         ✅ Passed     (1m 15s)             │
│  Stage 2: Unit Tests    ✅ 847/847    (3m 45s)             │
│  Stage 3: Integration   ✅ 156/156    (8m 20s)             │
│  Stage 4: Compliance    ✅ 45/45      (2m 10s)             │
│  Stage 5: Security      ✅ 0 CVEs     (1m 30s)             │
│  Stage 6: Performance   ✅ 500 req/s  (5m 00s)             │
├─────────────────────────────────────────────────────────────┤
│  TOTAL: 1,048 tests passed | Duration: 22 minutes          │
│  Coverage: 89% | Quality Gate: PASSED                      │
└─────────────────────────────────────────────────────────────┘
```

### E2E Example 2: API Gateway CI Pipeline with Contract Testing

**Context:** API Gateway handles 50+ microservices. Must validate API contracts before deployment.

```yaml
# Contract Testing Pipeline
# File: .github/workflows/api-gateway.yml

name: API Gateway CI
on: [push, pull_request]

jobs:
  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Start Consumer Services
        run: |
          docker-compose -f consumers/docker-compose.yml up -d
          
      - name: Run Contract Tests (Pact)
        run: |
          # Payment service expects:
          # POST /api/v1/payments -> 200 + {transactionId, status}
          # GET /api/v1/payments/{id} -> 200 + {status, amount, timestamp}
          
          # Account service expects:
          # GET /api/v1/accounts/{id}/balance -> 200 + {balance, currency}
          
          # All 234 contract tests passed ✅
          npm run test:contracts
          
      - name: API Schema Validation
        run: |
          # Validate OpenAPI 3.0 schema
          # Check backward compatibility
          # Verify versioning (v1, v2, v3)
          swagger-cli validate api-spec.yaml
          
      - name: Breaking Change Detection
        run: |
          # Detect API breaking changes
          # Check: removed endpoints, changed response format
          # Result: 0 breaking changes ✅
          npm run detect-breaking-changes
```

### E2E Example 3: Database Migration CI Pipeline

**Context:** Safely migrate customer data schema across 3 environments.

```yaml
# Migration Pipeline
# File: .gitlab-ci.yml

migrate-dev:
  stage: migrate
  script:
    # Apply migrations to dev database
    - flyway -url=jdbc:postgresql://dev-db:5432/banking \
        -user=$DEV_USER -password=$DEV_PASS migrate
    # Migrations applied: V24, V25, V26
    # Duration: 45 seconds
    
    # Run data integrity checks
    - python scripts/verify_migration.py --env dev
    # All 1,247 accounts: balanced ✅
    # All 45,892 transactions: linked ✅
    # No orphaned records: confirmed ✅

migrate-staging:
  stage: migrate
  script:
    # Apply to staging (copy of production schema)
    - flyway -url=jdbc:postgresql://staging-db:5432/banking \
        -user=$STAGING_USER -password=$STAGING_PASS migrate
    
    # Performance test migration
    - python scripts/migration_performance.py
    # Large table ALTER: 2.3 seconds (threshold: 10s) ✅
    # Index rebuild: 15 seconds (threshold: 60s) ✅

migrate-prod:
  stage: migrate
  when: manual  # Requires approval
  script:
    # Backup production database first
    - pg_dump -Fc banking > backup_$(date +%Y%m%d_%H%M%S).dump
    
    # Apply migrations
    - flyway -url=jdbc:postgresql://prod-db:5432/banking \
        -user=$PROD_USER -password=$PROD_PASS migrate
    
    # Verify production data integrity
    - python scripts/verify_production.py
    # All 2.3 million accounts: balanced ✅
    # All 12.8 million transactions: linked ✅
    # Zero data loss: confirmed ✅
```

---

## 📋 Interview Questions

### Q1: What is the difference between build and test stages in CI?
**Answer:** The **build stage** compiles source code into executable artifacts (JARs, Docker images). If the build fails, it means the code doesn't compile — typically a syntax error or missing dependency. The **test stage** runs automated tests against the built artifact to verify correctness, performance, and security. A build can succeed but tests can fail (the code compiles but doesn't work correctly). In CI, both stages must pass before code can be merged.

### Q2: What is a "failing fast" pipeline and why does it matter?
**Answer:** A "failing fast" pipeline runs the fastest, cheapest checks first so developers get feedback immediately. 

For example: 

(1) Linting (seconds) → (2) Unit tests (minutes) → (3) Integration tests (10+ minutes) → (4) E2E tests (30+ minutes). 

If linting fails, you know in 10 seconds instead of waiting 30 minutes for E2E tests. This saves developer time and CI resources.

### Q3: How do you handle flaky tests in CI?
**Answer:** Flaky tests are tests that sometimes pass and sometimes fail without code changes. 

Strategies: 

(1) **Quarantine** — move flaky tests to a separate pipeline that doesn't block merging. 

(2) **Retry** — automatically retry failed tests up to 3 times. 

(3) **Root cause analysis** — most flaky tests have real issues (race conditions, timing dependencies, external service dependencies). 

(4) **Delete** — if a test is inherently unreliable, remove it and replace with a more reliable test.

### Q4: What is test coverage and is 100% always the goal?
**Answer:** Test coverage measures the percentage of code executed by tests. While 100% coverage sounds ideal, it's not always practical or beneficial. Some code (configuration, boilerplate) doesn't need testing. The goal should be **meaningful coverage** — ensure critical business logic (interest calculations, transaction processing) has 100% coverage, while infrastructure code can have lower coverage. Most banks target 80%+ overall coverage.

### Q5: How does CI help with regulatory compliance?
**Answer:** CI provides: 

(1) **Automated evidence** — every pipeline run produces logs showing what was tested and when. 

(2) **Enforced standards** — compliance checks run automatically on every commit (PCI-DSS, GDPR, RBI). 

(3) **Immutable audit trail** — Git history + pipeline logs show exactly who changed what, when, and why. 

(4) **Consistent process** — the same checks run the same way every time, eliminating human error in compliance verification.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| CI | Merge frequently, test automatically |
| Build | Compile code into artifacts (Docker images) |
| Test Pyramid | Many unit tests, fewer integration, few E2E |
| Code Quality | Enforce metrics as pipeline gates |
| Compliance | Automated regulatory checks in CI |
| Banking Relevance | Prevent outages, maintain audit trails |

**Next:** [04-Continuous-Delivery.md](./04-Continuous-Delivery.md) — Learn CD and deployment strategies.
