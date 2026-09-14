# 86. Database Migration Breaking Application

## Scenario

The platform team deployed version 2.4.1 of the application using a blue-green deployment strategy. The deployment included a database migration created with Flyway that added a new `email_verified` column to the `users` table and modified an existing index on `orders(user_id, created_at)` to include a `status` column. The migration was tested in staging and appeared backward-compatible. However, immediately after the deployment, the application started throwing `PSQLException: ERROR: column "email_verified" does not exist`. The migration shows as "SUCCESS" in the Flyway history. The previous application version (2.3.x) is still running on 50% of pods during the rolling deployment. Database is PostgreSQL 15 running on AWS RDS.

## Interviewer Question

"A database migration was applied during deployment. The migration added a column and modified an index, but the application is now throwing errors about missing columns. The migration supposedly was backward-compatible. How do you handle this incident? Walk me through investigation, rollback strategy, and prevention."

## What I Should Think About

- Is this a migration failure or an application version mismatch?
- Blue-green deployment implications - which pods are running which version?
- Migration ran successfully but the old application version doesn't know about the new column
- Backward compatibility means the migration should work for BOTH old and new app versions
- Index modification could have locked the table during migration
- How to safely rollback without losing data
- Flyway migration state management
- Whether the migration was reversible
- Communication with the team during the incident

## Ideal Answer

**Root Cause Analysis:**

The migration was NOT truly backward-compatible. Two problems:

1. The migration added `email_verified` column WITHOUT a `DEFAULT` value. Old application version queries `SELECT * FROM users` which returns all columns, but the code doesn't handle the new column type correctly, or worse, the column has `NOT NULL` without `DEFAULT` causing inserts to fail.

2. The index modification dropped the old index and created a new one. During this window, old app version queries using the original index plan fell back to sequential scans.

**Immediate Actions:**

1. Stop the migration application (pause the deployment)
2. Assess the damage: are writes succeeding? Are reads affected?
3. Either complete the migration fully (scale old pods to 0, ensure new pods are healthy) OR rollback the migration
4. If rolling back: the index change needs careful handling since data in the new column must be preserved

## Architecture

```
  DEPLOYMENT TIMELINE:
  ─────────────────────────────────────────────────
  T+0:00  Deploy v2.4.1 starts (blue-green)
          ├── Migration runs: ADD COLUMN + ALTER INDEX
          ├── New pods (v2.4.1) start coming up
          └── Old pods (v2.3.x) still running (50%)

  T+0:02  Errors begin:
          ├── Old pods try INSERT without email_verified
          ├── PostgreSQL: NOT NULL constraint violated
          └── Old pods crash-loop on errors

  T+0:05  Deployment detected as unhealthy
          ├── Blue-green: traffic still split
          └── Users seeing errors on old pods

  FLYWAY STATE:
  ┌──────────────────────────────────────────────┐
  │ flyway_schema_history                         │
  │ ─────────────────────────────────────────    │
  │ version │ description       │ success │ type │
  │ 2.3.0   │ baseline          │ true    │ SQL  │
  │ 2.3.1   │ add_user_roles    │ true    │ SQL  │
  │ 2.4.0   │ add_email_verify  │ true    │ SQL  │  ← PROBLEM
  │ 2.4.1   │ alter_order_index │ true    │ SQL  │  ← PROBLEM
  │                                              │
  │ Both marked SUCCESS even though app broke!   │
  └──────────────────────────────────────────────┘

  CORRECT MIGRATION STRATEGY:
  ┌────────────────────────────────────────────────┐
  │ Phase 1: Deploy migration (backward compat)   │
  │   ADD COLUMN email_verified BOOLEAN DEFAULT F │
  │   (no NOT NULL, has DEFAULT)                  │
  │                                               │
  │ Phase 2: Deploy new app version               │
  │   App uses new column                          │
  │                                               │
  │ Phase 3: (Optional) Set NOT NULL after deploy │
  │   ALTER TABLE users ALTER COLUMN               │
  │   email_verified SET NOT NULL;                 │
  └────────────────────────────────────────────────┘
```

## Investigation

**Step 1: Check migration status**
```bash
# Check Flyway migration history
flyway info -url=jdbc:postgresql://primary.db:5432/prod \
  -user=flyway_admin -password=*** locations=filesystem:./migrations

# Check what migration ran
psql -h primary.db -p 5432 -U app_user prod -c \
  "SELECT version, description, success, installed_by, checksum
   FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;"
```

**Step 2: Check if the column actually exists**
```sql
-- Verify column existence
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'users' AND column_name = 'email_verified';

-- Check index state
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';

-- Check for locks from migration
SELECT pid, relation::regclass, mode, granted, age(clock_timestamp(), query_start)
FROM pg_locks l JOIN pg_stat_activity a ON l.pid = a.pid
WHERE relation::regclass::text LIKE 'orders%'
AND NOT granted;
```

**Step 3: Check application pod versions and logs**
```bash
# Check which pods are running which version
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# Check error logs
kubectl logs -l app=api --tail=100 | grep -i "PSQLException\|column.*does not exist"

# Check deployment status
kubectl rollout status deployment/api-service
```

**Step 4: Check for data integrity**
```sql
-- Verify no data loss from index change
SELECT COUNT(*) FROM orders;

-- Check if any rows have NULL in the new column
SELECT COUNT(*) FROM users WHERE email_verified IS NULL;

-- Verify index is functional
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'active';
```

## Commands

```bash
# IMMEDIATE: Pause the deployment to stop bleeding
kubectl rollout pause deployment/api-service

# ROLLBACK DEPLOYMENT (not migration)
kubectl rollout undo deployment/api-service

# IF MIGRATION ROLLBACK IS NEEDED:
# Step 1: Create reverse migration script
cat > V2.4.1_rollback.sql << 'EOF'
-- Recreate original index
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- Drop the new index
DROP INDEX IF EXISTS idx_orders_user_status_created;

-- Remove the column (CAUTION: destroys data)
ALTER TABLE users DROP COLUMN IF EXISTS email_verified;
EOF

# Step 2: Apply rollback migration
flyway migrate -url=jdbc:postgresql://primary.db:5432/prod \
  -user=flyway_admin -locations=filesystem:./rollback

# Step 3: Remove the failed migration from Flyway history
psql -c "DELETE FROM flyway_schema_history
         WHERE version = '2.4.0' OR version = '2.4.1';"

# Step 4: Resume deployment of old version
kubectl rollout resume deployment/api-service
kubectl rollout undo deployment/api-service
```

## Root Cause

1. **Migration was NOT backward-compatible** - `ADD COLUMN ... NOT NULL` without `DEFAULT` breaks old application version inserts
2. **Index modification was destructive** - Old index dropped before new one fully created, causing query plan regression
3. **No migration validation in CI/CD** - Migration tested in staging against new app version only, not against old version
4. **Missing pre-deployment checks** - No check that old app version can tolerate the migration
5. **Single-step migration** - Migration did too many things in one step (add column + modify index)

## Immediate Mitigation

```sql
-- If column has NOT NULL without DEFAULT, fix it immediately:
ALTER TABLE users ALTER COLUMN email_verified SET DEFAULT false;
ALTER TABLE users ALTER COLUMN email_verified DROP NOT NULL;

-- Backfill existing rows
UPDATE users SET email_verified = false WHERE email_verified IS NULL;

-- If index change is causing issues, recreate the original index:
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at);
-- Then drop the new index after confirming queries work
DROP INDEX CONCURRENTLY idx_orders_user_status_created;
```

## Permanent Fix

1. **Zero-downtime migration pattern**: Every migration must be backward-compatible with the CURRENT application version
2. **Two-phase deployment**: Deploy migration first, then deploy application change
3. **Migration linting in CI**: Use tools like `gh-ost` or custom scripts to validate migration safety
4. **Index changes**: Always use `CREATE INDEX CONCURRENTLY` and `DROP INDEX CONCURRENTLY`
5. **Column additions**: Always add columns with `DEFAULT` value and `NULLABLE` first
6. **Column removal**: Drop columns in a separate deployment AFTER confirming no code references them
7. **Test against old version**: Run integration tests with the previous application version after migration
8. **Migration size limits**: Keep migrations small and focused - one logical change per migration

## Monitoring

```bash
# Monitor migration execution time
# Flyway provides migration timing - set alerts for migrations > 30 seconds

# Monitor for column-related errors
# Application logs: grep for "column.*does not exist"

# Pre-deploy checks
# 1. Run EXPLAIN on critical queries with the new schema
# 2. Check table lock duration during index creation
# 3. Validate migration reversibility

# Post-deploy validation
psql -c "SELECT version, description, success FROM flyway_schema_history
         WHERE success = false;"
```

## Security

- Flyway migration user should have minimum required privileges
- Audit all schema changes via `pg_audit` extension
- Store migration files in version control with mandatory code review
- Migration checksums in `flyway_schema_history` prevent tampering
- Separate migration user from application user (least privilege)

## Production Considerations

- **HA**: Migration runs on primary - ensure replication won't lag during schema change
- **Scalability**: Large table migrations (ALTER TABLE) can lock tables - use online schema change tools
- **Cost**: RDS storage may increase with new indexes - monitor storage utilization
- **Compliance**: Schema changes should be auditable with timestamps and authors
- **Operational**: Maintain a rollback runbook for every migration type
- **RTO**: Plan migration windows for low-traffic periods; have hot-fix migration ready
- **CI/CD**: Add migration validation step between staging and production deployment

## Senior-Level Answer

"The root cause is that the migration wasn't truly backward-compatible. The `ADD COLUMN NOT NULL` without a `DEFAULT` breaks old app version inserts, and the index replacement wasn't atomic. I'd immediately fix the column definition to be nullable with a default, then complete the deployment for the new version. For rollback, I'd restore the original index using `CREATE INDEX CONCURRENTLY` before dropping the new one. Going forward, I'd implement a two-phase deployment strategy: first deploy the migration (backward-compatible with current version), validate it, then deploy the new application code. I'd add migration linting in CI to catch non-backward-compatible changes and enforce `CREATE INDEX CONCURRENTLY` for all index operations."

## Architect-Level Answer

"This incident reveals a fundamental gap in our deployment pipeline. We need a schema change management strategy that separates migration deployment from application deployment. I'd implement a formal 'Schema Change Review' process where DBAs review migrations for backward compatibility. We should adopt tools like `gh-ost` or `pt-online-schema-change` for production schema changes that might lock tables. The CI pipeline should include a 'backward compatibility test' that runs the migration against a database with the old application version's test suite. Long-term, we should consider an expand-and-contract pattern for all schema changes, and implement schema versioning that's decoupled from application versioning. For critical systems, consider event sourcing or CQRS patterns that reduce schema change impact."

## Follow-Up Questions

1. "How would you handle this if the migration had already modified millions of rows and rollback would lose data?"
2. "Explain the expand-and-contract pattern for schema changes and when you'd use it."
3. "How would you test a migration for backward compatibility in a CI/CD pipeline?"
4. "What's the difference between Flyway and Liquibase, and how does this scenario change with each?"
5. "How would you implement online schema changes for a 1TB table without downtime?"
