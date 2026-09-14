# 35. Orchestrating Deployment Across Multiple Environments

## Scenario

Your enterprise application (a lending platform with regulatory implications) must be deployed through five environments in sequence: DEV → SIT (System Integration Testing) → UAT (User Acceptance Testing) → STAGING → PRODUCTION. Each environment has different resource configurations, different network policies, and different access permissions. UAT requires business sign-off from the product owner — historically this takes 3 days. STAGING requires the security team to run a credential assessment. PRODUCTION requires an approved change request (RFC) with a 2-hour deployment window, and during deployment, other teams' deployments to production are frozen. The release train runs every two weeks, and 12 services must all be deployed together. Currently, deployment is a half-manual process with scripts and email approvals, and the error rate is high. You're asked to automate this end-to-end while respecting the business processes.

## Interviewer Question

"How do you design and implement an automated deployment pipeline that handles promotion through DEV → SIT → UAT → STAGING → PRODUCTION with different approval gates, different configurations, and different test suites at each stage?"

## What I Should Think About

- This is a classic **promotion pipeline (progressively deploying the same artifact through environments)**
- The artifact is built ONCE and promoted — never rebuilt per environment
- Configurations are environment-specific but stored in code
- Approval gates must be integrated into the pipeline (manual approvals in Jenkins/GitLab/GitHub)
- Deployment to each environment requires its own credentials/access
- Need to handle Rollback at each environment
- Consider GitOps (ArgoCD) vs pipeline-based deployment (Jenkins/Bitbucket/GitLab)
- Need to track promotion state (which version is in which environment)
- Compliance/audit trail requirements for regulated environments

## Ideal Answer

**Design Principles**

1. **Build once, promote many** — the artifact flows through environments unchanged (only the environment config changes)
2. **Environment parity** — all environments use the same pipeline templates, differing only in variables/secrets
3. **Progressive gating** — each environment has a quality gate that must pass before promotion
4. **Traceability** — every deployment records the artifact ID, Git commit, and who/what approved it

**Pipeline Architecture**

```yaml
# GitLab CI / Jenkins: promotion pipeline
stages:
  - build
  - test-dev
  - deploy-dev
  - test-sit
  - deploy-sit
  - uat-approval
  - deploy-uat
  - security-gate
  - deploy-staging
  - prod-approval
  - deploy-production
  - smoke-test-prod
```

**Approval Gate Implementation**

```yaml
stage: uat-approval
approval:
  type: manual
  when: simple
  environment: uat
  wait_for: "product-owner-approve"

# Production approval with specific approvers (GitLab CI API-level control)
production_approval:
  script: echo "Waiting for RFC-approved deployment window"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  manual:
    allowed_users:
      - release-manager
      - head-of-engineering
```

**Environment Configurations**

Every environment gets its own set of variables, but the app is the same image:

```yaml
# dev config
DEV_DB_HOST: "dev-db.internal"
DEV_DB_NAME: "lending_dev"

# staging config
STAGING_DB_HOST: "staging-db.internal"
STAGING_DB_NAME: "lending_staging"

# production config
PROD_DB_HOST: "prod-db.internal"
PROD_DB_NAME: "lending"
```

**Deployment Order and Gates:**

1. **DEV** — deploy automatically on merge, run smoke tests
2. **SIT** — deploy after dev tests pass, run integration tests against the SIT environment
3. **UAT** — manual approval gate (product owner signs off), deploy, run UAT suite
4. **STAGING** — security credential assessment (automated + manual), deploy, run final validation
5. **PRODUCTION** — manual approval gate + RFC approval, deploy during the 2-hour window, run smoke tests

## Architecture

```
RELEASE TRAIN PIPELINE:
┌──────────────────────────────────────────────────────────────────┐
│                    Build Once, Promote Many                      │
└──────────────────────────────────────────────────────────────────┘
                    │
         ┌──────────▼──────────┐
         │   BUILD ARTIFACT    │  ← Image with tag: 2026-09-28-r2
         │   (tag + ECR push)  │
         └──────────┬──────────┘
                    │
         ┌──────────▼──────────┐
         │   DEPLOY TO DEV     │  ─────────── Auto deploy on merge
         └──────────┬──────────┘
                    │ Unit + smoke tests pass ✓
         ┌──────────▼──────────┐
         │   DEPLOY TO SIT     │  ─────────── Integration tests
         └──────────┬──────────┘
                    │ Integration tests pass ✓
         ┌──────────▼──────────┐
         │   UAT APPROVAL      │  ─────────── MANUAL: Product Owner
         │   (3 days window)   │                signs off
         └──────────┬──────────┘
                    │ Approved ✓
         ┌──────────▼──────────┐
         │   DEPLOY TO STAGING │  ─────────── Security assessment
         └──────────┬──────────┘  Credential scan, pen test (manual + auto)
                    │ Security pass ✓
         ┌──────────▼──────────┐
         │   PROD APPROVAL     │  ─────────── MANUAL: Release Manager
         │   (RFC + window)    │                + approved window
         └──────────┬──────────┘
                    │ Approved ✓
         ┌──────────▼──────────┐
         │ DEPLOY TO PRODUCTION│  ─────────── 2-hour window
         └─────────────────────┘

ENVIRONMENT TOPOLOGY:
┌─────┐   ┌─────┐   ┌─────┐   ┌─────────┐   ┌──────────┐
│ DEV │ → │ SIT │ → │ UAT │ → │ STAGING │ → │ PRODUCTION│
│ 1-2 │   │  1  │   │  1  │   │    1    │   │  prod 3  │
│ envs│   │ env │   │ env │   │   env   │   │  across  │
└─────┘   └─────┘   └─────┘   └─────────┘   │ AZ or DR │
     ↑         ↑        ↑         ↑          └──────────┘
     Team     Team(s)  Business   Security    RFC + SLA
     auto     tests    sign-off   review      window
```

## Investigation

1. **Document current promotion flow**: How does an artifact move from DEV to PRODUCTION today?
2. **Map configuration differences**: What config/variables change per environment?
3. **Identify regulatory/approval requirements**: What must be audited?
4. **Check current error points**: What breaks most often in the current half-manual process?
5. **Inventory credentials**: What access does each environment need?
6. **Determine test suite per environment**: What tests run at each stage?
7. **Assess deployment window requirements**: When is production allowed to change?

## Commands

```bash
# GitLab CI: Single promotion pipeline with environment gates
stages:
  - build
  - deploy
  - approve
  - verify

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy_dev:
  stage: deploy
  environment:
    name: dev
  script:
    - kubectl set image deployment/lending-api \
        lending-api=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n dev

# Manual approval gate for UAT
uat_approval:
  stage: approve
  script: echo "Awaiting Product Owner sign-off"
  when: manual
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'

deploy_uat:
  stage: deploy
  environment:
    name: uat
  script:
    - kubectl set image deployment/lending-api \
        lending-api=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA -n uat
  when: manual  # only runs after uat_approval clicked

# Production with window enforcement via schedule
deploy_prod:
  stage: deploy
  environment:
    name: production
  script:
    - ./scripts/prod-release.sh
  when: manual
  rules:
    - if: '$PROD_APPROVED == "true" && $CI_PIPELINE_SOURCE == "schedule"'

# Track promotion state with environment URLs and deployment records
environment:
  name: uat
  url: https://uat.lending.internal
  on_stop: uat_stop

# Audit trail - record deployment metadata
record_deployment:
  before_script:
    - echo "Deployed: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA to production"
    - echo "Approver: $APPROVER_EMAIL"
  script:
    - curl -X POST https://deployment-tracker.internal/api/deployments \
        -d "artifact=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

## Root Cause

| Root Cause | Prevention |
|---|---|
| Each environment has different deploy processes | Use pipeline templates / shared libraries across all environments |
| Secrets per environment not centralized | Vault + external-secrets, environment-scoped secret references |
| No environment parity | Same K8s manifests per env, only values differ (Helm values files) |
| Business sign-off not integrated | Manual approval gates in the pipeline itself |
| No audit trail | Every gate/step records approver, timestamp, artifact ID |
| No click-track of release state | Use GitLab Environments, ArgoCD application history for visibility |

## Immediate Mitigation

1. **For the current release cycle**: Use ArgoCD ApplicationSet or manual approval flow with the existing process, but track everything in one pipeline to reduce manual steps
2. **Create a 'release status dashboard'** so everyone sees which artifact is in which environment
3. **Eliminate the duplicate config problem**: introduce a single source of truth for env configs
4. **Automate the credential handoff** for STAGING security review (reduce manual credential sharing)

## Permanent Fix

1. **GitOps with ArgoCD**: Declarative deployment per environment; approval gates implemented via `argocd app sync --async` + manual promotion of the environment branch
2. **Approval gates as CI stages** — no human email step; the approval click IS the gate
3. **One pipeline template** — 12 services reuse a single `.gitlab-ci.yml` template via `include`
4. **Release manager service** — a small service that enforces release windows, RFC approval, and freeze rules
5. **Automatic promotion on schedule** — a cron-based pipeline runs nightly to promote from SIT → UAT (not blocking dev work)
6. **Test suite per environment defined in code** — environment variable `ENV` determines which suite runs

## Monitoring

- **Release status dashboard** — artifact version tracked per environment in real time
- **Pipeline health per environment** — pass/fail + duration metrics
- **Approval gate latency** — how long sign-offs take (is a business gate blocking?)
- **Deployment frequency** — DORA metric per environment
- **Configuration drift detection** — compare env configurations for unexpected deviations
- **Audit log stream** — every deployment action recorded with approver metadata

## Security

- **Access control per environment**: Dev vs production should require different credentials
- **Approval authorization**: Only authorized people can approve UAT/production promotions (Role-Based Access Control on the CI)
- **Secrets per environment**: Never reuse secrets across environments
- **Audit logging**: Regulatory compliance requires full traceability of production changes
- **Approver verification**: Verify who approved what; require MFA for production approvals
- **Environment isolation**: Limit blast radius if one environment is compromised

## Production Considerations

- **HA**: The promotion pipeline must be resilient — if an environment is down, pipeline should fail cleanly and retry
- **Scalability**: As the org grows to 100 services, the same template must work
- **Cost**: Five environments = five times the infrastructure cost; use environment teardown for non-prod environments when idle
- **Compliance**: In regulated industries, the audit trail IS a legal requirement
- **Operational**: The release calendar must be integrated with the pipeline (release windows)
- **Rollback**: Each environment should have a documented rollback path (e.g., deploy previous artifact)

## Senior-Level Answer

"I'd build a single promotion pipeline that takes the SAME artifact through all five environments. The artifact is built once, tagged, and pushed to the registry. Each environment stage uses the same deployment template but with environment-specific values. Approval gates are implemented as manual pipeline stages — UAT requires the product owner to click approve, production requires the release manager to click approve during the approved window. Configurations live in Helm values files per environment. I'd add automatic test suites per stage, audit logging of every promotion action, and a release status dashboard. The key is that the pipeline becomes the single source of truth for promotion state — no more email approvals or manual kubectl commands."

## Architect-Level Answer

"I'd implement a **progressive delivery architecture** with a clear separation between three concerns: artifact flow, environment state, and approval workflow.

1. **Artifact flow** — build once, promote by re-tagging in the registry (never rebuild). Artifact metadata carries the commit SHA, build ID, and validation results.

2. **Environment state via GitOps** — each environment has its own Git branch or directory representing the desired state. ArgoCD (or Flux) syncs the environment branch to the cluster. Promotion = merging to the next branch. `dev` auto-promotes from main; `sit` merges after integration tests; `uat` requires an approved merge request from the product owner; `staging` requires security sign-off; `production` requires RFC approval from the release manager, enforced by a Release Guard service that opens the deployment window during the approved time.

3. **Approval workflow** — approvals are native GitOps pull-requests with branch protection + required reviewers. MFA enforced for production merges. The audit trail is Git history itself — who approved what, when, is immutable and verifiable.

For the enterprise, I'd standardize a **Release Management Standard** — environment tiers (dev, sit, uat, staging, prod), approval levels per tier, and a release calendar integrated with the pipeline. Automated environments (dev/sit) can self-serve; gated environments require formal sign-off. This makes the process auditable, repeatable, and scalable to 100 services."

## Follow-Up Questions

1. "Your UAT approval gate blocks the pipeline for 3 days. How do you keep the development team productive while the product owner signs off? (Branches, feature flags, environment reuse)"
2. "Production requires a 2-hour window, but your pipeline is 45 minutes with flaky tests. How do you ensure deployment completes within the window?"
3. "How do you handle environment drift — when UAT gets manually modified outside the pipeline? (Drift detection, reconciliation)"
4. "What happens if a release fails at STAGING security assessment? Does the pipeline roll back, or block the PRODUCTION gate?"
5. "How do you ensure 'build once, promote many' works when production requires different build flags or compiled assets than dev?"