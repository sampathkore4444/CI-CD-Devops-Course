# 20 — Database CI/CD: Flyway & Liquibase

> **Goal:** Master database change management — how banks safely migrate schemas in CI/CD pipelines.

---

## 🔍 Why Database CI/CD is Critical

```
Application Code:
  Deploy new version → If broken, rollback instantly
  (Docker image rollback: 30 seconds)

Database Schema:
  Deploy migration → If broken, can't rollback easily!
  (Schema change affects ALL data)
  
  WRONG: ALTER TABLE → DROP COLUMN → Data lost forever
  RIGHT: ADD COLUMN → Deploy code → Verify → DROP COLUMN later
```

### The Expand-Contract Pattern

```
Phase 1: EXPAND (backward compatible)
┌─────────────────────────────────────────────────────────────┐
│  ALTER TABLE transactions ADD COLUMN gst_amount DECIMAL;    │
│  (Old code ignores new column, new code uses it)            │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
Phase 2: DEPLOY (code works with both schemas)
┌─────────────────────────────────────────────────────────────┐
│  Deploy new application version                             │
│  (Reads/writes gst_amount if present, ignores if not)       │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
Phase 3: CONTRACT (cleanup old schema)
┌─────────────────────────────────────────────────────────────┐
│  Deploy code that no longer references old columns          │
│  THEN: ALTER TABLE transactions DROP COLUMN old_column;     │
│  (Only after all application instances are updated)         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Database Migration Tools

| Tool | Type | Best For |
|------|------|----------|
| **Flyway** | SQL-based | Simple migrations, Java projects |
| **Liquibase** | XML/YAML/SQL | Complex changes, rollback support |
| **Alembic** | Python | Python applications |
| **Rails Migrations** | Ruby | Ruby on Rails |

---

## 📋 Flyway Example

```sql
-- V1__create_transactions_table.sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    type VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- V2__add_reference_column.sql
ALTER TABLE transactions ADD COLUMN reference VARCHAR(50);
CREATE UNIQUE INDEX idx_transactions_reference ON transactions(reference);

-- V3__add_gst_columns.sql
ALTER TABLE transactions ADD COLUMN gst_amount DECIMAL(10,2) DEFAULT 0.00;
ALTER TABLE transactions ADD COLUMN gst_rate DECIMAL(5,4) DEFAULT 0.0000;
ALTER TABLE transactions ADD COLUMN gst_inclusive BOOLEAN DEFAULT FALSE;

-- V4__backfill_gst.sql
UPDATE transactions 
SET gst_amount = amount * 0.18,
    gst_rate = 0.1800,
    gst_inclusive = FALSE
WHERE transaction_date >= '2026-04-01'
  AND gst_amount = 0.00;

-- V5__add_audit_columns.sql
ALTER TABLE transactions ADD COLUMN created_by VARCHAR(50);
ALTER TABLE transactions ADD COLUMN updated_by VARCHAR(50);
ALTER TABLE transactions ADD COLUMN version INTEGER DEFAULT 1;
```

### Flyway in CI Pipeline
```yaml
# .gitlab-ci.yml
migrate-dev:
  stage: migrate
  script:
    - flyway -url=jdbc:postgresql://dev-db:5432/banking \
        -user=$DEV_USER -password=$DEV_PASS \
        -locations=filesystem:src/main/resources/db/migration \
        migrate
    - flyway info
    # +-----------+---------+-------------+------+---------------------+---------+----------+
    # | Category  | Version | Description | Type | Installed On        | State   | Undoable |
    # +-----------+---------+-------------+------+---------------------+---------+----------+
    # |           | 1       | create ...  | SQL  | 2026-09-04 10:00:00 | Success | No       |
    # |           | 2       | add ref ... | SQL  | 2026-09-04 10:00:01 | Success | No       |
    # |           | 3       | add gst ... | SQL  | 2026-09-04 10:00:02 | Success | No       |
    # |           | 4       | backfill .. | SQL  | 2026-09-04 10:00:03 | Success | No       |
    # |           | 5       | add audit . | SQL  | 2026-09-04 10:00:04 | Success | No       |
    # +-----------+---------+-------------+------+---------------------+---------+----------+

migrate-staging:
  stage: migrate
  script:
    - flyway -url=jdbc:postgresql://staging-db:5432/banking \
        -user=$STAGING_USER -password=$STAGING_PASS migrate
    # Run data integrity checks
    - python scripts/verify_migration.py --env staging

migrate-prod:
  stage: migrate
  when: manual
  script:
    # Backup first
    - pg_dump -Fc banking > backup_$(date +%Y%m%d_%H%M%S).dump
    # Apply migrations
    - flyway -url=jdbc:postgresql://prod-db:5432/banking \
        -user=$PROD_USER -password=$PROD_PASS migrate
    # Verify
    - python scripts/verify_production.py
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Safe Column Addition

**Context:** Add GST (Goods and Services Tax) columns to transactions table.

```sql
-- Step 1: Add columns (EXPAND)
-- V6__add_gst_columns.sql
BEGIN;

-- Add new columns (nullable, no data loss)
ALTER TABLE transactions 
ADD COLUMN IF NOT EXISTS gst_amount DECIMAL(10,2),
ADD COLUMN IF NOT EXISTS gst_rate DECIMAL(5,4),
ADD COLUMN IF NOT EXISTS gst_hsn_code VARCHAR(10);

-- Add indexes for performance
CREATE INDEX IF NOT EXISTS idx_transactions_gst_hsn ON transactions(gst_hsn_code);

COMMIT;

-- Step 2: Backfill data (safe, idempotent)
-- V7__backfill_gst_data.sql
BEGIN;

UPDATE transactions 
SET gst_amount = amount * 0.18,
    gst_rate = 0.1800,
    gst_hsn_code = '997159'
WHERE transaction_date >= '2026-04-01'
  AND gst_amount IS NULL
  AND type IN ('NEFT', 'RTGS', 'IMPS');

-- Verify no rows were missed
DO $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM transactions 
        WHERE transaction_date >= '2026-04-01' 
          AND gst_amount IS NULL
    ) THEN
        RAISE EXCEPTION 'GST backfill incomplete!';
    END IF;
END $$;

COMMIT;

-- Step 3: Add NOT NULL constraint (after backfill)
-- V8__enforce_gst_constraints.sql
BEGIN;

ALTER TABLE transactions 
ALTER COLUMN gst_amount SET NOT NULL,
ALTER COLUMN gst_rate SET NOT NULL,
ALTER COLUMN gst_hsn_code SET NOT NULL;

COMMIT;
```

### E2E Example 2: Database Migration with Rollback

**Context:** Migration fails in production; need to rollback safely.

```bash
# Apply migration
$ flyway -url=jdbc:postgresql://prod-db:5432/banking migrate
# Processing migrations...
# [SUCCESS] V6__add_gst_columns.sql
# [SUCCESS] V7__backfill_gst_data.sql
# [ERROR] V8__enforce_gst_constraints.sql
# Migration failed: column gst_amount contains NULL values

# Check current state
$ flyway info
# | Version | Description | State  |
# | 6       | add gst     | Success|
# | 7       | backfill    | Success|
# | 8       | enforce     | Failed |

# Option 1: Fix forward (preferred)
# Create V9__fix_null_gst.sql
$ cat > src/main/resources/db/migration/V9__fix_null_gst.sql << 'EOF'
-- Fix NULL gst_amount values
UPDATE transactions 
SET gst_amount = 0.00, gst_rate = 0.0000, gst_hsn_code = 'UNKNOWN'
WHERE gst_amount IS NULL;

-- Now apply the constraint
ALTER TABLE transactions 
ALTER COLUMN gst_amount SET NOT NULL,
ALTER COLUMN gst_rate SET NOT NULL,
ALTER COLUMN gst_hsn_code SET NOT NULL;
EOF

$ flyway -url=jdbc:postgresql://prod-db:5432/banking migrate
# [SUCCESS] V9__fix_null_gst.sql ✅

# Option 2: Rollback (if fix-forward not possible)
$ flyway -url=jdbc:postgresql://prod-db:5432/banking undo
# [UNDO] V8__enforce_gst_constraints.sql (if supported)
# Note: Flyway Community doesn't support undo; use manual rollback
```

### E2E Example 3: Schema Validation in CI

**Context:** Catch schema issues before production.

```yaml
# Schema validation pipeline
schema-validation:
  stage: test
  script:
    # 1. Check migration syntax
    - flyway -url=jdbc:postgresql://test-db:5432/banking validate
    # Successfully applied 7 migrations to schema "public"
    
    # 2. Check for breaking changes
    - python scripts/check_breaking_changes.py
    # ✅ No DROP TABLE found
    # ✅ No DROP COLUMN found (without prior ADD)
    # ✅ No NOT NULL added without DEFAULT
    
    # 3. Check data integrity
    - python scripts/verify_data_integrity.py
    # ✅ All foreign keys valid
    # ✅ No orphaned records
    # ✅ All constraints satisfied
    
    # 4. Check performance impact
    - python scripts/check_query_performance.py
    # ✅ No full table scans on large tables
    # ✅ New indexes created for frequent queries
    
    # 5. Dry run on production schema copy
    - flyway -url=jdbc:postgresql://test-db:5432/banking_copy migrate
    # ✅ All migrations apply cleanly
    # ✅ Rollback test: clean revert possible
```

---

## 📋 Interview Questions

### Q1: What is the expand-contract pattern and why is it essential?
**Answer:** **Expand** = add new columns/tables (backward compatible). **Contract** = remove old columns (after code is updated). Essential because: (1) Old code versions still work during rollout. (2) Rollback doesn't lose data. (3) Zero downtime deployment possible. Example: Add `gst_amount` column first, deploy code that uses it, then remove old tax calculation column later.

### Q2: How do you handle database migrations in a blue-green deployment?
**Answer:** (1) **Expand phase** — apply migration to both Blue and Green databases. (2) **Deploy new code** to Green. (3) **Switch traffic** to Green. (4) **Contract phase** — only after Green is stable, remove old columns. Critical: migrations must be **idempotent** (safe to run multiple times) and **backward compatible** (old code works with new schema).

### Q3: What is the difference between Flyway and Liquibase?
**Answer:** **Flyway** uses plain SQL migrations, simpler to learn, better for SQL-heavy teams. **Liquibase** supports XML, YAML, and SQL, has built-in rollback support, better for complex changes. Banks often choose Liquibase because: (1) Rollback support is critical. (2) XML/YAML provides structured change tracking. (3) Better audit trail for compliance.

### Q4: How do you handle zero-downtime schema changes?
**Answer:** (1) **Never lock tables** during migration. (2) **Additive changes only** — ADD COLUMN, CREATE INDEX (CONCURRENTLY). (3) **Backfill in batches** — update 1000 rows at a time, not millions. (4) **Deploy code first** — code handles both old and new schema. (5) **Remove old schema later** — after all app instances updated. Tools: `pg_repack` for index rebuilds, `pt-online-schema-change` for MySQL.

### Q5: How do you validate database migrations before production?
**Answer:** (1) **Syntax validation** — `flyway validate`. (2) **Schema diff** — compare expected vs actual schema. (3) **Breaking change detection** — scan for DROP TABLE, DROP COLUMN. (4) **Data integrity checks** — foreign keys, constraints. (5) **Performance testing** — explain queries, check indexes. (6) **Dry run on copy** — apply to production schema copy. (7) **Load testing** — simulate production traffic on migrated schema.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Expand-Contract | Add first, remove later |
| Idempotent | Safe to run multiple times |
| Flyway | SQL-based, simple |
| Liquibase | XML/YAML, rollback support |
| Zero-Downtime | Never lock tables |
| Banking Relevance | Data integrity, compliance, audit trails |

**Next:** [21-Disaster-Recovery.md](./21-Disaster-Recovery.md) — Learn DR strategies for banking.
