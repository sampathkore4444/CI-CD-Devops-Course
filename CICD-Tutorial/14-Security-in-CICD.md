# 14 — Security in CI/CD: DevSecOps

> **Goal:** Understand how security is integrated into every stage of the CI/CD pipeline — "Shift Left" security.

---

## 🔍 What is DevSecOps?

**DevSecOps** = Development + Security + Operations. It means **security is everyone's responsibility** and is integrated into every stage of the pipeline, not bolted on at the end.

```
Traditional (Dev + Ops):
  Dev: "I built the feature"
  Ops: "I deployed it"
  Security: "Wait, there's a vulnerability!"
  → Security is an afterthought, blocks releases

DevSecOps:
  Dev: "I built the feature"
  Pipeline: "Security scan: 0 vulnerabilities ✅"
  Security: "Automated checks passed, no manual review needed"
  → Security is built-in, continuous, automated
```

---

## 🏗️ Security at Every Pipeline Stage

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  CODE    │──▶│  BUILD   │──▶│  TEST    │──▶│  STAGE   │──▶│  PROD    │
│          │   │          │   │          │   │          │   │          │
│ Pre-     │   │ SAST     │   │ DAST     │   │ Image    │   │ Runtime  │
│ commit   │   │ Dep scan │   │ Pen test │   │ scan     │   │ monitor  │
│ hooks    │   │ License  │   │ Fuzz test│   │ Sign     │   │ WAF      │
│ Secret   │   │ check    │   │          │   │ Verify   │   │ RASP     │
│ scan     │   │          │   │          │   │          │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

---

## 🔒 Security Tools & Techniques

### 1. Secret Scanning (Pre-Commit)
```bash
# TruffleHog - scan for secrets before they reach Git
$ trufflehog filesystem --directory ./src
Found unverified result: 🐷🔑
Detector Type: AWS Access Key
Decoder Type: Plain
Raw result: AKIAIOSFODNN7EXAMPLE
File: src/config/AwsConfig.java
Line: 42
```

**Git Pre-Commit Hook:**
```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "🔒 Running security checks..."

# Check for secrets
if trufflehog filesystem --directory ./src --fail 2>/dev/null; then
    echo "❌ Secrets detected! Fix before committing."
    exit 1
fi

# Check for hardcoded passwords
if grep -rn "password\s*=\s*\"" ./src --include="*.java" --include="*.py"; then
    echo "❌ Hardcoded password detected! Use environment variables."
    exit 1
fi

echo "✅ Security checks passed"
```

### 2. SAST (Static Application Security Testing)
```bash
# SonarQube - analyze source code for vulnerabilities
$ sonar-scanner \
  -Dsonar.projectKey=payment-service \
  -Dsonar.sources=./src \
  -Dsonar.host.url=http://sonar.bank.com

# Results:
# Bugs: 0
# Vulnerabilities: 0
# Security Hotspots: 2 (review needed)
# Code Smells: 5
# Coverage: 87%
```

### 3. Dependency Scanning
```bash
# OWASP Dependency Check - scan for vulnerable libraries
$ dependency-check --project "Payment Service" \
  --scan ./target/*.jar \
  --out ./report

# Results:
# Dependency: log4j-core-2.14.0.jar
# Vulnerability: CVE-2021-44228 (Log4Shell)
# Severity: CRITICAL (10.0)
# CVSS Score: 10.0
# Recommendation: Upgrade to log4j-core-2.17.1

# CI Pipeline fails if CRITICAL vulnerabilities found
```

### 4. Container Image Scanning
```bash
# Trivy - scan Docker image for vulnerabilities
$ trivy image --severity HIGH,CRITICAL registry.bank.com/payment:v2.3.1

registry.bank.com/payment:v2.3.1 (debian 11.7)
Total: 2 (CRITICAL: 0, HIGH: 1, MEDIUM: 1)

┌──────────────┬────────────────┬──────────┬─────────────┬──────────────────┐
│   Library    │ Vulnerability  │ Severity │ Fixed       │ Title            │
├──────────────┼────────────────┼──────────┼─────────────┼──────────────────┤
│ libcurl4     │ CVE-2026-1234  │ HIGH     │ 7.74.1      │ Buffer overflow  │
│ openssl      │ CVE-2026-5678  │ MEDIUM   │ 1.1.1x      │ Info disclosure  │
└──────────────┴────────────────┴──────────┴─────────────┴──────────────────┘
```

### 5. DAST (Dynamic Application Security Testing)
```bash
# OWASP ZAP - scan running application for vulnerabilities
$ zap-cli quick-scan \
  --self-contained \
  --start-options '-config api.key=xxx' \
  https://staging.bank.com/api/v1/payments

# Results:
# High: SQL Injection in /api/v1/payments
# Medium: XSS in /api/v1/notifications
# Low: Information disclosure in response headers
```

### 6. Infrastructure as Code Scanning
```bash
# Checkov - scan Terraform/K8s manifests for misconfigurations
$ checkov -d ./terraform/ --framework terraform

# Results:
# ✅ PASSED: aws_s3_bucket.server_access_logging_enabled
# ❌ FAILED: aws_eks_cluster.encryption_config (CRITICAL)
#    Resource: aws_eks_cluster.banking_cluster
#    Issue: EKS cluster not configured with envelope encryption
#    Fix: Add encryption_config block with KMS key ARN

# ❌ FAILED: aws_security_group.open_to_world (HIGH)
#    Resource: aws_security_group.web_sg
#    Issue: Security group allows 0.0.0.0/0 on port 22
#    Fix: Restrict SSH access to internal CIDR only
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Preventing Data Leakage in Source Code
**Context:** A developer accidentally commits AWS credentials to Git. DevSecOps catches it.

**Automated Response:**
```bash
# Developer pushes code with AWS key
$ git push origin feature/new-api

# Pre-commit hook runs:
🔒 Running security checks...
❌ Secrets detected! Fix before committing.

# Even if pre-commit is bypassed, CI catches it:
$ git push origin feature/new-api
[Pipeline] Secret scan running...
[Pipeline] ❌ CRITICAL: AWS Access Key found in src/config/AwsConfig.java:42
[Pipeline] Pipeline FAILED - Secret detected
[Pipeline] Developer notified via Slack

# If somehow it reaches production:
[Runtime] Falco detects: Outbound connection to AWS API from unexpected IP
[Runtime] Alert: Possible credential misuse
[Response] Credentials automatically rotated via Vault
[Response] Audit log created with full trace
```

### Scenario 2: Compliance-Driven Security Gates
**Context:** Every deployment must pass PCI-DSS, GDPR, and RBI compliance checks.

**Security Gate Pipeline:**
```yaml
# .gitlab-ci.yml - Security gates
stages:
  - security-scan
  - compliance-check
  - deploy

sast-scan:
  stage: security-scan
  script:
    - sonar-scanner -Dsonar.qualitygate.wait=true
  # Quality gate must pass before proceeding

dependency-check:
  stage: security-scan
  script:
    - dependency-check --project "$CI_PROJECT_NAME" --scan . --out report
    - python scripts/check-vulnerabilities.py --max-critical 0 --max-high 0

container-scan:
  stage: security-scan
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE

iac-scan:
  stage: security-scan
  script:
    - checkov -d ./terraform/ --framework terraform --soft-fail-on MEDIUM
    - checkov -d ./k8s/ --framework kubernetes --hard-fail-on HIGH

# Compliance checks
pci-dss-check:
  stage: compliance-check
  script:
    - python compliance/pci_dss_validator.py
    # Checks: encryption at rest, network segmentation, access controls

gdpr-check:
  stage: compliance-check
  script:
    - python compliance/gdpr_validator.py
    # Checks: PII handling, data retention, consent management

rbi-check:
  stage: compliance-check
  script:
    - python compliance/rbi_validator.py
    # Checks: transaction limits, audit trails, data localization
```

### Scenario 3: Runtime Security Monitoring
**Context:** Detect and respond to attacks in real-time.

**Falco Runtime Security Rules:**
```yaml
# falco-rules.yaml
- rule: Unauthorized process in payment container
  desc: Detect unexpected processes in payment pods
  condition: >
    spawned_process and container and 
    k8s.ns.name = "production" and 
    k8s.pod.label.app = "payment-service" and
    not proc.name in (payment-service, java, bash)
  output: >
    Unexpected process in payment pod 
    (user=%user.name pod=%k8s.pod.name container=%container.name 
     proc=%proc.name parent=%proc.pname)
  priority: CRITICAL

- rule: Outbound connection to external IP
  desc: Detect payment service connecting to external IPs
  condition: >
    outbound and container and 
    k8s.ns.name = "production" and 
    k8s.pod.label.app = "payment-service" and
    not fd.sip.name in (internal_ips)
  output: >
    Payment pod connecting to external IP 
    (pod=%k8s.pod.name dest=%fd.sip.name port=%fd.sport)
  priority: HIGH
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Complete Security Pipeline for Payment Service

**Context:** Implement comprehensive security scanning at every stage of the pipeline.

```yaml
# Security pipeline stages
# File: .gitlab-ci.yml

stages:
  - pre-commit
  - build
  - test
  - security
  - deploy

# Stage 1: Pre-commit secret scanning
pre-commit:
  stage: pre-commit
  script:
    - trufflehog filesystem --directory ./src --fail
    - gitleaks detect --source . --verbose
  # Blocks commits with secrets

# Stage 2: SAST during build
sast:
  stage: security
  script:
    - sonar-scanner \
        -Dsonar.projectKey=payment-service \
        -Dsonar.sources=./src \
        -Dsonar.qualitygate.wait=true
  # Quality gate: 0 bugs, 0 vulnerabilities

# Stage 3: Dependency scanning
dependency-check:
  stage: security
  script:
    - dependency-check \
        --project "Payment Service" \
        --scan ./target/*.jar \
        --out ./report \
        --failOnCVSS 7
  # Fails if any dependency has CVSS >= 7

# Stage 4: Container image scanning
container-scan:
  stage: security
  script:
    - trivy image \
        --exit-code 1 \
        --severity HIGH,CRITICAL \
        $REGISTRY/payment:$CI_COMMIT_SHA
  # Blocks deployment if HIGH/CRITICAL found

# Stage 5: IaC scanning
iac-scan:
  stage: security
  script:
    - checkov -d ./terraform/ --framework terraform --hard-fail-on HIGH
    - checkov -d ./k8s/ --framework kubernetes --hard-fail-on HIGH
  # Blocks deployment if IaC has HIGH issues

# Stage 6: DAST on staging
dast:
  stage: security
  script:
    - zap-cli quick-scan --self-contained https://staging.bank.com/api/v1/payments
  # Scans running application for runtime vulnerabilities
```

```bash
# Security scan results
$ gitlab-ci-pipeline --security-report

# Security Scan Report
# ════════════════════════════════════════════════════════════════════
# Stage          Tool          Result         Issues Found
# ──────────────────────────────────────────────────────────────────
# Pre-commit     TruffleHog    ✅ PASS        0 secrets
# Pre-commit     Gitleaks      ✅ PASS        0 secrets
# Build          SonarQube     ✅ PASS        0 bugs, 0 vulns
# Dependencies   OWASP DC      ✅ PASS        0 CVEs (CVSS < 7)
# Container      Trivy         ✅ PASS        0 CRITICAL, 0 HIGH
# IaC            Checkov       ✅ PASS        0 HIGH issues
# DAST           OWASP ZAP     ✅ PASS        0 high-risk issues
# ════════════════════════════════════════════════════════════════════
# OVERALL: PASSED ✅ (All 7 security stages clean)
```

### E2E Example 2: Incident Response for Vulnerable Dependency

**Context:** New critical vulnerability (Log4Shell variant) discovered in log4j library.

```bash
# Hour 1: Discovery
$ trivy image --severity CRITICAL registry.bank.com/prod/payment:v2.4.0
# CRITICAL: CVE-2026-9999 (Log4Shell variant)
# Package: log4j-core-2.14.0.jar
# CVSS Score: 10.0

# Scan all services
$ for service in payment account gateway loan notification; do
    echo "Scanning $service..."
    trivy image --severity CRITICAL registry.bank.com/prod/$service:latest 2>&1 | grep -q "CRITICAL" && echo "  ❌ VULNERABLE" || echo "  ✅ CLEAN"
done
# Scanning payment... ❌ VULNERABLE
# Scanning account... ❌ VULNERABLE
# Scanning gateway... ✅ CLEAN (uses Logback, not Log4j)
# Scanning loan... ❌ VULNERABLE
# Scanning notification... ✅ CLEAN

# Hour 2: Remediation
# Update pom.xml in vulnerable services
$ sed -i 's/log4j-core-2.14.0/log4j-core-2.17.1/' payment-service/pom.xml
$ sed -i 's/log4j-core-2.14.0/log4j-core-2.17.1/' account-service/pom.xml
$ sed -i 's/log4j-core-2.14.0/log4j-core-2.17.1/' loan-service/pom.xml

# Rebuild and scan
$ for service in payment account loan; do
    docker build -t registry.bank.com/prod/$service:patched services/$service/
    docker push registry.bank.com/prod/$service:patched
    trivy image --severity CRITICAL registry.bank.com/prod/$service:patched
done
# All: 0 CRITICAL ✅

# Hour 3: Deploy patched versions
$ for service in payment account loan; do
    kubectl set image deployment/$service $service=registry.bank.com/prod/$service:patched -n production
    kubectl rollout status deployment/$service -n production --timeout=300s
done

# Hour 4: Verification
$ for service in payment account gateway loan notification; do
    trivy image --severity CRITICAL registry.bank.com/prod/$service:latest 2>&1 | grep -q "CRITICAL" && echo "❌ $service" || echo "✅ $service"
done
# ✅ payment
# ✅ account
# ✅ gateway
# ✅ loan
# ✅ notification

# Compliance report generated
$ python scripts/generate-security-report.py
# Report: CVE-2026-9999 remediation
# Services affected: 3
# Time to remediate: 4 hours
# Customer impact: None
# Regulatory notification: Not required (no data breach)
```

### E2E Example 3: Runtime Security Monitoring with Falco

**Context:** Detect and respond to attacks in real-time on production cluster.

```yaml
# Falco rules for banking
# File: falco/banking-rules.yaml

- rule: Unauthorized process in payment pod
  desc: Detect unexpected processes in payment containers
  condition: >
    spawned_process and container and
    k8s.ns.name = "production" and
    k8s.pod.label.app = "payment-service" and
    not proc.name in (payment-service, java, bash, curl)
  output: >
    Unauthorized process in payment pod
    (user=%user.name pod=%k8s.pod.name proc=%proc.name parent=%proc.pname)
  priority: CRITICAL

- rule: Sensitive file access in payment pod
  desc: Detect access to sensitive files
  condition: >
    open_read and container and
    k8s.ns.name = "production" and
    k8s.pod.label.app = "payment-service" and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/passwd or
     fd.name contains .env or
     fd.name contains credentials)
  output: >
    Sensitive file accessed in payment pod
    (pod=%k8s.pod.name file=%fd.name)
  priority: HIGH

- rule: Outbound connection to external IP
  desc: Detect payment service connecting to unexpected external IPs
  condition: >
    outbound and container and
    k8s.ns.name = "production" and
    k8s.pod.label.app = "payment-service" and
    not fd.sip.name in (internal_ips)
  output: >
    Unexpected outbound connection
    (pod=%k8s.pod.name dest=%fd.sip.name port=%fd.sport)
  priority: HIGH
```

```bash
# Falco detects suspicious activity
[Alert] CRITICAL: Unauthorized process in payment pod
  user=root pod=payment-abc123 proc=nc parent=bash
  Time: 2026-09-04 02:15:00 UTC
  Action: Alert sent to SOC team

# SOC response
$ kubectl exec payment-abc123 -- ps aux | grep nc
# root     1234  0.0  0.0  nc -l -p 4444
# Suspicious! Netcat listening on port 4444

# Isolate pod
$ kubectl cordon node-prod-01  # Prevent new pods on this node
$ kubectl delete pod payment-abc123 -n production  # Kill compromised pod
$ kubectl get pods -l app=payment-service -n production
# New pod created automatically (self-healing)
# payment-service-new-xyz 1/1 Running

# Forensics
$ kubectl logs payment-abc123 --previous | grep nc
# 2026-09-04 02:14:55 INFO  PaymentService - Starting
# 2026-09-04 02:14:58 WARN  - Process spawned: bash -> nc -l -p 4444

# Root cause: Compromised base image (unauthorized nc installed)
# Remediation: Updated base image, removed nc, rebuilt all services
# Time to detect: 30 seconds
# Time to respond: 2 minutes
# Customer impact: None
```

---

## 📋 Interview Questions

### Q1: What is "Shift Left" security and why is it important?
**Answer:** "Shift Left" means moving security checks earlier in the development lifecycle (to the left on a timeline). Instead of security testing at the end (right), it happens during coding and CI. 

Benefits: 

(1) **Cheaper fixes** — a vulnerability found during coding costs $10 to fix; in production, it costs $10,000+. 

(2) **Faster feedback** — developers fix issues immediately, not weeks later. 

(3) **Continuous security** — every commit is scanned, not just pre-release.

### Q2: What is the difference between SAST and DAST?
**Answer:** **SAST** (Static) analyzes **source code** without running it. It finds issues like SQL injection patterns, hardcoded secrets, and insecure code. **DAST** (Dynamic) tests the **running application** by sending malicious inputs. It finds runtime issues like authentication bypass, server misconfigurations, and API vulnerabilities. Banks use both: SAST in CI for quick feedback, DAST in staging for runtime testing.

### Q3: How do you implement a zero-trust security model in CI/CD?
**Answer:** Zero-trust means "never trust, always verify." 

In CI/CD: 

(1) **Code signing** — verify every commit is signed. 

(2) **Image signing** — verify Docker images before deployment. 

(3) **Mutual TLS** — all service-to-service communication encrypted. 

(4) **RBAC** — least-privilege access for pipeline tools. 

(5) **Secret rotation** — regular credential rotation. 

(6) **Network policies** — restrict pod-to-pod communication. 

(7) **Audit logging** — every action logged and reviewed.

### Q4: What is software composition analysis (SCA) and why is it critical?
**Answer:** SCA analyzes third-party libraries and dependencies for known vulnerabilities. Most application code is open-source dependencies (70-90%). Example: Log4Shell (CVE-2021-44228) affected thousands of banking applications. SCA tools (OWASP Dependency Check, Snyk, Black Duck) scan dependencies against vulnerability databases. Banks require SCA because a single vulnerable dependency can expose the entire application.

### Q5: How do you handle security in a microservices architecture?
**Answer:** Each microservice needs: 

(1) **Service mesh** — Istio/Linkerd for mTLS, traffic policies. 

(2) **API gateway** — centralized authentication, rate limiting. 

(3) **Network policies** — restrict pod-to-pod communication. 

(4) **Secret management** — per-service credentials via Vault. 

(5) **Image scanning** — each service's image scanned independently. 

(6) **Runtime security** — Falco for each pod. 

(7) **Centralized logging** — aggregate all security events.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| DevSecOps | Security is everyone's responsibility |
| SAST | Static code analysis for vulnerabilities |
| DAST | Dynamic testing of running applications |
| SCA | Third-party dependency vulnerability scanning |
| Secret Scanning | Prevent credentials from reaching Git |
| Runtime Security | Detect attacks in real-time |
| Banking Relevance | PCI-DSS, GDPR, RBI compliance |

**Next:** [15-Banking-CICD-Mastery.md](./15-Banking-CICD-Mastery.md) — Complete end-to-end banking CI/CD examples.
