# 70. Terraform Plan Shows Unexpected Changes

## Scenario

A routine Wednesday deploy. You run `terraform plan` as part of the standard PR pipeline before applying infrastructure changes. Instead of showing "No changes" for the unrelated module you touched, the plan shows **20 resources will be changed or destroyed** — security groups, IAM roles, subnet associations, and a couple of load balancers. You didn't modify any of the Terraform code for these resources. The plan would recreate or replace critical network and IAM infrastructure. Something silently drifted. Your job: figure out why the plan shows unexpected changes, decide whether it's safe, and fix the underlying cause so plans are predictable again.

## Interviewer Question

"Terraform plan unexpectedly shows 20 resources being changed or destroyed even though you didn't touch the code. Walk me through how you'd diagnose this, determine if the changes are real or drift-induced, and how you'd safely proceed."

## What I Should Think About

- Plan "unexpected changes" are almost never random — they're the difference between the *desired config* (HCL) and the *current state + live infra*. Unexpected changes mean either the config, the state, or the live infra changed.
- Main causes:
  1. **Live infra drifted** (someone changed in console, another tool, a service like AWS Bot, or CI)
  2. **State drift** (stale/out-of-date state, missing attributes)
  3. **Provider/config drift** (provider version updated, or resource attributes became non-optional / defaults changed)
  4. **Code in the branch includes changes merged by someone else** (CI merged main — a classic)
  5. **Terragrunt/workspace misuse** — plan applied to wrong env
  6. **Environment variables / default region changed** — resources get new IDs
  7. **Provider schema versioning** — new provider version with different `id` semantics, `force_new` flags (e.g., `aws_iam_role` `path` or `name_prefix` changes) → sets `force_recreate=*`
  8. **Backend/workspace change** — different state file → all "new"
- "Changed" vs "Destroyed" matters: destroy/recreate (force_new) is more dangerous than in-place modify
- Safety protocol: **plan is safe to run**, apply is NOT. Block apply on unexpected changes; add a human-review gate
- Techniques:
  - `terraform plan -detailed-exitcode` returns 2 when changes exist — CI can gate
  - Use `terraform show` on the plan to inspect each resource's `before` vs `after`
  - Filter plan output to `-target` or specific address for focused diagnosis
  - Compare against `git log` of the infra repo — maybe someone merged
  - Compare against last known-good plan: `terraform show <previous-plan-file>` (store plans as artifacts)
  - Check provider version: `terraform version` and `.terraform.lock.hcl` 
  - Use `~/.terraform` cache freshness
- AWS-specific: check things like default VPC, or security-group rules that Terraform tracks but AWS auto-adds (e.g., Classic Load Balancer SG, RDS modifications)
- Drift reconciliation strategy: either **adopt the drift** (plan apply accepts reality and updates state) or **revert the drift** (fix the live resource back to desired). Decide per resource!

## Ideal Answer

**Step 1 — Read the plan carefully (plan is safe)**

```bash
terraform plan -out=/tmp/plan.out
terraform show /tmp/plan.out
# Filter to the "unexpected" changes
terraform show /tmp/plan.out | grep -E '^  #|~ |- ' | head -80
# Inspect one resource in full before/after
terraform show -json /tmp/plan.out | jq '.resource_changes[] | select(.address=="aws_security_group.web") | .change'
```

**Step 2 — Diagnose why (4 buckets)**

**A. Is it in the code?** Compare current branch to the one that produced the last known-good plan:
```bash
git log --oneline -10
git diff main..HEAD -- infrastructure/
# If no infra code changes → not code
```
**B. Is it provider drift?**
```bash
terraform version
grep -A2 'aws' .terraform.lock.hcl
# Compare with CI's pinned version
```
**C. Is it state drift?**
```bash
terraform state list | sort > current_state.txt
# compare against expected list
```
**D. Is it live infra drift?** — did the cloud change:
```bash
# Example: was a security group modified in console?
aws ec2 describe-security-groups --group-ids sg-xxx --query 'SecurityGroups[0].IpPermissions'
# Compare with what terraform wants
```

**Step 3 — Categorize each unexpected change as REAL, DRIFT, or HARMONIZE**

- **REAL** (code changed deliberately → but you didn't code any → so REAL means someone merged):
  - verify with git; if real, it's expected; treat as a code review artifact
- **DRIFT** (live infra differs from desired): 
  - choose *adopt* (apply to update state) or *revert* (fix infra) based on which direction the business wants
- **HARMONIZE** (provider schema change → new attributes, defaults): plan applies safely; test in a copy of state (plan on `-module` insight)

**Step 4 — Contain and gate**

```bash
# Gate: never apply code around unexpected plan changes on prod without review
# Use an approval gate in CI: plan must be 'No changes' or explicitly approved
terraform plan -detailed-exitcode   # 0=no diff, 1=error, 2=changes
```

## Architecture

```
    Unexpected plan → where does it come from?
    ──────────────────────────────────────────

    Desired (HCL/module) ──────────┐
                                  ├──▶ terraform plan (compares) ──▶ changes?
    State (S3/DynamoDB) ──────────┤
                                  │
    Live infra (AWS) ─────────────┘
                                  │
                                  ▼
    ┌──────────────────────────────────────────────┐
    │ classify each resource change:               │
    │  A. Real (code changed)           git log     │
    │  B. Provider/schema upgrade       lock file   │
    │  C. State stale/drifted           state list  │
    │  D. Live infra drift (console/CI) describe    │
    └──────────────────────────────────────────────┘

    Decision per resource:
      drift adopt  → terraform apply (state ↔ infra)
      drift revert → fix live infra manually or with apply -target
      real         → review through PR gate, apply in CI
```

## Investigation

1. **Confirm it's not a merge problem** — check `git log` / branch state ("did someone update main and rebase my branch"?)
2. **Check provider version in both code and `.terraform.lock.hcl`** — was it upgraded? Was it pinned?
3. **Check the backend target** — did the plan run against the same state key (prod vs dev)?
4. **Check workspaces** — `terraform workspace list` might have selected the wrong one
5. **Compare the plan to the last stored plan artifact** — do the changed resources match a live-infra change in that window?
6. **Inspect attributes** — `before` vs `after` for each resource tells you the nature (add/remove/modify/force_recreate)
7. **Check AWS change events** — was a security group edited via console by a teammate in the last 24h?
8. **Check `terraform refresh` legacy drift** — v1.x uses refresh by default; drift may persist
9. **Check for tooling competition** — another IaC tool (CloudFormation, Ansible, scripted `aws cli`) also touching the same resources
10. **Validate with a clean refresh-only plan**, and (option for prod) plan against a **copy of state** to be extra safe before any apply

## Commands

```bash
# Detailed exit code (0 no change, 1 error, 2 changes)
terraform plan -detailed-exitcode

# Show the unexpected changes
terraform show -json plan.out | jq -r '.resource_changes[].address' | sort

# Focus-diagnose one resource
terraform show -json plan.out | jq '.resource_changes[] | select(.address=="aws_security_group.web") | .change.before, .change.after'

# Compare branch to last good
git diff --stat HEAD..origin/main -- modules/ infrastructure/

# Check provider locks
cat .terraform.lock.hcl | grep -A3 aws

# Check AWS-side drift for a specific SG
aws ec2 describe-security-groups --group-ids sg-xx \
  --query 'SecurityGroups[0].IpPermissions' --output json
# compare to the HCL

# Diff two plan files (use `terraform show -json` and jq)
terraform show -json old.plan > /tmp/old.json; terraform show -json new.plan > /tmp/new.json
diff <(jq -r '.resource_changes[]|.address' /tmp/old.json) <(jq -r '.resource_changes[]|.address' /tmp/new.json)

# State list for sanity
terraform state list | wc -l
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Teammate merged main / infra code | `git log` shows infra commit; branch stale | Rebase + plan again; PR review gate |
| Provider version upgraded | `.terraform.lock.hcl` changed; `before/after` shows `force_recreate` | Re-pin, plan-review with -target for risky roles |
| Live infra drift (console edit) | AWS describe differs from HCL | Adopt (`apply`) or revert — per resource decision |
| State stale (old provider)` | Resource has stale `id` or schema mismatch | `terraform plan -refresh-only` to align |
| Wrong workspace/backend | `workspace list`/state key mismatch | Verify `-state` path + env, never trust CWD |
| Another tool edits same res | CloudTrail shows run by other tool + interval | IaC ownership: one tool per resource |
| Missing `-reconfigure` | Local cache stale after backend change | `terraform init -reconfigure` |
| AWS default-change surface | e.g., new default SG rule auto-added | Pin `ingress`/`egress` explicit; ignore diff on managed defaults |

## Immediate Mitigation

1. **Do NOT apply.** Freeze deploy pipeline on prod.
2. **Snapshot state** (`terraform state pull > backup-$(date).tfstate`) so you can diff later.
3. **Preserve the plan artifact** (`terraform plan -out`) — do not lose it; it's the differential record.
4. **Isolate which resources** truly changed via `terraform show -json`.
5. **If it's genuinely a live-infra drift and the drift is harmless** (e.g., auto-added SG rule): adopt by `apply`, or use `-target` for just that.
6. **For high-risk resources (IAM roles, LB):** on `force_recreate`/`destroy`, revert live infra manually or veto apply until reviewed.
7. **Add degrade-safe approval**: the change is real only if reviewed; if not reviewed, revert.

## Permanent Fix

1. **Plan-review gate**: CI stores every plan artifact; a human must approve any plan that contains riskier-than-modify changes
2. **Drift detection & diff-baseline**: nightly `terraform plan -refresh-only` compared against a stored baseline; any new delta → alert + ticket
3. **Provider/version pinning**: pin provider version (`.terraform.lock.hcl` committed; CI verifies hash), upgrade in a separate reviewed PR
4. **IaC single-ownership**: apply policy one-tool-per-resource; prevent orphan AWS console edits with SCP/guardrails for critical resources
5. **Full `git` audit trail** on infra code — all env changes via PR, `terraform plan` output attached
6. **State hygiene**: S3 versioning + DynamoDB lock + nightly snapshots; alert on plan that changes > X resources
7. **Runbook**: 'unexpected changes' flow — classify, decide adopt vs revert, and document decisions

## Monitoring

```yaml
- alert: TFUnexpectedChanges
  expr: terraform_plan_resources_changed_tracker{category="unexpected"} > 0
  # emitted by CI after plan parse — captures large/force_recreate changes
  severity: warning → critical if includes IAM role or LB

- alert: TFProvisionerStuckForGenesis
  expr: terraform_last_plan_duration > threshold
  # signal that a provider upgrade is dancing plans
```

Store plan artifacts with metadata (`created_at`, `branch`, `status`) so future incidents can diff at a click.

## Security

- Never auto-apply plans that touch IAM roles/policies or security groups on prod — blast radius is severe
- Guard: plan approving principal must not be the same as the applying principal (separation of duties)
- Watch for privilege escalation in a drift-driven change (e.g., altering an IAM role policy inline in console = accidental)
- Validate that drift is not a sign of compromise (unexpected changes to SGs/LBs are a classic indicator)
- Keep state encrypted & in restricted IAM; never apply with a broad-alias `IVA:AdministratorAccess` session

## Production Considerations

- **Reliability**: unexpected changes that force recreates are the number-one source of "Terraform broke prod" — block auto-apply accordingly
- **Cost**: drift can silently delete/create instances/volumes — plan-review prevents surprise billing too
- **Compliance**: an unmodified-but-recreated resource violates change audit rules; plan-baselining meets that need
- **Operational**: adopt/drift decisions need a clear owner each time; document in the ticket
- **HA/DR**: treat the plan artifact as a config record; store per deploy for replayability
- **Multi-env**: same pattern repeated per env with strict env-bound workspace names

## Senior-Level Answer

"I never apply a plan that surprises me. First I isolate the source: is it code (someone merged against my branch), is it provider (version bump changing `force_recreate`), is it stale state, or is it live-infra drift (console edits)? I inspect the plan with `terraform show -json` and classify each resource change as adopt, revert, or real. Git diff tells me if code changed; `.terraform.lock.hcl` plus `terraform version` tells provider; a `plan -refresh-only` shows drift; AWS describe cross-checks console edits. Then per-resource: adopt if the drift is harmless, revert if it would destroy something important, treat it as code review if it's a merge. I gate CI so plans with destroy/recreate need explicit human approval and store plan artifacts for future diffing — and I fix the systemic cause: single-ownership IaC, provider pinning, and nightly drift baselining."

## Architect-Level Answer

"Unexpected plan changes are the operational bug report of an IaC estate — every one of them means the declared intent, the recorded state, and reality are inconsistent. Architecturally I'd tackle four layers: (1) **Config** — single-branch trunk workflow where infra changes only land via reviewed PRs, and `terraform plan` output is part of the PR; (2) **Provider** — locked versions, controlled upgrades via dedicated PRs, so schema drift is never silent; (3) **State** — versioned, locked, nightly-refreshed, and baselined so any delta is visible within hours; (4) **Runtime** — drift monitoring that compares live infra to state and alerts on console edits. The plan itself becomes a controlled artifact stored per deploy. The strategic rule: no unapproved diff is ever applied, and any drift is either intentionally adopted in a reviewed change or reverted to declared state. That turns '20 unexpected changes' from a Wednesday surprise into a beloved, diffable, reviewed workflow."

## Follow-Up Questions

1. "Explain the difference between an in-place modification, a 'force new resource', and a resource destroy/recreate in a Terraform plan — and which are riskier."
2. "A teammate edited a security group rule in the AWS console. Show how you detect that drift, how you decide adopt vs revert, and the exact commands."
3. "What provider-level behaviors in the AWS provider trigger `~` (in-place) changes silently, and how do you keep them from surfacing as unexpected diffs?"
4. "Design a CI pipeline that plans in a feature branch, diffs against the last-production plan, and blocks the PR if it contains a destroy — include the actual Terraform/CLI snippets and thresholds."
5. "How do `-refresh-only`, `-target`, and `-replace` each give you control when you must fix a single drifted resource without applying the rest of the plan?"