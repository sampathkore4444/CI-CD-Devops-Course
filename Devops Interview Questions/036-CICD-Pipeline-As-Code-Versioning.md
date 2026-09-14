# 36. Pipeline as Code - Versioning and Review Strategy

## Scenario

Your organization has 15 development teams, each managing their own service repositories with CI/CD needs. Currently, every team has its own manually-configured Jenkins jobs managed through the Jenkins web UI. There's no version control for the pipeline definitions, no code review for pipeline changes, and fragmentation — 15 teams using 15 different CI/CD patterns. A developer accidentally breaks the Jenkins global config, and now half the pipelines are red. Management has mandated a move to Pipeline as Code using a single standard: your company has standardized on GitLab CI (but the same principles apply to GitHub Actions/Jenkinsfile). You need to:

1. Migrate all 15 teams from UI-based Jenkins to Pipeline as Code
2. Create a versioned, reviewed, and shared approach for pipeline definitions
3. Handle shared libraries (common build stages, security scans, deployment steps)
4. Ensure pipeline configuration changes go through proper code review
5. Support teams with diverse needs (Java, Go, Node.js, Python, mobile)

## Interviewer Question

"How do you approach migrating 15 teams from Jenkins UI jobs to pipeline-as-code with a shared library strategy? Walk me through your standardization, shared library design, versioning, and the rollout plan."

## What I Should Think About

- The goal: pipeline definitions become code — versioned in Git, reviewed via merge requests, tested before promoting
- Shared library strategy: common stages (security scan, Docker build, deploy) centralized, versioned, released
- GitLab CI component-based includes (`include: component:`, `include: project:`)
- The tension between standardization and flexibility
- Migration strategy: coexistence period (some teams on Jenkins, some on GitLab CI)
- A pipeline should be a config file in each service repo (drift-free — no UI-only config)
- Consider internal pipeline template releases (semantic versioning of shared pipelines)
- Governance: how to enforce that teams use the templates

## Ideal Answer

**Phase 0 — Inventory Current State**

1. Audit all 15 teams' existing Jenkins jobs: What stages? What tools? What parameters?
2. Identify common patterns (build, test, security scan, deploy) vs. unique needs
3. Create migration effort estimates per team

**Phase 1 — Define the Standard (the shared library)**

Create a centralized GitLab project (`ci-common`) containing:

1. **CI templates** — GitLab CI includes. Common jobs codified:
   - `build.yml` — Docker build + push
   - `tests.yml` — unit + integration test execution
   - `security.yml` — SonarQube + Trivy + gitleaks
   - `deploy.yml` — environment deployment (K8s, release manager)

2. **Shared shell scripts** — Dockerfile generation, release tagging, version bumping
3. **Semantic versioning** of the template library — teams pin to a version for stability:

```yaml
# Service .gitlab-ci.yml
include:
  - project: 'platform/ci-common'
    file: '/templates/build.yml'
    ref: v3.2.1   # ← pinned version — stable, reviewed changes only
  - project: 'platform/ci-common'
    file: '/templates/security.yml'
    ref: v3.2.1
```

**Phase 2 — Service Repo Adoption**

Each team adds a `.gitlab-ci.yml` to their repo, referencing the shared templates, plus service-specific steps:

```yaml
# service: lending-api/.gitlab-ci.yml
include:
  - local: '/.gitlab/ci/build.yml'
  - project: 'platform/ci-common'
    file: '/templates/security.yml'
    ref: v3.2.1

stages:
  - build
  - test
  - security
  - deploy

build:
  extends: .build_job

service_codegen:
  stage: build
  script:
    - ./generate-protos.sh  # service-specific build step

test:
  extends: .test_job

sonarqube:
  extends: .sonar_job

deploy-dev:
  extends: .deploy_dev_job
  environment: dev
```

**Phase 3 — Pipeline Change Review Process**

Pipeline-as-code changes go through normal code review:

1. Branch-based development of pipeline configs
2. Merge request to main requires:
   - Review from the CI platform team (for shared template changes)
   - Tests passing (use a canary service to validate pipeline templates)
   - For shared library changes: a review from at least 2 senior engineers + CI platform team
3. Semantic versioning of the shared library — teams pin to stable versions; breaking changes require a major version bump

**Phase 4 — Migration Rollout**

1. **Pilot**: Pick 3 teams — one Java, one Go/Node, one frontend — migrate first
2. **Validate**: Compare Jenkins vs GitLab CI results, catch gaps
3. **Train**: Documentation + office hours
4. **Roll out**: Remaining 12 teams over 2 sprints
5. **Sunset Jenkins**: After 1 month of dual-running, decommission

## Architecture

```
SHARED LIBRARY ARCHITECTURE:
┌──────────────────────────────────────────────────────────┐
│              platform/ci-common (GitLab project)         │
│                                                          │
│  templates/                                              │
│    ├── build.yml          ← Docker build + push          │
│    ├── tests.yml          ← unit + integration tests     │
│    ├── security.yml       ← SonarQube + Trivy + gitleaks │
│    ├── deploy.yml         ← K8s deployment               │
│    └── release.yml        ← semantic version + tag       │
│                                                          │
│  scripts/                                                │
│    ├── build-docker.sh    ← common build script          │
│    ├── tag-release.sh     ← version bump + git tag       │
│    └── verify-artifact.sh ← digest verification          │
│                                                          │
│  ├── .gitlab-ci.yml       ← CI for ci-common itself      │
│  └── CHANGELOG.md         ← template versions documented │
└──────────────────────────────────────────────────────────┘
        │   v3.2.1 (pinned by teams)
        ▼
┌──────────────────────────────────┐
│  15 service repositories         │
│  ┌────────────────────────────┐  │
│  │ lending-api/.gitlab-ci.yml │  │  ← include shared templates
│  │ ┌─────────────────────────┐│  │     overrides allowed for
│  │ │ include: ci-common @v3.2││  │     service-specific needs
│  │ └─────────────────────────┘│  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │ payments-api/.gitlab-ci.yml │  │
│  └────────────────────────────┘  │
│  ⋯                              │
└──────────────────────────────────┘

REVIEW WORKFLOW:
┌──────────────────────────────────────────────────┐
│ Developer changes .gitlab-ci.yml or ci-common    │
│ ──→ creates MR ──→ CI runs on MR branch ──→     │
│   review (CI platform team for shared changes)   │
│   → merge to main → teams pinned to old version  │
│   stay stable; unpinned teams get new version    │
└──────────────────────────────────────────────────┘
```

## Investigation

1. **Inventory existing Jenkins jobs** — stage breakdown, duplicate stages across teams
2. **Identify shared vs. unique pipeline logic** — build a matrix of common stages across all 15 teams
3. **Assess team maturity** — who's comfortable with GitLab CI vs. who needs training
4. **Check GitLab runner availability** — will existing runners handle 15 teams?
5. **Review security scanning needs** — SonarQube, Trivy, Bandit, dependency checks
6. **Document deploy targets** — who deploys to K8s, who to PaaS, who to mobile app stores
7. **Identify edge cases** — monorepo teams, legacy builds, unusual test frameworks
8. **Set up the pilot** — select heterogeneous teams (Java + Node.js) to validate the templates

## Commands

```bash
# GitLab CI: Include shared template with pinning
include:
  - project: 'platform/ci-common'
    file: '/templates/security.yml'
    ref: v3.2.1

# GitLab CI: Use .env to define reusable variables in shared template
# templates/build.yml
.build_job:
  stage: build
  variables:
    IMAGE_TAG: "$CI_COMMIT_SHORT_SHA"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$IMAGE_TAG" .
    - docker push "$CI_REGISTRY_IMAGE:$IMAGE_TAG"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always

# Local include with override
include:
  - local: '/.gitlab/ci/build.yml'
  - project: 'platform/ci-common'
    file: '/templates/common-functions.yml'
    ref: v3.2.1

# Test the shared library template — CI for ci-common itself:
# .gitlab-ci.yml of platform/ci-common
template-validation:
  stage: test
  script:
    - ./scripts/validate-templates.sh
  rules:
    - if: '$CI_MERGE_REQUEST_IID'
      when: always

# Migration: generate a .gitlab-ci.yml from an existing Jenkins job
# (idea: export & convert; use Jenkins Job DSL)
java -jar jenkins-cli.jar get-job lending-api > test-job.xml
# Convert XML job → YAML CI pipeline (write a parser script)

# Verify runners capacity before 15 teams hit the cluster
kubectl get pods -l app=gitlab-runner -o wide
kubectl top nodes --no-headers | sort -k5 -n
```

## Root Cause

| Root Cause | Prevention |
|---|---|
| Pipelines configured only in Jenkins UI (no version control) | Move to pipeline-as-code (`.gitlab-ci.yml` in each repo) |
| No shared templates — 15 teams reinvent the wheel | Centralized `ci-common` library with `include:` |
| No review of pipeline changes | Pipeline changes go through MR review like code |
| No validation that templates work | `ci-common` project runs its own CI validation |
| No versioning of templates | Semantic versioning + teams pin to known-good versions |
| Jenkins master is a single point of failure | GitLab CI control plane; ephemeral runners |

## Immediate Mitigation

1. **Pilot immediately** — pick 3 teams (Java, Node, Python) and migrate them to GitLab CI with a minimal shared template
2. **Restore the broken Jenkins config** from backup (from the XML backup) or export all jobs as YAML immediately so nothing is lost
3. **Freeze pipeline changes** — take a snapshot of all current Jenkins job configs
4. **Create the shared CI project** `ci-common` and start with security templates (which teams need most urgently before the migration)
5. **Document the migration playbook** and get sign-off on phase plan

## Permanent Fix

1. **Pipeline-as-Code enforced** — every repo must contain `.gitlab-ci.yml`; Jenkins UI configuration is forbidden going forward
2. **Semantic-versioned template library** — CI platform team owns `ci-common`; releases are tagged (v1.0.0, v2.0.0) and any template change requires:
   - MR to `ci-common` with the change + rationale
   - CI validation runs on the template
   - At least 2 reviewers (orchestrator + a consumer team)
   - Tag + release notes (breaking changes = major version bump)
3. **Branch protection** — the `main` branch of each service repo requires at least 1 reviewer for any change including `.gitlab-ci.yml`
4. **Change review discipline** — a pipeline-as-code change is a code change; review for: security scanning present? DRY (uses shared library)? correct environment deployments?
5. **Canary rollout of template versions** — before tagging v3.0.0, one pilot team runs with it for 1 week
6. **CI platform SLA** — the platform team owns runner infrastructure, template library, and support

## Monitoring

- **Pipeline-as-code coverage** — % of repos with a `.gitlab-ci.yml` (visibility metric)
- **Template adoption rate** — % of services using `ci-common` includes vs. "forked" custom pipelines
- **Shared template version usage** — which version is each team on? (pinned vs. `latest`)
- **Pipeline change frequency** — how often teams modify `.gitlab-ci.yml`
- **Runners utilization** — peak runner load during the migration period
- **Pipeline success rate** before/after migration (compare Jenkins vs. GitLab CI over 2 weeks)
- **Time-to-fix for broken pipelines** — did the incident response improve?

## Security

- Pipeline configs are code — secrets must never appear inline (use CI variables/Vault)
- Shared templates could introduce vulnerabilities to every team if not reviewed — the security scan template is the highest-risk code in the org
- Restrict who can modify `ci-common` — the shared library is a supply-chain attack vector
- GitLab CI variables: mark as `masked` and `protected`
- Audit git history of `.gitlab-ci.yml` files — who changed what
- Scan pipeline templates for hardcoded credentials (gitleaks on `ci-common` repo)

## Production Considerations

- **Scalability**: One shared library scales to 100 teams, not 15 — the template stays consistent
- **Reliability**: Pinned template versions isolate teams from breaking changes
- **Cost**: Shared templates reduce duplication → less CI infrastructure waste
- **Compliance**: The shared library standardizes security scanning across all teams
- **Operational**: Onboarding a new team = adding a `.gitlab-ci.yml` + training, not custom infrastructure
- **Governance**: The CI Federation (platform team + team representatives) owns the shared library roadmap

## Senior-Level Answer

"I'd create a shared pipeline template library in a dedicated GitLab project (`ci-common`), versioned with semantic versioning and consumed via `include:`. Teams pin to a specific version for stability. The shared library contains the common stages: build, security scan, Docker image build/push, and deploy. Service repos have a minimal `.gitlab-ci.yml` that includes templates and adds service-specific steps. Pipeline changes go through normal MR review — the CI platform team reviews shared template changes, and I'd add branch protection requiring review for `.gitlab-ci.yml` changes. I'd pilot with 3 teams from different stacks, validate against the old Jenkins results, then roll out to the remaining teams, keeping a 2-week dual-run period before sunsetting Jenkins."

## Architect-Level Answer

"This is an organizational change as much as a technical one. The architecture I'd propose has four layers:

1. **Standardization layer** — a pipeline-as-code standard: every repo has a `.gitlab-ci.yml`, all environments deploy via the same template patterns, all teams use the shared library. Enforced via MR branch protection and a settings/config standard (not manual audits).

2. **Shared capability layer** — `ci-common` as an **internal template marketplace**: build, test, security (Sonar + Trivy + gitleaks), secret scanning, Docker build, deploy, and release templates, all semantic-versioned with a release process (CI-validation, canary, release notes).

3. **Governance layer** — the CI platform team acts as a maintainer of the shared library (not a gatekeeper). Merge-request reviews, deprecation policy (e.g., breaking changes announced 2 versions ahead), and an adoption dashboard tracking template usage.

4. **Migration layer** — a phased rollout: pilot teams → validation against old Jenkins → roll out → dual-run → decommission Jenkins. Each team gets a migration playbook and pairing support.

The key architectural goal: **pipelines become a product** — versioned, tested, documented, and available for teams to consume, with the CI platform team providing the platform and the shared components while teams retain full ownership of their service-specific pipeline logic. This is what makes CI at enterprise scale sustainable — the 15th team inherits the same reliability and security as the first."

## Follow-Up Questions

1. "A team says the shared templates are 'too restrictive' and wants to fork `ci-common`. How do you handle this without alienating them?"
2. "Your CI platform team releases v4.0.0 of the shared library with a breaking change, but 5 teams are still on v2.x, missing critical security scans. How do you manage the upgrade?"
3. "How do you test/validate a shared template change before releasing it? (Given it affects 15 teams)"
4. "In a monorepo with multiple services, how do you structure `.gitlab-ci.yml` to only run affected pipelines?"
5. "Jenkins still runs for 2 legacy teams that can't move. How do you maintain the shared library standard while they stay on Jenkins?"