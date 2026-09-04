# 07 — Docker Registry: Secure Image Storage

> **Goal:** Understand Docker registries — where images live, how they're managed, and security in banking.

---

## 🔍 What is a Docker Registry?

A **Docker Registry** is a storage and distribution system for Docker images. Think of it as a **GitHub for Docker images** — you push images to it and pull them when needed.

### Registry Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CI PIPELINE                           │
│                                                         │
│  Build Image ──▶ Tag Image ──▶ Push to Registry         │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                 DOCKER REGISTRY                          │
│            (registry.bank.com)                           │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ payment  │  │ account  │  │ notify   │              │
│  │ :v2.3.1  │  │ :v1.2.0  │  │ :v3.0.0  │              │
│  │ :v2.3.0  │  │ :v1.1.9  │  │ :v2.9.9  │              │
│  │ :latest  │  │ :latest  │  │ :latest  │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                         │
│  Features:                                              │
│  ✅ Image versioning                                    │
│  ✅ Access control (RBAC)                               │
│  ✅ Vulnerability scanning                              │
│  ✅ Audit logging                                       │
│  ✅ Image signing                                       │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                 KUBERNETES CLUSTER                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Worker   │  │ Worker   │  │ Worker   │              │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │              │
│  │          │  │          │  │          │              │
│  │ Pull ──▶ │  │ Pull ──▶ │  │ Pull ──▶ │              │
│  │ payment  │  │ payment  │  │ payment  │              │
│  │ :v2.3.1  │  │ :v2.3.1  │  │ :v2.3.1  │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ Types of Registries

| Registry | Type | Best For |
|----------|------|----------|
| **Docker Hub** | Public | Open-source projects |
| **Amazon ECR** | Cloud | AWS-based banking systems |
| **Google GCR** | Cloud | GCP-based banking systems |
| **Azure ACR** | Cloud | Azure-based banking systems |
| **Harbor** | Self-hosted | On-premise banking (most common) |
| **Nexus** | Self-hosted | Enterprise artifact management |

### Why Banks Use Private Registries

```
Docker Hub (Public):
  ❌ No access control
  ❌ No audit logs
  ❌ Images visible to everyone
  ❌ No vulnerability scanning
  ❌ Not compliant with banking regulations

Harbor/ECR (Private):
  ✅ Role-based access control (RBAC)
  ✅ Complete audit logging
  ✅ Images encrypted at rest
  ✅ Automatic vulnerability scanning
  ✅ Image signing and verification
  ✅ Retention policies
  ✅ Compliance with RBI, PCI-DSS, GDPR
```

---

## 🔧 Registry Operations

### Pushing Images
```bash
# Login to private registry
docker login registry.bank.com
Username: ci-pipeline
Password: ****

# Tag the image
docker tag payment-service:v2.3.1 registry.bank.com/banking/payment-service:v2.3.1

# Push to registry
docker push registry.bank.com/banking/payment-service:v2.3.1
# The push refers to repository [registry.bank.com/banking/payment-service]
# 5d2f8e: Pushed
# a1b2c3: Pushed
# v2.3.1: digest: sha256:abc123... size: 1234
```

### Pulling Images
```bash
# Kubernetes pulls images automatically
# But you can also pull manually:
docker pull registry.bank.com/banking/payment-service:v2.3.1
# v2.3.1: Pulling from banking/payment-service
# 5d2f8e: Pull complete
# a1b2c3: Pull complete
# Status: Downloaded newer image for registry.bank.com/banking/payment-service:v2.3.1
```

### Image Tagging Strategy
```bash
# Semantic versioning (recommended for banking)
docker tag payment-service registry.bank.com/banking/payment-service:2.3.1
docker tag payment-service registry.bank.com/banking/payment-service:2.3
docker tag payment-service registry.bank.com/banking/payment-service:2
docker tag payment-service registry.bank.com/banking/payment-service:latest

# Git SHA (for traceability)
docker tag payment-service registry.bank.com/banking/payment-service:abc123def

# Build number (for CI pipelines)
docker tag payment-service registry.bank.com/banking/payment-service:build-1847
```

---

## 🔒 Registry Security

### Image Scanning
```bash
# Scan image for vulnerabilities
$ trivy image registry.bank.com/banking/payment-service:v2.3.1

registry.bank.com/banking/payment-service:v2.3.1 (debian 11.7)
Total: 2 (UNKNOWN: 0, LOW: 1, MEDIUM: 1, HIGH: 0, CRITICAL: 0)

┌──────────────┬────────────────┬──────────┬───────────────────┬─────────────┬──────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Installed Version │ Fixed Version│        Title             │
├──────────────┼────────────────┼──────────┼───────────────────┼─────────────┼──────────────────────────┤
│ libcurl4     │ CVE-2026-1234  │ MEDIUM   │ 7.74.0            │ 7.74.1      │ Buffer overflow          │
│ openssl      │ CVE-2026-5678  │ LOW      │ 1.1.1w            │ 1.1.1x      │ Information disclosure   │
└──────────────┴────────────────┴──────────┴───────────────────┴─────────────┴──────────────────────────┘
```

### Content Trust (Image Signing)
```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Sign image when pushing
docker push registry.bank.com/banking/payment-service:v2.3.1
# You'll be prompted for a passphrase to sign the image

# Verify signature when pulling
docker pull registry.bank.com/banking/payment-service:v2.3.1
# Pulling signed image...
# Signature verified ✅
```

### Retention Policies
```yaml
# Harbor retention policy
retention_policy:
  rules:
    # Keep all tagged releases
    - action: retain
      tag_pattern: "v*"
      most_recent: 0  # Keep all versions
    
    # Keep only last 10 builds per branch
    - action: retain
      tag_pattern: "build-*"
      most_recent: 10
    
    # Delete untagged images older than 7 days
    - action: delete
      tag_pattern: ""
      days: 7
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Secure Image Promotion Pipeline
**Context:** Images must pass through multiple stages before reaching production, each with different security requirements.

**Image Promotion Flow:**
```bash
# Stage 1: Development (every commit)
$ docker push registry.bank.com/dev/payment-service:abc123
# ✅ Basic scan: 0 critical vulnerabilities
# ❌ NOT allowed in staging or production

# Stage 2: Staging (when PR is merged to develop)
$ docker tag registry.bank.com/dev/payment-service:abc123 \
             registry.bank.com/staging/payment-service:v2.3.0-rc1
$ docker push registry.bank.com/staging/payment-service:v2.3.0-rc1
# ✅ Deep scan: 0 critical, 0 high vulnerabilities
# ✅ License compliance check passed
# ✅ Signed image
# ❌ NOT allowed in production yet

# Stage 3: Production (when release is tagged)
$ docker tag registry.bank.com/staging/payment-service:v2.3.0-rc1 \
             registry.bank.com/prod/payment-service:v2.3.0
$ docker push registry.bank.com/prod/payment-service:v2.3.0
# ✅ Final scan: 0 critical, 0 high, 0 medium vulnerabilities
# ✅ Signed by release manager
# ✅ Approved by security team
# ✅ Audit log recorded
# ✅ Available for production deployment
```

### Scenario 2: Incident Response — Vulnerable Image Discovery
**Context:** A new critical vulnerability (CVE) is discovered in a library used by 15 banking microservices.

**Automated Response:**
```bash
# 1. Security team discovers CVE-2026-9999 in OpenSSL
# 2. Automated scan of all production images

$ for image in $(curl -s https://harbor.bank.com/api/v2.0/projects/banking/repositories | jq -r '.[].name'); do
    echo "Scanning: $image"
    trivy image --severity CRITICAL registry.bank.com/$image:latest
done

# Results:
# payment-service: ❌ VULNERABLE (OpenSSL 1.1.1w)
# account-service: ❌ VULNERABLE (OpenSSL 1.1.1w)
# notification-service: ✅ CLEAN (uses Alpine, no OpenSSL)
# gateway-service: ❌ VULNERABLE (OpenSSL 1.1.1w)
# ... (12 more services)

# 3. Automated rebuild triggered
# 4. All vulnerable images rebuilt with patched OpenSSL
# 5. New images pushed to registry
# 6. Kubernetes rolls out new images automatically
# 7. Timeline: 4 hours (vs 2-3 weeks manually)
```

### Scenario 3: Audit Trail for Regulators
**Context:** RBI auditor requests complete traceability for production image `payment-service:v2.3.0`.

**Registry Audit Query:**
```bash
$ harbor-cli image history registry.bank.com/banking/payment-service:v2.3.0

Image: registry.bank.com/banking/payment-service:v2.3.0
Digest: sha256:abc123def456...
Created: 2026-09-01 14:30:00 IST
Created By: ci-pipeline@bank.com (automated)

Build History:
┌──────────────┬─────────────────────┬──────────────────────────────┐
│    Layer     │      Command        │         Timestamp            │
├──────────────┼─────────────────────┼──────────────────────────────┤
│ Layer 1      │ FROM openjdk:17     │ 2026-09-01 14:25:00          │
│ Layer 2      │ RUN groupadd...     │ 2026-09-01 14:25:01          │
│ Layer 3      │ COPY app.jar        │ 2026-09-01 14:25:02          │
│ Layer 4      │ HEALTHCHECK...      │ 2026-09-01 14:25:03          │
└──────────────┴─────────────────────┴──────────────────────────────┘

Security Scan Results:
  Scanner: Trivy v0.45.0
  Scan Date: 2026-09-01 14:25:10
  Result: 0 CRITICAL, 0 HIGH, 1 MEDIUM, 3 LOW
  
Signatures:
  Signed By: release-manager@bank.com
  Signed At: 2026-09-01 14:28:00
  Certificate: valid until 2027-09-01
  
Access Log:
  2026-09-01 14:30:00 - Pull by k8s-prod-node-1 ✅
  2026-09-01 14:30:01 - Pull by k8s-prod-node-2 ✅
  2026-09-01 14:30:02 - Pull by k8s-prod-node-3 ✅
  2026-09-01 14:30:03 - Pull by k8s-prod-node-4 ✅
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Complete Image Lifecycle for Payment Service

**Context:** Track a payment service image from build to production deployment.

```bash
# Step 1: Developer pushes code
$ git push origin feature/instant-transfer

# Step 2: CI builds image
[Pipeline] docker build -t registry.bank.com/payment:abc123def .
[Pipeline] docker push registry.bank.com/payment:abc123def
# Image pushed to Harbor registry ✅

# Step 3: Harbor scans image automatically
[Harbor] Scanning registry.bank.com/payment:abc123def...
[Harbor] Result: 0 CRITICAL, 0 HIGH, 2 MEDIUM, 5 LOW
[Harbor] Scan completed in 45 seconds

# Step 4: Image promoted to staging
$ docker tag registry.bank.com/payment:abc123def \
             registry.bank.com/staging/payment:abc123def
$ docker push registry.bank.com/staging/payment:abc123def
# Staging deployment triggered

# Step 5: Staging validation passes
$ kubectl exec payment-staging -- ./validate.sh
✅ All 247 unit tests passed
✅ All 89 integration tests passed
✅ Performance: P99 latency 180ms (threshold: 500ms)

# Step 6: Image promoted to production
$ docker tag registry.bank.com/staging/payment:abc123def \
             registry.bank.com/prod/payment:v2.4.0
$ docker push registry.bank.com/prod/payment:v2.4.0
# Production deployment triggered

# Step 7: Verify in production
$ kubectl get pods -l app=payment -n production
# NAME                    READY   STATUS    IMAGE
# payment-abc123          1/1     Running   registry.bank.com/prod/payment:v2.4.0
# payment-def456          1/1     Running   registry.bank.com/prod/payment:v2.4.0
# payment-ghi789          1/1     Running   registry.bank.com/prod/payment:v2.4.0

# Image audit trail:
$ harbor-cli image history registry.bank.com/prod/payment:v2.4.0
# Created: 2026-09-04 14:30:00
# Author: CI Pipeline (automated)
# Git SHA: abc123def
# Scanner: Trivy v0.45.0
# Signer: release-manager@bank.com
# Last pull: k8s-prod-node-1, 2, 3 (14:35:00)
```

### E2E Example 2: Vulnerability Response Pipeline

**Context:** Critical CVE discovered in OpenSSL. Must rebuild and redeploy 20 services within 4 hours.

```bash
# Hour 1: Discovery & Assessment
$ trivy image --severity CRITICAL registry.bank.com/prod/payment:v2.4.0
# CRITICAL: CVE-2026-9999 in openssl:1.1.1w

# Scan all production images
$ for image in $(harbor-cli list-images prod); do
    trivy image --severity CRITICAL $image 2>/dev/null | grep -q "CRITICAL" && echo "VULNERABLE: $image"
done
# VULNERABLE: registry.bank.com/prod/payment:v2.4.0
# VULNERABLE: registry.bank.com/prod/account:v1.8.2
# VULNERABLE: registry.bank.com/prod/gateway:v3.1.0
# ... (18 services total)

# Hour 2: Base image update
$ cat Dockerfile.updated
FROM openjdk:17-jre-slim
RUN apt-get update && apt-get install -y openssl=1.1.1x-0+deb11u1

$ docker build -t registry.bank.com/base/openjdk:17-jre-slim-patched .
$ docker push registry.bank.com/base/openjdk:17-jre-slim-patched
$ trivy image registry.bank.com/base/openjdk:17-jre-slim-patched
# Result: 0 CRITICAL ✅

# Hour 3: Rebuild all vulnerable services
$ for service in payment account gateway loan notification; do
    echo "Rebuilding $service..."
    sed -i "s|FROM openjdk:17-jre-slim|FROM registry.bank.com/base/openjdk:17-jre-slim-patched|" services/$service/Dockerfile
    docker build -t registry.bank.com/prod/$service:patched services/$service/
    docker push registry.bank.com/prod/$service:patched
    # Trivy scan: 0 CRITICAL ✅
done

# Hour 4: Deploy patched images
$ for service in payment account gateway loan notification; do
    kubectl set image deployment/$service $service=registry.bank.com/prod/$service:patched -n production
    kubectl rollout status deployment/$service -n production --timeout=300s
done

# Verification:
$ for image in $(harbor-cli list-images prod); do
    trivy image --severity CRITICAL $image 2>/dev/null | grep -q "CRITICAL" && echo "STILL VULNERABLE: $image" || echo "CLEAN: $image"
done
# CLEAN: registry.bank.com/prod/payment:v2.4.0
# CLEAN: registry.bank.com/prod/account:v1.8.2
# CLEAN: registry.bank.com/prod/gateway:v3.1.0
# ... (all 20 services clean) ✅

# Total response time: 4 hours (vs 2-3 weeks manually)
# Zero customer impact
# Complete audit trail for regulators
```

### E2E Example 3: Harbor Image Retention & Compliance

**Context:** Implement image retention policy for PCI-DSS compliance (90-day retention).

```yaml
# Harbor retention policy
# File: harbor-retention-policy.yaml

retention_policy:
  rules:
    # Production images: keep all tagged versions
    - repo_selectors:
        - "prod/**"
      tag_selectors:
        - "v*"
      most_recent: 0  # Keep all
      
    # Staging images: keep last 30 versions
    - repo_selectors:
        - "staging/**"
      tag_selectors:
        - "*"
      most_recent: 30
      
    # Dev images: keep last 10 versions
    - repo_selectors:
        - "dev/**"
      tag_selectors:
        - "*"
      most_recent: 10
      
    # Untagged images: delete after 7 days
    - repo_selectors:
        - "**"
      tag_selectors:
        - ""
      untagged: true
      expire_days: 7

# Compliance settings
compliance:
  vulnerability_scanning:
    enabled: true
    scan_on_push: true
    auto_block: true  # Block deployment if CRITICAL found
  
  image_signing:
    enabled: true
    algorithm: "RSA-2048"
  
  audit_logging:
    enabled: true
    retention_days: 365  # Keep logs for 1 year (PCI-DSS)
```

```bash
# Weekly compliance report
$ harbor-cli generate-report --format pci-dss

# PCI-DSS Compliance Report
# Generated: 2026-09-04
# Period: 2026-08-28 to 2026-09-04
#
# ✅ 6.5.1: All images scanned for vulnerabilities
# ✅ 6.5.2: No CRITICAL vulnerabilities in production
# ✅ 6.5.3: Image signing enabled for all production images
# ✅ 6.5.4: Audit logs retained for 365 days
# ✅ 6.5.5: Access control: RBAC enabled
# ✅ 6.5.6: Image immutability: enabled
#
# Total images scanned: 1,247
# Vulnerabilities found: 0 CRITICAL, 3 HIGH (remediated)
# Images signed: 100%
# Audit log entries: 45,892
# Compliance score: 100% ✅
```

---

## 📋 Interview Questions

### Q1: What is the difference between Docker Hub and a private registry like Harbor?
**Answer:** Docker Hub is a public registry with limited access control and no audit logging. Harbor is a self-hosted private registry designed for enterprise use. 

Key differences: 

(1) **Access control** — Harbor supports RBAC with LDAP/AD integration. 

(2) **Security scanning** — Harbor has built-in vulnerability scanning. 

(3) **Image signing** — Harbor supports Notary for content trust. 

(4) **Audit logging** — Harbor records every push/pull. 

(5) **Retention policies** — Harbor can auto-delete old images. 

Banks use Harbor or cloud registries (ECR, ACR) for compliance.

### Q2: Why is image tagging strategy important in banking?
**Answer:** Proper tagging provides: 

(1) **Traceability** — tag includes Git SHA or build number linking to exact code version. 

(2) **Rollback** — specific tags can be redeployed instantly. 

(3) **Compliance** — auditors can trace which code produced which image. 

(4) **Environment separation** — dev, staging, and prod images are clearly distinguished. 

Never use `latest` tag in production — it's ambiguous and breaks rollback.

### Q3: How do you handle image vulnerability scanning in CI/CD?
**Answer:** Integrate scanning into the pipeline: 

(1) **Scan during build** — run Trivy/Snyk after building the image. 

(2) **Set quality gates** — fail the pipeline if CRITICAL or HIGH vulnerabilities are found. 

(3) **Continuous scanning** — re-scan all images in registry periodically (new CVEs are discovered daily). 

(4) **Auto-rebuild** — when base image is patched, automatically rebuild all dependent images. 

(5) **Exception process** — for accepted risks, document the exception with business justification.

### Q4: What is image immutability and why does it matter?
**Answer:** Once a Docker image is pushed to the registry with a specific tag, it should **never be overwritten**. This is image immutability. 

Benefits: 

(1) **Deterministic deployments** — the same image runs everywhere. 

(2) **Rollback** — previous versions are always available. 

(3) **Audit** — you know exactly what's running in production. 

(4) **Security** — no one can tamper with a deployed image. 

Use unique tags (Git SHA, build number) instead of overwriting `latest`.

### Q5: How do you manage secrets in Docker images for banking?
**Answer:** Never bake secrets into images. 

Use: 

(1) **Runtime injection** — pass secrets as environment variables at container start. 

(2) **Kubernetes Secrets** — mount secrets as volumes. 

(3) **External secret management** — HashiCorp Vault, AWS Secrets Manager. 

(4) **Build-time secrets** — Docker BuildKit supports `--secret` mount that doesn't persist in layers. 

(5) **Scan for secrets** — use TruffleHog or GitLeaks to detect accidentally committed secrets.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Registry | Central storage for Docker images |
| Private Registry | Essential for banking security & compliance |
| Image Tagging | Use semantic versions + Git SHA for traceability |
| Vulnerability Scanning | Automated, integrated into CI pipeline |
| Image Signing | Verify image integrity and authenticity |
| Banking Relevance | Audit trails, compliance, secure distribution |

**Next:** [08-Kubernetes-Basics.md](./08-Kubernetes-Basics.md) — Learn Kubernetes, the container orchestration platform.
