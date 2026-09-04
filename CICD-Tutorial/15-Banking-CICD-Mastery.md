# 15 — Banking CI/CD Mastery: End-to-End Examples

> **Goal:** Bring everything together — complete banking CI/CD implementations from code to production.

---

## 🎯 Complete Banking CI/CD Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BANKING CI/CD ARCHITECTURE                            │
│                                                                             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐     │
│  │  Git    │──▶│ Jenkins │──▶│ Harbor  │──▶│ ArgoCD  │──▶│ K8s     │     │
│  │  Repo   │   │ (CI)    │   │ Registry│   │ (CD)    │   │ Cluster │     │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘     │
│       │             │             │             │             │             │
│       ▼             ▼             ▼             ▼             ▼             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐     │
│  │ GitLab  │   │ Sonar   │   │ Trivy   │   │ Helm    │   │Prometheus│     │
│  │ (SCM)   │   │ (SAST)  │   │ (Scan)  │   │ Charts  │   │Grafana  │     │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘     │
│                                                                             │
│  Security Layer: HashiCorp Vault + Falco + Network Policies                │
│  Compliance Layer: Automated PCI-DSS + GDPR + RBI Checks                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏦 End-to-End Example 1: UPI Payment Service

### Business Context
Bank needs to build and deploy a UPI payment service that handles 10,000+ transactions per minute.

### Complete CI/CD Pipeline

```yaml
# Jenkinsfile - Complete UPI Payment Service Pipeline
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.bank.com'
        APP_NAME = 'upi-payment-service'
        SONAR_TOKEN = credentials('sonar-token')
        VAULT_ADDR = 'https://vault.bank.com'
    }
    
    stages {
        // ═══════════════════════════════════════════
        // STAGE 1: SOURCE
        // ═══════════════════════════════════════════
        stage('Source') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 2: BUILD
        // ═══════════════════════════════════════════
        stage('Build') {
            steps {
                sh '''
                    mvn clean compile -DskipTests
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 3: UNIT TESTS
        // ═══════════════════════════════════════════
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 4: SECURITY SCANS
        // ═══════════════════════════════════════════
        stage('Security Scan') {
            parallel {
                stage('SAST') {
                    steps {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=upi-payment \
                                -Dsonar.sources=./src \
                                -Dsonar.qualitygate.wait=true
                        '''
                    }
                }
                stage('Dependency Check') {
                    steps {
                        sh '''
                            dependency-check \
                                --project "UPI Payment" \
                                --scan ./target/*.jar \
                                --out ./report \
                                --format HTML
                        '''
                    }
                }
                stage('Secret Scan') {
                    steps {
                        sh '''
                            trufflehog filesystem --directory ./src \
                                --fail --json > secrets-report.json
                        '''
                    }
                }
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 5: BUILD DOCKER IMAGE
        // ═══════════════════════════════════════════
        stage('Build Image') {
            steps {
                sh '''
                    docker build \
                        --build-arg JAR_FILE=target/*.jar \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT} \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:latest \
                        .
                    docker push ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT}
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 6: IMAGE SECURITY SCAN
        // ═══════════════════════════════════════════
        stage('Image Scan') {
            steps {
                sh '''
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        ${DOCKER_REGISTRY}/${APP_NAME}:${GIT_COMMIT_SHORT}
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 7: INTEGRATION TESTS
        // ═══════════════════════════════════════════
        stage('Integration Tests') {
            steps {
                sh '''
                    docker-compose -f docker-compose.test.yml up -d
                    mvn verify -P integration-tests
                    docker-compose -f docker-compose.test.yml down
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 8: COMPLIANCE CHECKS
        // ═══════════════════════════════════════════
        stage('Compliance') {
            steps {
                sh '''
                    python compliance/pci_dss_check.py
                    python compliance/upi_limits_check.py
                    python compliance/data_localization_check.py
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 9: DEPLOY TO STAGING
        // ═══════════════════════════════════════════
        stage('Deploy Staging') {
            steps {
                sh '''
                    helm upgrade --install payment-staging \
                        ./helm/payment-chart \
                        -f ./helm/values-staging.yaml \
                        --set image.tag=${GIT_COMMIT_SHORT} \
                        -n staging
                    kubectl rollout status deployment/payment-service -n staging --timeout=300s
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 10: E2E TESTS ON STAGING
        // ═══════════════════════════════════════════
        stage('E2E Tests') {
            steps {
                sh '''
                    mvn verify -P e2e-tests -Dbase.url=https://staging.bank.com
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 11: APPROVAL GATE
        // ═══════════════════════════════════════════
        stage('Approval') {
            steps {
                input message: 'Deploy UPI Payment Service to Production?',
                       ok: 'Deploy to Production',
                       submitter: 'release-manager,security-team'
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 12: DEPLOY TO PRODUCTION
        // ═══════════════════════════════════════════
        stage('Deploy Production') {
            steps {
                sh '''
                    helm upgrade --install payment-prod \
                        ./helm/payment-chart \
                        -f ./helm/values-prod.yaml \
                        --set image.tag=${GIT_COMMIT_SHORT} \
                        -n production
                    kubectl rollout status deployment/payment-service -n production --timeout=600s
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 13: POST-DEPLOY VALIDATION
        // ═══════════════════════════════════════════
        stage('Post-Deploy Validation') {
            steps {
                sh '''
                    python scripts/validate-production.py \
                        --service payment-service \
                        --duration 5m \
                        --error-threshold 0.001
                '''
            }
        }
        
        // ═══════════════════════════════════════════
        // STAGE 14: MONITORING & NOTIFICATION
        // ═══════════════════════════════════════════
        stage('Notify') {
            steps {
                slackSend(
                    channel: '#deployments',
                    message: """
                        ✅ *UPI Payment Service Deployed*
                        Version: ${GIT_COMMIT_SHORT}
                        Environment: Production
                        Deployer: ${currentBuild.currentResult}
                        Pipeline: ${env.BUILD_URL}
                    """
                )
            }
        }
    }
    
    post {
        failure {
            slackSend(
                channel: '#alerts',
                message: "❌ Pipeline failed for ${APP_NAME}"
            )
        }
    }
}
```

---

## 🏦 End-to-End Example 2: Core Banking System Migration

### Business Context
Migrate a 20-year-old core banking system from physical servers to Kubernetes.

### Migration CI/CD Strategy

```
Phase 1: Strangler Fig Pattern (Month 1-3)
┌─────────────────────────────────────────┐
│           API Gateway                    │
│    ┌──────────────┬──────────────┐      │
│    │   New (K8s)  │   Old (VM)   │      │
│    │              │              │      │
│    │ Payment Svc  │ Account Svc  │      │
│    │ (Container)  │ (Physical)   │      │
│    └──────────────┴──────────────┘      │
└─────────────────────────────────────────┘

Phase 2: Incremental Migration (Month 4-9)
┌─────────────────────────────────────────┐
│           API Gateway                    │
│    ┌──────────────┬──────────────┐      │
│    │   New (K8s)  │   Old (VM)   │      │
│    │              │              │      │
│    │ Payment Svc  │ Account Svc  │      │
│    │ Notify Svc   │ Report Svc   │      │
│    │ (Container)  │ (Physical)   │      │
│    └──────────────┴──────────────┘      │
└─────────────────────────────────────────┘

Phase 3: Complete Migration (Month 10-12)
┌─────────────────────────────────────────┐
│           API Gateway                    │
│    ┌──────────────────────────────┐     │
│    │        All New (K8s)          │     │
│    │                              │     │
│    │ Payment │ Account │ Notify   │     │
│    │ Report  │ Auth    │ Gateway  │     │
│    │ (All Containerized)          │     │
│    └──────────────────────────────┘     │
└─────────────────────────────────────────┘
```

### Migration Pipeline
```yaml
# migration-pipeline.yml
stages:
  - analyze
  - containerize
  - test
  - validate
  - deploy

analyze:
  stage: analyze
  script:
    - python migration/analyze_app.py --app account-service
    # Output: Dependencies, ports, env vars, health checks

containerize:
  stage: containerize
  script:
    - python migration/generate_dockerfile.py --app account-service
    - docker build -t registry.bank.com/core/account-service:${VERSION} .
    - docker push registry.bank.com/core/account-service:${VERSION}

validate-migration:
  stage: validate
  script:
    # Shadow traffic: send same requests to old and new
    - python migration/shadow_traffic.py \
        --old-url https://old.bank.com/api/accounts \
        --new-url https://new.bank.com/api/accounts \
        --duration 1h \
        --compare-responses
    # Verify 100% response match
```

---

## 🏦 End-to-End Example 3: Multi-Region ATM Software

### Business Context
Deploy ATM software to 10,000 ATMs across 50 countries with regional compliance.

### Regional Deployment Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    GLOBAL ARGOCD                             │
│                    (GitOps Controller)                       │
└──────────┬──────────────────┬──────────────────┬───────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  India Cluster   │ │  UAE Cluster     │ │  Singapore       │
│  (Mumbai)        │ │  (Dubai)         │ │  Cluster         │
│                  │ │                  │ │                  │
│  ┌─────────────┐│ │  ┌─────────────┐│ │  ┌─────────────┐│
│  │atm-software ││ │  │atm-software ││ │  │atm-software ││
│  │  5,000 ATMs ││ │  │  2,000 ATMs ││ │  │  3,000 ATMs ││
│  │  RBI rules  ││ │  │  CBUA rules ││ │  │  MAS rules  ││
│  └─────────────┘│ │  └─────────────┘│ │  └─────────────┘│
│                  │ │                  │ │                  │
│  Region-specific │ │  Region-specific │ │  Region-specific │
│  compliance      │ │  compliance      │ │  compliance      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### Regional Helm Values
```yaml
# values-india.yaml
region: india
compliance: rbi
atms:
  count: 5000
  connection_type: satellite
features:
  upi: true
  neft: true
  rtgs: true
  bhim: true
transaction_limits:
  upi_daily: 100000
  neft_per_txn: 500000
  rtgs_min: 200000

# values-uae.yaml
region: uae
compliance: cbuae
atms:
  count: 2000
  connection_type: fiber
features:
  uae_fts: true
  wps: true
  local_transfers: true
transaction_limits:
  uae_fts_daily: 500000
  wps_per_txn: 2000000
```

---

## 📋 Interview Questions

### Q1: Walk me through a complete CI/CD pipeline for a banking application.
**Answer:** 

(1) **Source** — Developer pushes code to Git. 

(2) **Build** — Maven/Gradle compiles the application. 

(3) **Unit Tests** — Automated tests verify business logic. 

(4) **SAST** — SonarQube scans for code vulnerabilities. 

(5) **Dependency Check** — OWASP scans libraries for CVEs. 

(6) **Docker Build** — Application packaged as Docker image. 

(7) **Image Scan** — Trivy scans image for vulnerabilities. 

(8) **Integration Tests** — Tests verify component interactions. 

(9) **Compliance** — Automated PCI-DSS/GDPR checks. 

(10) **Staging Deploy** — Helm deploys to staging. 

(11) **E2E Tests** — Full user flow testing. 

(12) **Approval** — Release manager approves. 

(13) **Production Deploy** — Helm deploys to production. 

(14) **Monitoring** — Prometheus/Grafana validates deployment.

### Q2: How do you handle a production incident using CI/CD?
**Answer:** 

(1) **Detect** — Monitoring alerts on error rate spike. 

(2) **Diagnose** — Check logs (ELK), traces (Jaeger), metrics (Prometheus). 

(3) **Fix** — Developer creates hotfix branch, fixes the bug. 

(4) **CI** — Pipeline runs tests and security scans. 

(5) **CD** — Hotfix deployed to production (expedited pipeline). 

(6) **Validate** — Monitoring confirms error rate normalized. 

(7) **Post-mortem** — Document root cause, add tests to prevent recurrence. 

Total time: 30-60 minutes (vs hours/days manually).

### Q3: How do you implement feature flags in a banking application?
**Answer:** Feature flags decouple deployment from release. 

(1) Deploy code with flag `false` (feature hidden). 

(2) Enable for internal users first. 

(3) Enable for 5% of customers. 

(4) Monitor for issues. 

(5) Enable for 100%. If issues found, disable flag instantly (no rollback needed). 

Use LaunchDarkly, Unleash, or custom implementation. 

In banking, feature flags are critical for regulatory compliance (gradual rollout required).

### Q4: How do you ensure database schema changes don't cause downtime?
**Answer:** Use the **expand and contract** 

pattern: 

(1) **Expand** — add new columns/tables (backward compatible). 

(2) **Deploy code** that works with both old and new schema. 

(3) **Migrate data** if needed. 

(4) **Contract** — remove old columns in next release. 

Tools: Flyway or Liquibase for version-controlled migrations. CI runs migration on a copy of production schema to verify before deployment.

### Q5: What metrics would you track to measure CI/CD effectiveness?
**Answer:** DORA metrics (DevOps Research and Assessment): 

(1) **Deployment Frequency** — how often you deploy (target: daily for microservices). 

(2) **Lead Time for Changes** — time from commit to production (target: < 1 day). 

(3) **Change Failure Rate** — % of deployments causing failures (target: < 5%). 

(4) **Mean Time to Recovery (MTTR)** — time to recover from failure (target: < 1 hour). 

Additional: pipeline duration, test coverage, security scan pass rate.

---

## 🎓 Your CI/CD Learning Path

```
You started here:
"I know nothing about CI/CD"
          │
          ▼
┌─────────────────────────────────────────────┐
│  Foundation (Topics 1-3)                     │
│  ✅ CI/CD Basics                            │
│  ✅ Git Version Control                      │
│  ✅ Continuous Integration                   │
│  Status: You understand the concepts         │
└─────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│  Intermediate (Topics 4-7)                   │
│  ✅ Continuous Delivery                      │
│  ✅ Continuous Deployment                    │
│  ✅ Docker Fundamentals                     │
│  ✅ Docker Registry                         │
│  Status: You can build and containerize apps │
└─────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│  Advanced (Topics 8-12)                      │
│  ✅ Kubernetes Basics                        │
│  ✅ Kubernetes Advanced                     │
│  ✅ Pipeline Tools                          │
│  ✅ Helm Charts                             │
│  ✅ Infrastructure as Code                   │
│  Status: You can deploy at scale             │
└─────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────┐
│  Expert (Topics 13-15)                       │
│  ✅ Monitoring & Observability               │
│  ✅ DevSecOps                                │
│  ✅ Banking CI/CD Mastery                   │
│  Status: You are a CI/CD master! 🎉          │
└─────────────────────────────────────────────┘
```

---

## 📚 Final Summary

| Topic | Key Takeaway |
|-------|-------------|
| CI/CD Basics | Automate builds and deployments |
| Git | Version control is the foundation |
| CI | Catch bugs early with automated testing |
| CD (Delivery) | Always be deployable |
| CD (Deployment) | Fully automated releases |
| Docker | Consistent, portable containers |
| Docker Registry | Secure image storage |
| Kubernetes | Container orchestration at scale |
| K8s Advanced | Namespaces, secrets, resource management |
| Pipeline Tools | Jenkins, GitLab CI, ArgoCD |
| Helm | K8s package management |
| IaC | Terraform + Ansible for infrastructure |
| Monitoring | Prometheus + Grafana + ELK |
| Security | DevSecOps, shift-left security |
| Banking | End-to-end compliance-driven CI/CD |

**You are now a CI/CD Master! 🎉**

**Next steps:**
1. Practice by setting up a personal CI/CD pipeline
2. Contribute to open-source CI/CD projects
3. Get certified: AWS DevOps, Kubernetes CKA/CKAD
4. Build a portfolio of CI/CD implementations
