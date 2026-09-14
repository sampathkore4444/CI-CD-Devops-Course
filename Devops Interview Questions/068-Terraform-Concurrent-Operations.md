# 68. Two Engineers Running Terraform Concurrently

## Scenario

You have a production Terraform state stored in an S3 bucket with DynamoDB locking enabled. Two DevOps engineers are deploying independent changes to the same infrastructure. In a morning incident, Engineer A runs `terraform apply` and completes successfully. Then Engineer B, who ran concurrently, gets the error `Error: Error acquiring the state lock` and aborts. Now a check finds that the state in S3 has unexpected contents — the previous successful apply of Engineer B seems missing. The infra looks mixed: subnet security groups have A's change but the IAM roles have B's change. Both engineers insist their workspaces were correct and neither touched the other's code. You suspect the state file may be corrupted or stale. You need to recover, reconcile, and implement a workflow so two engineers can't clobber each other again.

## Interviewer Question

"Two engineers ran terraform apply at the same time against the same S3-backed state. One got a lock error and aborted, the other succeeded. Now the state looks inconsistent and the infrastructure reflects a mix of both changes. How do you diagnose what actually happened, recover the state safely, and prevent concurrent operations from ever corrupting it again?"

## What I Should Think About

- The DynamoDB lock should have prevented true concurrent applies — a lock error means B correctly aborted
- If B's apply "aborted", then B's changes should NOT be in state — but infra shows a mix — so something else happened
- Common failure paths: (1) lock timeout while lock is genuinely held, (2) apply NOT actually using the same state path (different workspace/region/bucket), (3) state pushed to S3 by a non-Terraform process or a different account, (4) stale/inconsistent state file overwritten, (5) `-refresh=false` from a stale local `.terraform/terraform.tfstate`, (6) destroy/prevent_destroy oddities, (7) partial apply that committed remote prior to error
- Terraform writes state to S3 on every apply (final write). If B broke during apply AFTER a partial local update but BEFORE remote write, remote keeps A's state → B's resources missing — but B's physical resources may exist (drift)
- If B's changes did land physically but not in state (because apply failed/aborted mid-way), subsequent `plan` will show "create" for B's resources → risk of "orphaned" duplicates
- The safest recovery: compare desired (B's code) vs current remote state vs real infra (Drift). Never blindly re-run apply over an inconsistent state
- Recovery tools: `terraform state rm`, `terraform state mv`, `terraform import`, `terraform refresh` (deprecated in favor of plan -refresh-only), and `terraform force-unlock` ONLY after confirming the lock holder is dead
- Lock ID safety: force-unlock is dangerous — verify with the process holding the lock first; if stale (crash), it's OK
- Prevent future: `-lock=true`, ensure both engineers use the same `-state` path, CI serialization, and add `checkov`/`tfsec` + drift detection in pipeline
- Use Terraform workspaces or separate state per environment to isolate concurrency (never share production state between two unrelated changes)

## Ideal Answer

**Step 1 — Understand what each engineer did (forensic, not fixing yet)**

```bash
# Who held/locked state
# In the S3 bucket, check metadata; in DynamoDB, look at the lock table
aws dynamodb scan --table-name terraform-locks
aws s3 ls s3://my-tfstate-bucket/env:/prod/  --recursive
```

**Step 2 — Check current remote state vs. code vs. live infra**

```bash
# Take the remote state
terraform state pull > /tmp/remote.json

# Which resources does the remote state have?
terraform show -json > /tmp/state.json
jq -r '.values.root_module.resources[].address' /tmp/state.json

# What does the plan think?
terraform plan -out=tf.plan
terraform show -json tf.plan | jq ...
```

**Step 3 — Detect the "mix"**

The infra is mixed (SG has A's change, IAM has B's). Candidates:
- B ran with a **different state path or a stale local cache**. Common cause: `.terraform.lock` / old `.terraform` directory, or `-state` flag pointing at different file, or two workspaces.
- B's apply "aborted" at the lock error, so B's resources were created on a *previous* successful apply held locally but never written to S3 (state drift).

**Step 4 — Reconcile state**

The golden path: bring state into agreement with reality, resource by resource, using targeted operations:
```bash
# For a resource that exists physically but not in state → import it
terraform import 'aws_iam_role.payments' 'payments-prod-role'

# For a resource that is in state but you want to remove → unlink, never destroy blindly
terraform state rm 'aws_security_group.app'

# Move between addresses
terraform state mv 'aws_security_group.app' 'aws_security_group.app.new'

# Force-unlock ONLY if you verified no active terraform process owns the lock:
aws dynamodb get-item --table-name terraform-locks --key '{"LockID":{"S":"<bucket>/<key>-md5"}}'
# Then:
terraform force-unlock <lock-id>
```

**Step 5 — Verify convergence**

```bash
terraform plan  # should be clean (no changes)
# then a fresh apply from CI, serialized (never two engineers direct)
```

## Architecture

```
    The concurrent apply race (what should happen):
    ┌────────────────────┐   ┌────────────────────┐
    │ Engineer A         │   │ Engineer B         │
    │ terraform apply    │   │ terraform apply    │
    └─────────┬──────────┘   └─────────┬──────────┘
              │                       │
              ▼                       ▼
    ┌──────────────────────────────────────┐
    │  DynamoDB lock table (already held)  │
    │  A acquires lock  ✓                 │
    │  B attempts lock    ✗ → blocks/error │
    └──────────────────────┬───────────────┘
                           │ A completes → writes final state to S3
                           ▼
                      S3 remote state
                           │
              B error "Error acquiring the state lock" → B aborts (safe)

    What actually went wrong (mix observed):
    B may have had a STALE local cache or a different state path/workspace,
    performed a partial apply earlier, or the error happened mid-apply
    → B's resources exist physically but not in S3 state → "drift"

    Recovery model:
       Desired (B's code)  ──▶ compare ──▶ Real infra (drift)
       Remote state (S3)   ──▶ reconcile (import/mv/rm) ──▶ converged plan
```

## Investigation

1. **Check DynamoDB lock table** — did B's lock error correspond to A holding the lock?
2. **Check lock duration** — how long was the lock held; was a force-unlock needed?
3. **Check both engineer's terraform version & workspace**:
   ```bash
   terraform version
   terraform workspace list
   {
      "query": "git log --oneline -5 -- .terraform.lock"
   }
   ```
4. **Check if B was on a different state path** (`-state`, `-state-out`, or backend with different bucket/key)
5. **Compare remote state with live infra** (`terraform plan -refresh-only` now shows drift — that list IS your reconciliation problem set)
6. **Identify which resources are "orphaned" (physical, no state) and which are "phantom" (state, no physical)**
7. **Check for a partial B apply** — B's apply may have exited after creating resources but before remote write; look at B's last apply output/CI logs
8. **Check CloudTrail/other log on S3 writes** — when was the state file last PUT and by whom
9. **Check `.terraform/terraform.tfstate` local copy drift** — `terraform validate` won't show it; but `plan` will use remote
10. **Ensure both engineers had `-lock=true`** — and confirm someone didn't `terraform apply -force-unlock` manually

## Commands

```bash
# Lock inspection
aws dynamodb scan --table-name terraform-locks --output json
aws dynamodb get-item --table-name terraform-locks \
  --key '{"LockID":{"S":"s3://my-state/env:/prod/terraform.tfstate-md5"}}'

# Pull remote state
terraform state pull > /tmp/remote-state.json
jq -r '.resources[].address' /tmp/remote-state.json

# Detect drift (state vs reality) — deprecated `refresh` replaced with plan -refresh-only
terraform plan -refresh-only -out=refresh.plan
terraform show refresh.plan | head -100

# Reconcile by importing missing resources
terraform import aws_iam_role.payments payments-prod-role

# Remove wrongly-tracked resource
terraform state rm aws_security_group.app

# Move
terraform state mv aws_security_group.app aws_security_group.app_v2

# Force-unlock — ONLY after confirming no active Terraform process owns the lock
terraform force-unlock 6ace8be0-3f1a-4f0b-8f9d-0123456789ab
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| DynamoDB lock genuinely held by A | Lock table shows A's info + timestamp | Wait for A; force-unlock ONLY if stale |
| B working with different state path/backend | Different bucket/key in B's `.terraform` | Standardize single backend config for the workspace |
| Stale local cache (.terraform) | Plan/apply made against local copy | `terraform init -reconfigure`, delete `.terraform` |
| Missing DynamoDB table / lock disabled | No `dynamodb_table` in backend | Enable DynamoDB table in backend config |
| Partial apply before lock error | Remote state older than physical resources | Import missing; reconcile drift |
| Different workspaces | Both ran `terraform workspace select` differently | Enforce one workspace per env |
| Human force-unlock prematurely | Lock acquired by B but released early | Document force-unlock conditions + dual approval |
| Terraform version mismatch | Both used different versions | Standardize via `required_version`, containerized runs |

## Immediate Mitigation

1. **Freeze prod applies** — communicate "no terraform apply" until reconciliation is done
2. **Confirm lock table is clean** — no stale lock; if a stuck lock exists, verify the owning process died (`ps`/CI job), then `terraform force-unlock`
3. **Snapshot the remote state** (copy to a date-stamped file/backup bucket) before touching anything
4. **Classify the drift**: use `terraform plan -refresh-only`, split resources into `import-needed` (physical but missing) and `remove-from-state` (phantom)
5. **Reconcile in CI using targeted `terraform import`/`state rm`/`state mv`** — never `apply` blindly against inconsistent state
6. **Verify with a clean `terraform plan`** showing zero unexpected changes
7. **Re-establish serialized deployment**: single CI pipeline that runs applies, with `-lock=true`, one at a time

## Permanent Fix

1. **Single CI pipeline owns all applies** — engineers never run `terraform apply` against prod directly; enable the `-auto-approve` only in CI with PR gates
2. **Enforce remote locking** — backend config:
   ```hcl
   backend "s3" {
     bucket         = "my-tfstate"
     key            = "env:/prod/terraform.tfstate"
     region         = "us-east-1"
     dynamodb_table = "terraform-locks"
     encrypt        = true
   }
   ```
3. **Enforce `-lock=true` in all commands** (`terraform plan -lock=true`, CI `-lock-timeout=10m`)
4. **Separate workspaces per environment** (`dev`, `staging`, `prod`) — different state keys, no cross-env clobbering
5. **Drift detection scheduled** in CI (nightly `plan -refresh-only` compared against baseline) → creates a ticket automatically
6. **Pin Terraform version** (`.terraform.lock.hcl` + containerized runs) to avoid state format mismatches
7. **Apply serialization with concurrency guard** — CI queue or a lock/DynamoDB blip check at pipeline start
8. **Every state-changing action reviewed** — `terraform plan` output attached to PR; `plan` must show "No changes" after approve

## Monitoring

```yaml
# Monitoring for state operations
- alert: TFStateLockAcquisitionError
  expr: rate(terraform_state_lock_errors) > 0
  # This comes from CI logs metric
  severity: warning

- alert: TFStateWriteFrequent
  expr: rate(terraform_state_write_total[10m]) > 5
  # state written more often than expected → suspicious loop
  severity: warning
```

Watch:
- DynamoDB table auto-paging (active LockID entries)
- CI job duration and success rate per env
- `.terraform.lock.hcl` hash changes = dependency drift
- S3 state file LAST_MODIFIED audit in CloudTrail (S3 PutObject on the state key)

## Security

- **Restrict write access to the state** — S3 bucket policy allowing `s3:PutObject` only from CI role, not engineers' local machines
- **Encrypt state at rest** (SSE-KMS) since it may contain secrets (RDS passwords, spot instances details)
- **Restrict DynamoDB lock table** writes similarly
- **Never store secrets in plaintext state** — use Secrets Manager/SSM Parameter Store + `data` lookups, or at minimum `sensitive` attributes
- **Audit all state modifications** via CloudTrail (`PutObject`, `DeleteObject` on state bucket)
- **Least privilege IAM** — engineers get `plan` only, CI role gets `apply`
- Backup the state bucket to a second region / version-enabled S3

## Production Considerations

- **Reliability**: versioning on the S3 state bucket with lifecycle to keep N versions; enable **Terraform audit to plan-review** so each apply is reviewed
- **Cost**: state storage is tiny; the cost risk is humans clicking apply — process is the "cost" lever
- **HA**: state availability — the state bucket is part of DR; keep a textbook copy off-region
- **Compliance**: state file = "source of truth" — treat as a governed artifact (retention, backup, immutability)
- **Operational**: handoff between engineers — use CI queue to serialize deploys; check-in `/review` history of state diff after every apply
- **DR**: test restore of a stale-but-clean state; RTO < 30 min by importing latest-known-good state

## Senior-Level Answer

"The lock error itself is *safe* — it's the system working. The problem is a state-reality mix: something applied changes that never landed in the remote state. First I'd freeze applies, snapshot the state, and check DynamoDB's lock table to confirm whether B's error was a real lock conflict or something worse (stale cache, different state path/backend, or workspace). Then I'd run `plan -refresh-only` to enumerate drift — every resource marked for create is a resource that physically exists without state tracking. I reconcile by `terraform import` for those, `state rm` for phantoms, and `state mv` for renamed resources, verifying each with a clean plan. Then I'd force-unlock only after confirming the owning process is dead. To prevent recurrence: all applies through CI with `-lock=true`, one queue, enforced one-workspace-per-env, staged drift detection nightly, and S3 state write permissions restricted to the CI role."

## Architect-Level Answer

"The root issue is tooling that allows parallel human mutation of a single shared artifact. Production-grade Terraform requires governance: (1) the state is a governed artifact — S3 + versioning + KMS + restricted IAM, and DynamoDB lock enforced; (2) all applies serialize through a single CI pipeline per environment with lock timeout, so concurrency is an impossibility rather than a rule; (3) drift detection runs nightly and creates tickets, so state-reality inconsistencies surface proactively; (4) platforms use separate state per environment and per logical unit (network, app, data) to reduce blast radius and contention; (5) a 'state hygiene' review after every incident — why did the lock fail, why did we have drift, what's the new invariant? The strategic answer: stop moving state by hand and never force-unlock casually. Automated, serialized, code-reviewed applies turn Terraform from a single point of contention into a reliable control plane."

## Follow-Up Questions

1. "DynamoDB locking — how does Terraform generate the lock ID, and what happens to the lock if the apply process is killed mid-write?"
2. "When is it safe to `terraform force-unlock` and when is it dangerous? Walk through the verification steps."
3. "Explain how `terraform state rm`, `terraform import`, and `terraform refresh-only` differ, and which to use when state has drifted."
4. "How would you design the CI pipeline and state layout for 5 environments across 3 accounts such that no two engineers ever contend on the same state?"
5. "A partial apply left resources physically created but not in state. Using `terraform import`, reconcile a specific `aws_db_instance` — exact commands and how you validate no duplicate is created."