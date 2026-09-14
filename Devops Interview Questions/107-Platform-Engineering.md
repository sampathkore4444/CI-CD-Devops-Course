# 107. Designing an Internal Developer Platform

## Scenario

Your organization has 15 development teams deploying 50+ microservices across 3 Kubernetes clusters. Each team manages their own CI/CD pipelines, Kubernetes manifests, monitoring dashboards, and infrastructure provisioning. The results are inconsistent: some teams deploy 10x/day with high confidence, while others deploy once a week with anxiety. Security standards vary wildly — some teams run image scanning, others don't. Onboarding a new developer takes 2 weeks because they need to learn each team's unique tooling. The CTO has tasked you with designing an Internal Developer Platform (IDP) that provides golden paths while giving teams the autonomy they need.

## Interviewer Question

"How would you design an Internal Developer Platform that standardizes developer workflows across 15 teams while maintaining team autonomy? Walk me through the architecture, tooling choices, and how you'd measure success."

## What I Should Think About

- Platform Engineering is the discipline of building internal platforms that improve developer productivity
- The goal: reduce cognitive load on development teams by abstracting infrastructure complexity
- Golden paths: opinionated, supported workflows that make the right thing easy (not mandatory)
- Self-service is key: developers shouldn't need to file tickets for infrastructure
- Backstage (Spotify) as a developer portal for service catalog, templates, and documentation
- Standardized CI/CD: reusable pipeline templates, not dictated implementations
- Observability as a service: pre-configured dashboards and alerting per service
- Security guardrails: OPA/Gatekeeper policies that enforce standards without blocking speed
- Measuring success: DORA metrics (deployment frequency, lead time, MTTR, change failure rate)
- Platform team structure: dedicated platform team, not DevOps as shared service

## Ideal Answer

I'd design the platform around five core pillars:

**1. Developer Portal (Backstage)**
A single pane of glass for all developer needs. Service catalog showing all 50+ microservices with ownership, documentation, API specs, and health status. Software templates for scaffolding new services, creating databases, or provisioning environments. TechDocs for living documentation that stays up-to-date automatically. Plugins for Kubernetes, CI/CD, monitoring, and security scanning.

**2. Golden Path CI/CD Pipelines**
Reusable pipeline templates using Tekton or GitHub Actions. Teams select a template based on their service type (Go, Java, Node.js) and the template handles: linting, unit tests, image building, security scanning, deployment to staging, smoke tests, and production deployment with approval gates. Teams can customize within guardrails but can't bypass security scanning or approval requirements.

**3. Self-Service Infrastructure**
A CLI tool or Backstage plugin that lets developers provision: Kubernetes namespaces with resource quotas, PostgreSQL databases (via Crossplane or CloudSQL), Redis instances, secrets (via Vault), monitoring dashboards (via Grafana provisioning), and alerting rules. No tickets required — everything is API-driven and automated.

**4. Observability Stack**
Pre-configured for every service: Prometheus metrics collection, Grafana dashboards with service-specific panels, alerting rules based on SLOs, distributed tracing via OpenTelemetry, and log aggregation. New services get observability automatically when they deploy using the golden path.

**5. Security & Compliance Guardrails**
OPA/Gatekeeper policies that enforce: only approved base images, image scanning requirements, resource limits, network policies, and RBAC. Developers get clear error messages when they violate policies, with links to documentation on how to fix it.

## Architecture

```
Internal Developer Platform Architecture

┌─────────────────────────────────────────────────────────────┐
│                    Developer Portal (Backstage)             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Service  │  │ Software │  │ TechDocs │  │  Plugin  │   │
│  │ Catalog  │  │Templates │  │          │  │  Store   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  Platform API Layer (CI/CD, Infra,          │
│                     Secrets, Monitoring)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                 Kubernetes Clusters (3)                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              OPA/Gatekeeper Policies                │    │
│  │  - Image scanning  - Resource limits  - RBAC        │    │
│  │  - Network policies                                │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ Team A  │  │ Team B  │  │ Team C  │  │ Team N  │       │
│  │Namespace│  │Namespace│  │Namespace│  │Namespace│       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
└─────────────────────────────────────────────────────────────┘

Developer Workflow:
Backstage → Template → Git Push → Golden Path Pipeline → Build → Scan → Deploy → Observability
```

## Investigation

1. **Audit Current State**: Interview all 15 teams to understand their current CI/CD, monitoring, and infrastructure practices; catalog the top frustrations
2. **Map Variations**: Document where teams differ and where they should be standardized (security) vs. where autonomy is fine (language choice)
3. **Define Golden Paths**: Create 3-4 pipeline templates that cover 90% of use cases (Go microservice, Java API, Node.js frontend, Python data service)
4. **Select Platform Tools**: Backstage for portal, Tekton/GitHub Actions for CI/CD, Crossplane for infrastructure, Vault for secrets, OPA for policies
5. **Build Self-Service APIs**: Create APIs for namespace provisioning, database creation, secret management, and monitoring setup
6. **Implement Guardrails**: Deploy OPA/Gatekeeper policies for security and compliance requirements
7. **Measure Baseline**: Record current DORA metrics across all teams to establish improvement targets
8. **Pilot with One Team**: Start with one team, iterate based on feedback, then roll out to all teams
9. **Continuous Improvement**: Monthly developer satisfaction surveys, quarterly platform reviews

## Commands

```bash
# Install Backstage
npx @backstage/create-app@latest; cd my-backstage-app; yarn install; yarn dev

# Configure Backstage Software Catalog (catalog-info.yaml)
cat > catalog-info.yaml <<EOF
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Handles order processing and management
  annotations:
    github.com/project-slug: myorg/order-service
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: team-commerce
  system: e-commerce-platform
  providesApis:
    - order-api
  dependsOn:
    - resource:postgres-orders
    - resource:redis-cache
EOF

# Install Crossplane for infrastructure provisioning
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace

# Create a Crossplane composition for database provisioning
cat > postgres-composition.yaml <<EOF
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql
spec:
  compositeTypeRef:
    apiVersion: database.platform.io/v1alpha1
    kind: PostgreSQL
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.crossplane.io/v1alpha1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            engineVersion: "14"
            dbInstanceClass: db.t3.medium
            allocatedStorage: 20
EOF

# OPA/Gatekeeper - Install and configure policies
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts && helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace

# Policy: Require approved base images (OPA/Gatekeeper)
cat > require-image-scan.yaml <<EOF
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredimagepolicy
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredImagePolicy
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredimagepolicy
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not startswith(container.image, "registry.internal.com/")
          msg := sprintf("Container %v must use internal registry. Image: %v", [container.name, container.image])
        }
EOF

# Policy: Require resource limits
cat > require-resource-limits.yaml <<EOF
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresourcelimits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits
          msg := sprintf("Container %v must have resource limits", [container.name])
        }
EOF

# Apply Gatekeeper policies to production namespaces
kubectl apply -f require-image-scan.yaml -f require-resource-limits.yaml

# Create namespace with resources quotas and RBAC
cat > team-namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: team-commerce
  labels:
    team: commerce
    environment: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-commerce-quota
  namespace: team-commerce
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "20"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-commerce-developer
  namespace: team-commerce
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "deployments", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Tekton - Install for CI/CD pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Create a Tekton pipeline template (golden path)
cat > pipeline-template.yaml <<EOF
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: golden-path-java
spec:
  params:
    - name: repo-url
      type: string
    - name: image-name
      type: string
  workspaces:
    - name: shared-workspace
  tasks:
    - name: fetch-source
      taskRef: {name: git-clone}
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: $(params.repo-url)
    - name: build
      taskRef: {name: maven-build}
      runAfter: ["fetch-source"]
      workspaces:
        - name: source
          workspace: shared-workspace
    - name: security-scan
      taskRef: {name: trivy-scan}
      runAfter: ["build"]
      params:
        - name: image-ref
          value: $(params.image-name)
    - name: deploy
      taskRef: {name: kubernetes-deploy}
      runAfter: ["security-scan"]
      params:
        - name: environment
          value: "$(params.target-environment)"
        - name: image-ref
          value: $(params.image-name)
EOF
```

## Root Cause

1. **Lack of Standardization**: Each team independently chose tools and practices, creating 15 different ways to do the same thing
2. **No Platform Team**: Infrastructure was everyone's responsibility, so it was no one's priority
3. **Missing Self-Service**: Developers filed tickets for infrastructure, creating bottlenecks and delays
4. **Inconsistent Security**: Without enforcement, teams adopted security practices at their own pace
5. **Documentation Scattered**: Knowledge lived in individual repos, wikos, and team members' heads
6. **No Feedback Loop**: No mechanism to measure developer productivity or satisfaction
7. **Tool Sprawl**: 15 teams using 15 different CI/CD tools, monitoring solutions, and deployment strategies

## Immediate Mitigation

1. **Create a Platform Team**: Dedicate 3-5 engineers to building and maintaining the IDP
2. **Standardize CI/CD**: Roll out golden path pipeline templates to all teams within 30 days
3. **Deploy Backstage**: Launch the developer portal with service catalog and templates
4. **Implement Self-Service**: Build APIs for namespace provisioning, database creation, and secret management
5. **Enforce Security**: Deploy OPA/Gatekeeper policies for image scanning and resource limits
6. **Measure Baseline**: Record current DORA metrics to establish improvement targets
7. **Communicate**: Announce the platform initiative, gather feedback, and iterate

## Permanent Fix

1. **Platform as Product**: Treat the IDP as a product with a roadmap, feedback loop, and continuous improvement
2. **Golden Paths, Not Golden Cages**: Provide opinionated defaults that are easy to follow but allow escape hatches for特殊 requirements
3. **Developer Experience First**: Measure success by developer satisfaction, not just technical metrics; every infrastructure need should be provisionable via API or CLI
4. **Security by Default**: Policies that enforce standards without blocking development velocity
6. **Observability as a Service**: Every service gets dashboards and alerts automatically
7. **Continuous Measurement**: Track DORA metrics, developer satisfaction, and platform adoption monthly

## Monitoring

- **Platform Adoption Metrics**: Number of services on golden paths, API usage, self-service provisioning frequency
- **DORA Metrics**: Deployment frequency, lead time for changes, MTTR, change failure rate per team
- **Developer Satisfaction**: Monthly surveys measuring ease of use, documentation quality, support responsiveness
- **Platform Health**: API latency, error rates, provisioning success rates
- **Security Compliance**: Percentage of services with scanning enabled, policy violation rates; infrastructure cost per team, resource utilization
- **Onboarding Time**: Time from new developer join to first production deployment

## Security

- **Image Scanning**: Mandatory scanning before any image reaches production registry
- **RBAC**: Team-level access control — developers can only access their namespace and resources
- **Secret Management**: Vault-based secrets with automatic rotation, no secrets in code or environment variables
- **Network Policies**: Default deny all ingress/egress, explicit allow for required communication paths
- **Audit Logging**: Track all provisioning, deployment, and configuration changes for compliance
- **Base Image Standards**: Only approved base images from internal registry, no public Docker Hub pulls (policies version-controlled as code)

## Production Considerations

- **HA**: Backstage and platform APIs must be highly available — deploy with multiple replicas, use managed databases
- **Scalability**: Platform must support 50+ services and 15 teams without performance degradation
- **Reliability**: Platform failures shouldn't block deployments — graceful degradation, cached responses
- **Cost**: Platform infrastructure cost should be offset by developer productivity gains (typically 20-30% improvement)
- **Compliance**: SOC2, ISO27001 requirements for access control, audit logging, and change management
- **Operational**: Platform team needs on-call rotation, incident response, and continuous improvement process
- **Migration**: Gradual migration from team-managed to platform-managed — don't force big-bang; teams may opt out of specific golden paths when justified, with clear process

## Senior-Level Answer

I'd design an Internal Developer Platform with five pillars: a Backstage developer portal for service catalog and templates, golden path CI/CD pipelines using Tekton, self-service infrastructure provisioning via Crossplane, a pre-configured observability stack with Prometheus and Grafana, and OPA/Gatekeeper security guardrails. The platform team operates it as a product with a roadmap and feedback loop. Success is measured by DORA metrics improvement — targeting 2x deployment frequency and 50% reduction in lead time within 6 months. Key design principle: golden paths, not golden cages — opinionated defaults with escape hatches for special requirements.

## Architect-Level Answer

The Internal Developer Platform is a strategic investment in organizational velocity. I'd architect it as a layered platform: a developer experience layer (Backstage), a platform API layer (infrastructure provisioning, CI/CD, secrets), and a control plane layer (OPA policies, resource quotas, RBAC). The key architectural decision is treating the platform as a product — with a dedicated team, roadmap, and continuous improvement based on developer feedback. Measuring success through DORA metrics and developer satisfaction surveys ensures the platform delivers real value. The long-term vision is a platform that enables autonomous team delivery while maintaining organizational standards for security, compliance, and cost efficiency.

## Follow-Up Questions

1. How do you handle team resistance to adopting the platform — teams that want to keep their existing tooling?
2. What's the difference between Platform Engineering and DevOps, and should they be separate teams?
3. How do you handle multi-tenancy on Kubernetes — namespaces, resource quotas, and RBAC at scale?
4. How do you measure the ROI of a platform engineering investment to justify the headcount?
5. How would you design the platform differently for a regulated industry like healthcare or finance?
