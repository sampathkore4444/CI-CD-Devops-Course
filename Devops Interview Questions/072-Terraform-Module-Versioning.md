# 72. Terraform Module Versioning Strategy

## Scenario

Your organization maintains 30+ Terraform modules in a private registry. Teams are using different versions — some pin to exact versions (`v1.2.3`), others use ranges (`~> 1.2`), and some use `latest`. A breaking change in the VPC module (v2.0.0) removed an output that 8 teams depended on. Deployments across 3 environments broke simultaneously. The module registry has no formal versioning policy, no compatibility matrix, and no deprecation process. You need to implement a comprehensive versioning strategy.

## Interviewer Question

"We have 30+ Terraform modules with inconsistent versioning. A breaking change just broke multiple teams. How do you implement a module versioning and distribution strategy that prevents these issues?"

## What I Should Think About

- Semantic versioning (SemVer) for Terraform modules
- Module registry vs. Git-based module sources
- Version constraints: exact pinning vs. ranges vs. floating
- Backward compatibility and deprecation policies
- Module testing and CI/CD for module updates
- Cross-environment consistency
- Breaking change detection and communication
- Module dependency management

## Ideal Answer

**1. Implement Semantic Versioning (SemVer)**
- All modules follow MAJOR.MINOR.PATCH format
- MAJOR: Breaking changes (removing outputs, changing variable types)
- MINOR: Backward-compatible new features (new optional variables)
- PATCH: Bug fixes (correcting logic, documentation)

**2. Version Constraint Strategy**
- Root modules (apps): Pin to exact version (`v1.2.3`)
- Shared modules: Use pessimistic constraint (`~> 1.2`) for minor updates
- Never use `latest` or no version in production

**3. Module Registry Setup**
- Use Terraform Cloud/Enterprise or private registry
- Each module gets a changelog, README, and examples
- Automated testing in CI/CD before publishing

**4. Deprecation Policy**
- 2-sprint deprecation window for breaking changes
- Deprecation warnings in module output
- Communication via Slack + wiki

**5. Module Distribution**
- Private registry for production modules
- Git tags with SemVer for source control
- Module Mirroring for air-gapped environments

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              MODULE VERSIONING STRATEGY                   │
│                                                          │
│  ┌─────────────┐     ┌──────────────────────────┐      │
│  │ Module Repo  │────►│ CI/CD Pipeline            │      │
│  │ (Git + Tags) │     │ ┌─────────────────────┐  │      │
│  │ v1.0.0       │     │ │ Lint (tflint)       │  │      │
│  │ v1.1.0       │     │ │ Validate (terraform │  │      │
│  │ v2.0.0       │     │ │   validate)         │  │      │
│  └─────────────┘     │ │ Test (terratest)    │  │      │
│                       │ │ Security (tfsec)    │  │      │
│                       │ └─────────┬───────────┘  │      │
│                       └───────────┼──────────────┘      │
│                                   ▼                      │
│                       ┌───────────────────────┐         │
│                       │ Private Module Registry│         │
│                       │ (Terraform Cloud/      │         │
│                       │  Enterprise)           │         │
│                       └───────────┬───────────┘         │
│                                   │                      │
│                    ┌──────────────┼──────────────┐      │
│                    ▼              ▼              ▼      │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│              │ Team A    │  │ Team B    │  │ Team C    │ │
│              │ v1.2.3    │  │ v1.2.3    │  │ ~> 1.2   │ │
│              │ (pinned)  │  │ (pinned)  │  │ (range)  │ │
│              └──────────┘  └──────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Audit current module versions across all workspaces:**
   ```bash
   # List all Terraform files
   find . -name "*.tf" -type f | xargs grep -l "module" | head -20
   
   # Extract module sources and versions
   grep -rn "source\|version" --include="*.tf" . | grep -E "module\." | head -50
   
   # Check Terraform Cloud workspaces (if applicable)
   terraform workspace list
   for ws in $(terraform workspace list | awk '{print $2}'); do
     terraform workspace select $ws
     terraform state pull | jq -r '.modules[].resources | keys[]' | sort -u
   done
   ```

2. **Check for breaking changes in module history:**
   ```bash
   # Check git tags in module repo
   cd modules/
   git tag -l "v*" --sort=-version:refname | head -20
   
   # Check what changed between versions
   git diff v1.9.0..v2.0.0 --stat
   
   # Look for removed outputs
   git diff v1.9.0..v2.0.0 -- outputs.tf
   ```

3. **Identify teams using outdated or floating versions:**
   ```bash
   # Find all module blocks with version constraints
   rg -n 'version\s*=' --include="*.tf" . | grep -v "#" | sort
   
   # Find modules without version (floating)
   rg -n 'source\s*=' --include="*.tf" . -A2 | grep -v "version"
   ```

## Commands

```bash
# 1. Tag a new module version
cd modules/vpc/
git tag -a v1.3.0 -m "Add optional NAT gateway support"
git push origin v1.3.0

# 2. Publish to private registry
terraform registry push modules/vpc v1.3.0

# 3. Create module with proper structure
mkdir -p modules/vpc/{examples,tests,docs}
cat > modules/vpc/main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"
    }
  }
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "enable_nat_gateway" {
  type        = bool
  description = "Enable NAT Gateway (added in v1.3.0)"
  default     = false
}
EOF

# 4. Set version constraints in consuming code
cat > main.tf << 'EOF'
module "vpc" {
  source  = "app.terraform.io/org/vpc/aws"
  version = "1.3.0"  # Exact pin for production
  # OR
  version = "~> 1.3"  # Allow patch updates
}
EOF

# 5. Run tflint to check for compatibility
tflint --module
tflint --config .tflint.hcl

# 6. Run terratest for module testing
cd modules/vpc/tests/
go test -v -timeout 30m
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| No versioning policy | Enforce SemVer with CI/CD validation |
| Floating/latest versions in production | Enforce exact version pinning via policy as code |
| No automated testing before module release | Add Terratest + integration tests in CI/CD |
| No deprecation process | Implement 2-sprint deprecation window |
| No compatibility matrix | Create module compatibility documentation |
| No breaking change detection | Add breaking change detection in CI/CD |

## Immediate Mitigation

1. **Roll back the breaking module** — pin all teams to v1.9.0 (last working version)
2. **Fix the v2.0.0 module** — restore removed outputs as deprecated
3. **Notify all teams** via Slack and incident channel
4. **Publish v2.1.0** with backwards-compatible changes and deprecation warnings

## Permanent Fix

1. Implement SemVer enforcement in CI/CD pipeline
2. Add automated module testing (Terratest) before any release
3. Create a private module registry with version visibility
4. Implement version constraint policies via Sentinel/OPA
5. Create a module deprecation and migration guide template
6. Set up module dependency tracking across teams

## Monitoring

- **Module version drift detection** — weekly scan of all workspaces for version mismatches
- **CI/CD pipeline** — block module publish without passing tests
- **Deprecation alerts** — notify teams using deprecated module versions
- **Module usage metrics** — track which versions are in use across environments
- **Breaking change detection** — automated diff analysis between versions

## Security

- Modules should be scanned with tfsec/Checkov before publishing
- Use signed modules (GPG signing) to prevent tampering
- Restrict module registry access via IAM
- Audit module changes via Git history
- Implement module review process (PR review required)
- Consider using module mirroring for air-gapped environments

## Production Considerations

- **30+ modules** need a centralized registry — Terraform Cloud or self-hosted
- Cross-environment consistency requires same module versions across dev/staging/prod
- Module updates should be rolled out gradually (dev → staging → prod)
- Cost of Terraform Cloud: $20/user/month + $0.0001 per resource run
- Module compatibility testing across Terraform versions (1.3, 1.4, 1.5, 1.6)
- Consider using Terragrunt for dependency management across 12 environments

## Senior-Level Answer

"I'd implement a comprehensive module governance strategy: (1) Private registry with SemVer enforcement and automated testing, (2) CI/CD pipeline that requires linting, validation, Terratest, and tfsec before publishing, (3) Version constraint policies — exact pinning for root modules, pessimistic constraints for shared modules, (4) Deprecation policy with 2-sprint migration windows and automated notifications to consuming teams, (5) Breaking change detection in CI/CD that blocks releases unless properly versioned."

## Architect-Level Answer

"At the organizational level, I'd establish a Terraform Center of Excellence (CoE) that owns the module library. The CoE would define module standards, review breaking changes, and maintain a compatibility matrix. I'd implement a module marketplace where teams can discover and consume modules with version visibility. For cross-team coordination, I'd establish a module update cadence (monthly) with a shared changelog and migration guide. I'd also implement policy-as-code (Sentinel/OPA) to enforce versioning policies across all workspaces."

## Follow-Up Questions

1. "How do you handle a situation where a critical security patch requires a breaking change in a module?"
2. "What's your approach to testing Terraform modules that create real infrastructure?"
3. "How do you manage module versions across multiple Terraform workspaces with dependencies?"
4. "How would you implement a module migration guide for teams upgrading from v1.x to v2.x?"
5. "What's the difference between using Terraform Registry vs. Git sources for module distribution?"
