# 69. Terraform State File Corruption

## Scenario

A critical production environment is managed by Terraform with remote backend in S3 (`acme-terraform-state`) with DynamoDB locking. Yesterday, someone ran `terraform state pull` and manually edited the state JSON, then pushed it back. Today, `terraform plan` reads the state and concludes ALL 200 resources need to be recreated — including the RDS instance, the production EKS cluster, and the load balancer. The live infrastructure is healthy (users still reach the site, the DB is up). If you ran `apply`, Terraform would destroy and rebuild everything — disaster. You need to recover the state safely, bring it into alignment with reality, and restore `plan` to showing zero or minimal changes without touching production.

## Interviewer Question

"The Terraform state file in S3 appears corrupted — terraform plan wants to recreate every resource. The real infrastructure is fine. How do you recover the state without destroying production resources, and how do you restore a clean plan output?"

## What I Should Think About

- State corruption symptoms: plan wants to replace everything / weird `tainted` markers / null attributes / `InvalidFormat` or `Failed to load state`
- **Never run `apply`** against a suspicious state — that's how people destroy prod (worst-case practice to avoid; plan is safe, apply is not).
- Recovery options exist in priority order:
  1. **Restore the previous valid state** from versioning (S3 versioning on!) or Terraform Cloud history
  2. **`terraform state pull`** and repair the JSON manually (only if you understand the format)
  3. **`terraform import`** for each resource to rebuild state from reality
  4. **`terraform refresh-only`** (v1.2+) to realign state snapshot from infra without destroying
- S3 versioning on the state bucket is the single most important safety net — restore the previous version
- DynamoDB lock table holding a stale lock can also block recovery — check and clear safely
- If state includes `serial` mismatches, `-ignore-remote-version` or `-reconfigure`
- **Isolate and quarantine**: get a backup copy BEFORE touching
- If you don't have versioning (painful!), build state from `terraform import` for each resource — deterministic, matches reality
- Some "corruption" is caused by an attribute being opaque/unhappy (e.g., `id` wrong, `uuid`, or last-applied annotation)
- Environmental keys: check if the config changed (backend/drivers) vs state genuinely broken
- Terraform state is a JSON mapping of resource → attributes; plan "recreates" when `id` missing, or `omitfromstate` weirdness

## Ideal Answer

**Step 1 — Preserve and analyze**

```bash
# 1. Pull the broken state to a file for inspection (never modify remote yet)
terraform state pull > /tmp/corrupt-state.json

# 2. Back up the broken state & take the S3 old version
aws s3api list-object-versions --bucket acme-terraform-state --prefix env:/prod/terraform.tfstate \
  --query 'Versions[0:5].[VersionId,LastModified]'
aws s3api get-object \
  --bucket acme-terraform-state \
  --key env:/prod/terraform.tfstate \
  --version-id <VERSION_ID> \
  /tmp/previous-good.tfstate
```

**Step 3 — Diagnose what's wrong**

```bash
# 3a. Validate JSON
jq empty /tmp/corrupt-state.json && echo "valid json" || echo "BROKEN JSON"

# 3b. Compare to previous good state
jq -r '.resources[].address' /tmp/corrupt-state.json > /tmp/broken.txt
jq -r '.resources[].address' /tmp/previous-good.tfstate > /tmp/good.txt
diff /tmp/broken.txt /tmp/good.txt   # did numbers line up?

# 3c. Sanity check numberOfResources vs live infra
jq '.resources | length' /tmp/good.tfstate
# vs actual VPC/EC2/RDS via aws cli
```

**Step 4 — Recover**

Priority: restore good version. If AVAILABLE and you trust it's the last coherent state:

```bash
# Restore the previous good state from S3 versioning
aws s3api put-object \
  --bucket acme-terraform-state \
  --key env:/prod/terraform.tfstate \
  --body /tmp/previous-good.tfstate

# (Versioning on) then:
terraform plan   # should now show minimal/no changes
```

If NO good version exists (no versioning), rebuild via imports:

```bash
# For each resource that exists but has no usable entry:
terraform import aws_db_instance.payments payments-prod-db
terraform import aws_security_group.app sg-0abc123
# (Batch via script for the 200 resources)
```

Then realign everything else with reality:
```bash
terraform plan -refresh-only -out=refresh.plan
terraform apply refresh.plan  # updates state only, no resource changes
```

**Step 5 — Verify**

```bash
terraform plan   # expected: "No changes" or changes only for genuinely-broken resources
terraform validate
```

## Architecture

```
    State recovery decision flow
    ─────────────────────────────

              ┌───────────────────────────┐
              │ Corrupted state detected   │
              └────────────┬──────────────┘
                           │
          ┌────────────────▼───────────────┐
          │ Back up broken state           │
          │ (s3api get-object prom)        │
          └────────────────────────────────┘
                           │
              What does plan show? (SAFE: never apply)
                           │
          ┌────────────────▼───────────────────┐
          │ S3 versioning available & good      │
        ┌─┤ previous version?                   │
        │ └───────────────┬────────────────────┘
        │                 │  Yes
        │        ┌────────▼──────────┐
        │        │ Restore prev ver  │
        │        │ (aws s3api put)   │
        │        └────────┬──────────┘
        │                 │
        │        ┌────────▼──────────┐
        │        │ plan → No changes │
        │        └───────────────────┘
        │
        │   No good version
        ▼
    ┌───────────────────────────────────────────────┐
    │ Rebuild from reality:                         │
    │  1. terraform import each missing resource    │
    │  2. terraform plan -refresh-only (align for   │
    │     anything already tracked)                 │
    │  3. apply refresh.plan (only state changes)   │
    │  4. verify plan → No changes                  │
    └───────────────────────────────────────────────┘

    NEVER: terraform apply on a state that plans "recreate"
    ALWAYS: versioning on the state bucket + a stale-lock guard
```

## Investigation

1. **Get a backup copy of the current (broken) state** before any action
2. **Validate the JSON** — is it well-formed or truncated?
3. **Check S3 versioning** — is there a prior good version? (This decides the recovery path)
4. **Check `serial`** — Terraform uses `serial` as a monotonically increasing ID; if remote serial is lower than expected → stale remote
5. **Check the lock table** — stale DynamoDB lock can block the recovery apply
6. **Compare resources in state vs. live infra** — `terraform plan -refresh-only` (non-destructive) tells you exactly the delta
7. **Inspect specific resources in state** — missing `id`, null `attributes`, wrong `provider` path
8. **Check for accidental manual edits** — do the `schema_version`/`private` fields look plausible?
9. **Check who wrote last** — S3 version history / CloudTrail PutObject events
10. **Confirm config unchanged** — if you also ran `ierro change in code`, some of the "changes" might be genuine, not corruption

## Commands

```bash
# 1. Pull broken state
terraform state pull > corrupt.tfstate

# 2. JSON sanity
jq empty corrupt.tfstate

# 3. State attributes of one resource
jq '.resources[] | select(.address=="aws_db_instance.payments")' corrupt.tfstate

# 4. S3 versioning + listing old versions
aws s3api get-bucket-versioning --bucket acme-terraform-state
aws s3api list-object-versions --bucket acme-terraform-state --prefix env:/prod/terraform.tfstate --max-items 10

# 5. Restore previous good state (put back as current)
aws s3api put-object \
  --bucket acme-terraform-state --key env:/prod/terraform.tfstate \
  --body previous-good.tfstate

# 6. Clear a stale lock (after verifying process dead)
aws dynamodb scan --table-name acme-tf-locks
terraform force-unlock <lock-id>

# 7. For partial rebuild, import
terraform import aws_s3_bucket.static static-site-prod

# 8. Align state to real infra without destroying
terraform plan -refresh-only -out=refresh
terraform apply refresh

# 9. Final validation
terraform plan
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Manual edit of state JSON | Didn't match schema; `serial` wrong; null attributes | Restore from versioning / rebuild via import |
| S3 versioning disabled | No old version available | Enable versioning NOW |
| Accidental `terraform state mv` to wrong address | Resources moved/duplicated | Reverse with state mv |
| Provider version change | `schema_version` mismatch / provider bugs | Pin provider versions; upgrade with -refresh-only |
| Local cache vs remote mismatch | `plan` using local .terraform/tfstate | `terraform init -reconfigure` |
| Stale DynamoDB lock | Lock entry remains after crash | force-unlock validated |
| Remote backend drift | `terraform workspaces` / wrong AWS profile | Standardize backend config |
| Truncated write / partial PUT | JSON ends abruptly, half a resource | Restore last version; enable checksums on PUT |
| Import with wrong ID | `id` wrong → plan replaces | Import with correct AWS id |

## Immediate Mitigation

1. **Freeze all `terraform apply` on this environment** (communication + CI gate)
2. **Copy the broken state to a quarantine file** (`terraform state pull > corrupt-$(date).tfstate`)
3. **Attempt S3 version restore** — if a good prior version exists, restore it and run `terraform plan` to check
4. **If no good version**: run `terraform import` for resources that physically exist but lack state entries
5. **Never apply a "recreate" plan** — if plan still says recreate, hand-fix with targeted `state rm` + `import` rather than applying
6. **Restore service if prod was already destroyed** — use `terraform import` to absorb surviving infra (S3-backed backups help) then confirm DNS/LB recover

## Permanent Fix

1. **Enable S3 versioning + lifecycle** on the state bucket NOW — with the lock, plus `encrypt=true`
2. **Replace backend config to force versioning-friendly defaults in all projects** (via shared module / policy)
3. **Implement state integrity check** in CI — validate JSON + count resources before/after every apply, alert on anomalies
4. **Block manual pull/push** — IAM policy restricts `s3:PutObject` on state bucket to CI role only
5. **Documented state-submission process**: all state changes go through CI; `state rm`/`import` require PR review + plan artifact
6. **Backup strategy**: copy state to secondary bucket + DynamoDB lock snapshot scripted nightly
7. **Alerts on suspicious state writes** (PUT to state key from non-CI principal → incident)

## Monitoring

```yaml
- alert: TFStateCorruptWrite
  expr: s3_put_state_bucket_non_ci_principal > 0
  # from CloudTrail + tag principal
  severity: critical

- alert: TFStatePlanShowsRecreateAll
  expr: terraform_plan_resources_changed > 60% of total and plan_type == "all replace"
  severity: critical  # guards state integrity before apply

- alert: HeatStateFileModified
  expr: rate(cloudtrail_PutObject{key=~"terraform.tfstate"}) > 0
  severity: warning
```

Also: nightly `terraform plan -refresh-only` drift scan, and an alert when `serial` goes backwards.

## Security

- State contains provider data possibly with secrets/properties (DB endpoints, SGs) — enforce KMS SSE on bucket + DynamoDB KMS
- Never leak `terraform.tfstate` into web-accessible buckets or logs (secrets!)
- Restrict who can `PutObject` on the state — only CI role, no manual edits
- CloudTrail logs on any state write/delete; guard with SCP
- Rotate any credentials that appear inside state objects (import history)
- The recovery process should be run by a senior/architect with a runbook + audit trail

## Production Considerations

- **Reliability**: state is part of the operational control plane — versioning + lock + backups make it recoverable
- **HA**: restore state from the secondary copy / Terraform Cloud if primary S3 path is region-impaired
- **Cost**: state storage is cheap; the real cost risk is developer time in an outage — prevention pays
- **Operational**: maintain per-env state key pattern `env:/<env>/terraform.tfstate`, keep plan artifact in every PR
- **Compliance**: state audits + retention per policy; the state contains logical descriptions — treat as a controlled record
- **DR**: have a tested restore of state in a drill (restore version → plan clean → pass)

## Senior-Level Answer

"Never `apply` a plan that says 'create everything'. First, I snapshot the bad state and pull the previous version from S3 versioning — if a good version exists, restore it, verify the plan flattens to no changes, and I'm done. If there's no versioning, I rebuild state from reality: `terraform import` each resource that exists physically, then `terraform plan -refresh-only` + `apply` to align state attributes without changing infra. I also validate with `terraform validate` and diff the resource addresses between broken and previous state. The permanent fix is ensuring versioning on the state bucket, restricting state writes to the CI role, and adding nightly drift detection. Force-unlock only after confirming no live apply process owns the lock."

## Architect-Level Answer

"This is a control-plane integrity problem. The architectural mandates are: (1) the state bucket is a versioned, KMS-encrypted, IAM-governed, backup-protected artifact — versioning is a hard requirement, not an option; (2) no human gets write access to state; all changes flow through CI with `serial` monotonicity asserted; (3) every environment has a tested recovery drill (restore previous good version → plan shows zero drift); (4) drift detection nightly by `plan -refresh-only` vs baseline so any inconsistency surfaces in hours, not by bombshell; (5) disaster restores use `import`-based rebuild if versioning was unavailable. At design level I treat state as the operating record of the platform — with the same resilience you'd give a database. That way corruption is an incident-class event with a designated runbook, not a panic that leads someone to click 'recreate all'."

## Follow-Up Questions

1. "What exactly does the `serial` field in a state file represent, and why would a plan think everything needs to be recreated if it's wrong?"
2. "Compare `terraform state pull` + restore via `aws s3api put-object` vs `terraform state push` — what are the safety implications of each?"
3. "S3 versioning is off and you must rebuild state for 40 resources. Design the import-order and how you'd minimize risk of plan-time ambiguity."
4. "How would you detect state corruption impact (plan wants to recreate all) in a CI pipeline BEFORE any apply, as a guard?"
5. "Someone ran `terraform destroy` by mistake and killed prod. How do you recover with minimum downtime using state recovery techniques?"