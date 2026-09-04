# 10 — CI/CD Pipeline Tools: Jenkins, GitLab CI, ArgoCD

> **Goal:** Understand the major CI/CD tools and how they power banking pipelines.

---

## 🛠️ The CI/CD Tool Landscape

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD TOOL ECOSYSTEM                       │
│                                                             │
│  CI Tools (Build & Test):                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Jenkins  │  │ GitLab CI│  │ GitHub   │  │ CircleCI │  │
│  │          │  │          │  │ Actions  │  │          │  │
│  │ (Most    │  │ (Built-in│  │ (Cloud-  │  │ (SaaS    │  │
│  │ popular) │  │ with Git)│  │ native)  │  │  CI)     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
│                                                             │
│  CD Tools (Deploy):                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ ArgoCD   │  │ FluxCD   │  │ Spinnaker│  │ Octopus  │  │
│  │          │  │          │  │          │  │ Deploy   │  │
│  │ (GitOps  │  │ (GitOps  │  │ (Multi-  │  │ (Enterprise│ │
│  │  for K8s)│  │  native) │  │  cloud)  │  │  CD)     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Jenkins: The Industry Standard

### What is Jenkins?
Jenkins is the most widely used open-source CI/CD server. It automates building, testing, and deploying software.

### Jenkins Pipeline (Jenkinsfile)

```groovy
// Jenkinsfile - Declarative Pipeline
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.bank.com'
        APP_NAME = 'payment-service'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .
                    docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                '''
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                        -n staging
                    kubectl rollout status deployment/${APP_NAME} -n staging
                '''
            }
        }
        
        stage('Approve for Production') {
            steps {
                input message: 'Deploy to production?', ok: 'Yes, deploy!'
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh '''
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                        -n production
                    kubectl rollout status deployment/${APP_NAME} -n production
                '''
            }
        }
    }
    
    post {
        success {
            slackSend channel: '#deployments', 
                      message: "✅ ${APP_NAME} v${BUILD_NUMBER} deployed successfully"
        }
        failure {
            slackSend channel: '#deployments', 
                      message: "❌ ${APP_NAME} pipeline failed"
        }
    }
}
```

---

## 🔧 GitLab CI: Built-In with Git

### What is GitLab CI?
GitLab CI is integrated directly into GitLab — no separate server needed. Pipelines are defined in `.gitlab-ci.yml`.

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - staging
  - production

variables:
  DOCKER_REGISTRY: registry.bank.com
  APP_NAME: payment-service

# Stage 1: Build
build:
  stage: build
  image: maven:3.8-openjdk-17
  script:
    - mvn clean package -DskipTests
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
  artifacts:
    paths:
      - target/*.jar

# Stage 2: Unit Tests
unit-tests:
  stage: test
  image: maven:3.8-openjdk-17
  script:
    - mvn test
  coverage: '/Total.*?(\d+%)/'
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

# Stage 3: Integration Tests
integration-tests:
  stage: test
  image: maven:3.8-openjdk-17
  services:
    - name: postgres:15
      alias: db
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: test_user
    POSTGRES_PASSWORD: test_pass
  script:
    - mvn verify -P integration-tests
  allow_failure: false

# Stage 4: Security Scan
security-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA

# Stage 5: Deploy to Staging
deploy-staging:
  stage: staging
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/$APP_NAME -n staging --timeout=300s
  environment:
    name: staging
    url: https://staging.bank.com
  only:
    - develop

# Stage 6: Deploy to Production
deploy-production:
  stage: production
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA -n production
    - kubectl rollout status deployment/$APP_NAME -n production --timeout=300s
  environment:
    name: production
    url: https://api.bank.com
  when: manual
  only:
    - main
```

---

## 🔧 ArgoCD: GitOps for Kubernetes

### What is ArgoCD?
ArgoCD is a **GitOps** tool that keeps Kubernetes in sync with Git. Instead of `kubectl apply` in pipelines, ArgoCD watches Git and applies changes automatically.

### GitOps Flow
```
Traditional CD:                    GitOps (ArgoCD):
Developer → CI Pipeline → kubectl   Developer → CI Pipeline → Push to Git
                                                         │
                                                         ▼
                                                   ArgoCD watches Git
                                                         │
                                                         ▼
                                                   Auto-sync to K8s
```

### ArgoCD Application
```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: banking
  source:
    repoURL: https://git.bank.com/infra/k8s-manifests.git
    targetRevision: main
    path: payment-service/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Jenkins for Legacy Banking Systems
**Context:** A bank has a 15-year-old Java application running on physical servers. They need to modernize the CI/CD pipeline.

**Jenkins Migration:**
```
BEFORE:
  Developer → FTP to server → Manual deploy → Hope it works
  
AFTER:
  Developer → Git commit → Jenkins pipeline → Docker build → Deploy to K8s
  
Timeline:
  Month 1: Set up Jenkins, create pipeline
  Month 2: Containerize application
  Month 3: Migrate to Kubernetes
  Month 4: Decommission old servers
```

### Scenario 2: GitLab CI for Microservices
**Context:** Bank has 30 microservices, each with its own GitLab repository.

**GitLab CI with Shared Templates:**
```yaml
# .gitlab-ci-templates/banking-standard.yml
# Shared pipeline template for all microservices

.build-template:
  image: maven:3.8-openjdk-17
  before_script:
    - mvn dependency:resolve
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .m2/repository

.test-template:
  extends: .build-template
  script:
    - mvn test
  coverage: '/Total.*?(\d+%)/'

.security-template:
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE

# Each microservice inherits the template:
# payment-service/.gitlab-ci.yml
include:
  - project: 'devops/pipeline-templates'
    file: '/templates/banking-standard.yml'

stages:
  - build
  - test
  - security
  - deploy

build:
  extends: .build-template
  stage: build

test:
  extends: .test-template
  stage: test

security:
  extends: .security-template
  stage: security
```

### Scenario 3: ArgoCD for Multi-Cluster Banking
**Context:** Bank operates 3 Kubernetes clusters: Mumbai (primary), Delhi (DR), Singapore (international).

**ArgoCD Multi-Cluster Sync:**
```yaml
# ArgoCD Application for multi-cluster deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-global
  namespace: argocd
spec:
  project: banking
  source:
    repoURL: https://git.bank.com/infra/k8s-manifests.git
    targetRevision: main
    path: payment-service/overlays/production
  destinations:
    # Mumbai - Primary
    - server: https://mumbai-k8s.bank.com
      namespace: production
    # Delhi - Disaster Recovery
    - server: https://delhi-k8s.bank.com
      namespace: production
    # Singapore - International
    - server: https://sg-k8s.bank.com
      namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  # ArgoCD syncs all 3 clusters simultaneously
  # Each cluster gets the exact same version
  # Rollback: revert Git commit, ArgoCD auto-syncs
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Jenkins Pipeline for Core Banking Migration

**Context:** Migrate 50 microservices from physical servers to Kubernetes using Jenkins.

```groovy
// Jenkinsfile - Core Banking Migration Pipeline
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.bank.com'
        APP_NAME = "${env.JOB_NAME.split('/')[0]}"
        K8S_NAMESPACE = 'production'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }
        
        stage('Build & Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                        junit 'target/surefire-reports/TEST-*.xml'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'docker-compose -f docker-compose.test.yml up -d'
                        sh 'mvn verify -P integration-tests'
                        sh 'docker-compose -f docker-compose.test.yml down'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL $REGISTRY/$APP_NAME:$GIT_SHA'
                    }
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                sh "docker build -t $REGISTRY/$APP_NAME:$GIT_SHA ."
                sh "docker push $REGISTRY/$APP_NAME:$GIT_SHA"
            }
        }
        
        stage('Deploy Staging') {
            steps {
                sh "helm upgrade --install $APP_NAME ./helm/$APP_NAME -f ./helm/values-staging.yaml --set image.tag=$GIT_SHA -n staging"
                sh "kubectl rollout status deployment/$APP_NAME -n staging --timeout=300s"
            }
        }
        
        stage('E2E Tests') {
            steps {
                sh 'mvn verify -P e2e-tests -Dbase.url=https://staging.bank.com'
            }
        }
        
        stage('Approval') {
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
            }
        }
        
        stage('Deploy Production') {
            steps {
                sh "helm upgrade --install $APP_NAME ./helm/$APP_NAME -f ./helm/values-prod.yaml --set image.tag=$GIT_SHA -n $K8S_NAMESPACE"
                sh "kubectl rollout status deployment/$APP_NAME -n $K8S_NAMESPACE --timeout=600s"
            }
        }
    }
    
    post {
        success {
            slackSend channel: '#deployments', message: "✅ $APP_NAME deployed successfully"
        }
        failure {
            slackSend channel: '#alerts', message: "❌ $APP_NAME deployment failed"
        }
    }
}
```

**Pipeline Execution:**
```
Job: core-banking/payment-service
Build: #1847
Duration: 22 minutes

Stage: Checkout          ✅ 15s
Stage: Unit Tests        ✅ 3m 45s (847/847 tests passed)
Stage: Integration Tests ✅ 8m 20s (156/156 tests passed)
Stage: Security Scan     ✅ 1m 30s (0 CVEs)
Stage: Docker Build      ✅ 2m 15s (image: 215MB)
Stage: Deploy Staging    ✅ 1m 30s
Stage: E2E Tests         ✅ 4m 00s (45/45 tests passed)
Stage: Approval          ✅ 0s (approved by: release-manager)
Stage: Deploy Production ✅ 2m 15s

Result: SUCCESS ✅
Commits: 3 (abc123, def456, ghi789)
Artifacts: registry.bank.com/prod/payment:v2.4.0
Environment: production (cluster-prod-01)
```

### E2E Example 2: GitLab CI for 30 Microservices

**Context:** Standardized pipeline template for all banking microservices.

```yaml
# Shared template (devops/pipeline-templates/banking-standard.yml)
.banking-template:
  image: maven:3.8-openjdk-17
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .m2/repository
  
  before_script:
    - mvn dependency:resolve

.build:
  extends: .banking-template
  stage: build
  script:
    - mvn clean package -DskipTests

.test:
  extends: .banking-template
  stage: test
  script:
    - mvn test
    - mvn jacoco:report
  coverage: '/Total.*?(\d+%)/'
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

.security:
  stage: security
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
    - sonar-scanner -Dsonar.qualitygate.wait=true

.deploy:
  stage: deploy
  script:
    - helm upgrade --install $CI_PROJECT_NAME ./helm -f ./helm/values-${CI_ENVIRONMENT_NAME}.yaml --set image.tag=$CI_COMMIT_SHA -n $CI_ENVIRONMENT_NAME
    - kubectl rollout status deployment/$CI_PROJECT_NAME -n $CI_ENVIRONMENT_NAME --timeout=300s

# Each microservice inherits the template:
# File: payment-service/.gitlab-ci.yml
include:
  - project: 'devops/pipeline-templates'
    file: '/templates/banking-standard.yml'

stages:
  - build
  - test
  - security
  - staging
  - production

build:
  extends: .build

test:
  extends: .test

security:
  extends: .security

deploy-staging:
  extends: .deploy
  environment:
    name: staging
  only:
    - develop

deploy-production:
  extends: .deploy
  environment:
    name: production
  when: manual
  only:
    - main
```

**Results across 30 microservices:**
```
Service                    Pipeline Time   Success Rate
─────────────────────────────────────────────────────
payment-service            22 min          98.5%
account-service            18 min          99.2%
loan-service               25 min          97.8%
fraud-detection            35 min          96.5%
notification-service       12 min          99.8%
gateway-service            15 min          99.0%
reconciliation-service     28 min          98.0%
...

Total: 30 services, 1,247 pipeline runs this month
Average success rate: 98.2%
Average pipeline time: 21 minutes
```

### E2E Example 3: ArgoCD GitOps for Multi-Cluster Banking

**Context:** Deploy to 3 Kubernetes clusters (Mumbai, Delhi, Singapore) using ArgoCD.

```yaml
# ArgoCD Application for multi-cluster deployment
# File: argocd/payment-application.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: banking
  source:
    repoURL: https://git.bank.com/infra/k8s-manifests.git
    targetRevision: main
    path: payment-service/overlays/production
  destination:
    server: https://mumbai-k8s.bank.com
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-dr
  namespace: argocd
defaults:
  spec:
    project: banking
    source:
      repoURL: https://git.bank.com/infra/k8s-manifests.git
      targetRevision: main
      path: payment-service/overlays/production
    destination:
      server: https://delhi-k8s.bank.com
      namespace: production
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
```

```bash
# ArgoCD syncs all clusters automatically
$ argocd app list
# NAME              CLUSTER                        STATUS    HEALTH
# payment-service   https://mumbai-k8s.bank.com    Synced    Healthy
# payment-service-dr https://delhi-k8s.bank.com     Synced    Healthy
# payment-service-sg https://sg-k8s.bank.com        Synced    Healthy

# Developer pushes change to Git
$ git push origin main
# ArgoCD detects change within 3 minutes
# Auto-syncs to all 3 clusters
# Zero manual intervention

# Rollback: revert Git commit
$ git revert HEAD
$ git push origin main
# ArgoCD auto-syncs rollback to all 3 clusters
# Rollback time: 3 minutes
```

---

## 📋 Interview Questions

### Q1: When would you choose Jenkins over GitLab CI?
**Answer:** 

Jenkins is preferred when: 

(1) You need extensive plugin ecosystem (1800+ plugins). 

(2) You have complex, custom pipeline logic. 

(3) You need to integrate with multiple SCM systems (not just Git). 

(4) You want full control over the infrastructure. 

GitLab CI is preferred when: 

(1) You already use GitLab for source control. 

(2) You want a simpler, integrated experience. 

(3) You prefer YAML over Groovy. 

Many banks use Jenkins for its maturity and flexibility.

### Q2: What is GitOps and why is it gaining popularity in banking?
**Answer:** GitOps is a deployment methodology where Git is the single source of truth for infrastructure and application configuration. Every change goes through Git (code review, audit trail). ArgoCD/FluxCD watches Git and automatically syncs to Kubernetes. 

Benefits for banking: 

(1) Complete audit trail in Git. 

(2) Easy rollback (revert Git commit). 

(3) Declarative configuration. 

(4) Separation of concerns (developers own code, ops own manifests).

### Q3: How do you handle secrets in CI/CD pipelines?
**Answer:** Never hardcode secrets. Use: (1) **Pipeline variables** — store in CI tool's encrypted variable store (GitLab CI Variables, Jenkins Credentials). (2) **Vault integration** — HashiCorp Vault for dynamic secrets. (3) **Cloud KMS** — AWS KMS, Azure Key Vault. (4) **Kubernetes Secrets** — for runtime secrets in pods. (5) **OIDC** — short-lived tokens instead of long-lived credentials. Example: Jenkins uses "Credentials" plugin; GitLab uses protected variables.

### Q4: What is the difference between push-based and pull-based CD?
**Answer:** 

**Push-based** (Jenkins, GitLab CI): The CI server pushes changes to the cluster via `kubectl apply`. The CI server needs cluster credentials. 

**Pull-based** (ArgoCD, Flux): The CD tool runs inside the cluster and pulls changes from Git. The cluster doesn't expose credentials. Pull-based is more secure (no inbound access to cluster) and is the GitOps standard. Banks increasingly prefer pull-based for security.

### Q5: How do you implement pipeline-as-code and why does it matter?
**Answer:** Pipeline-as-code means defining CI/CD pipelines in version-controlled files (Jenkinsfile, .gitlab-ci.yml, GitHub Actions). 

Benefits: 

(1) **Auditability** — every pipeline change is tracked in Git. 

(2) **Reviewability** — pipeline changes go through code review. 

(3) **Reproducibility** — same pipeline runs the same way every time. 

(4) **Rollback** — revert pipeline changes if they break. 

(5) **Documentation** — the pipeline IS the documentation. Banks require this for compliance.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Jenkins | Most popular, flexible, plugin-rich CI server |
| GitLab CI | Integrated with Git, YAML-based, simpler |
| ArgoCD | GitOps for K8s, pull-based, auto-sync |
| Pipeline-as-Code | Version-controlled, auditable pipelines |
| Banking Relevance | Audit trails, compliance, multi-cluster |

**Next:** [11-Helm-Charts.md](./11-Helm-Charts.md) — Learn Kubernetes package management.
