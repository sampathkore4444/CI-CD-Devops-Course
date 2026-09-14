# 10. Cron Job Not Executing as Expected

## Scenario
It's 10:00 AM on a Monday. The disaster recovery team runs a quarterly DR test and discovers that the nightly PostgreSQL backup hasn't been running for the past 3 days. The backup cron job (`/etc/cron.d/pgbackup`) is supposed to run `pg_dump` at 2:00 AM every night and upload the result to S3. The last successful backup in S3 is from Thursday night. Friday, Saturday, and Sunday backups are missing. The production database has been accumulating WAL files because the backup-based retention cleanup wasn't running. You need to investigate why the cron job stopped, restore the backup schedule, and catch up on the missing backups.

## Interviewer Question
"A critical cron job that runs database backups every night hasn't executed for 3 days. No one noticed until a disaster recovery test revealed missing backups. How do you investigate why it failed and restore the backup schedule?"

## What I Should Think About
- **Cron service status**: Is crond running?
- **Cron job syntax**: Is the cron entry syntactically correct?
- **Cron permissions**: Are `/etc/cron.allow` and `/etc/cron.deny` configured?
- **User context**: Does the cron job run as the correct user?
- **Environment**: Does the cron environment have the right PATH, HOME, etc.?
- **Mail/output**: Did the cron job fail silently? Where does output go?
- **Script issues**: Does the backup script have bugs?
- **System changes**: Was there a system update or config change?

## Ideal Answer

**Phase 1: Verify Cron Service is Running**

```bash
# Check if crond is running
systemctl status crond
ps aux | grep cron

# Check cron logs
grep CRON /var/log/cron | tail -30
grep CRON /var/log/syslog | tail -30  # Debian/Ubuntu

# Check the cron job entry
cat /etc/cron.d/pgbackup
crontab -l -u postgres
```

The output shows crond is running, but the cron logs show:
```
Oct 11 02:00:01 prod-db-01 CRON[12345]: (postgres) CMD (/opt/scripts/pgbackup.sh)
Oct 11 02:00:15 prod-db-01 CRON[12345]: (postgres) ERROR: pg_dump: connection to server failed
```

Wait — the cron job IS running, but the script is failing. Let me check further.

**Phase 2: Investigate the Script Failure**

```bash
# Check the backup script
cat /opt/scripts/pgbackup.sh

# Check if the script can run manually
su - postgres -c "/opt/scripts/pgbackup.sh" 2>&1

# Check PostgreSQL status
systemctl status postgresql

# Check PostgreSQL authentication
sudo -u postgres psql -c "SELECT version();"

# Check S3 connectivity
su - postgres -c "aws s3 ls s3://prod-backups/" 2>&1

# Check cron environment
su - postgres -c "env" > /tmp/postgres_env.txt
# Compare with cron's environment
```

The investigation reveals: On Friday, the PostgreSQL server's `pg_hba.conf` was updated during a security hardening exercise, changing the authentication method from `md5` to `scram-sha-256`. The backup script uses `pg_dump` with a password file (`~/.pgpass`) that still has the old `md5` format. The authentication fails silently because the cron job's output goes to a mailbox that nobody checks.

**Phase 3: Fix the Immediate Issue**

```bash
# Update the pgpass file to work with scram-sha-256
# Or use a different authentication method

# Option A: Update pgpass (if the server still accepts md5 for local connections)
# Check pg_hba.conf
cat /var/lib/pgsql/data/pg_hba.conf | grep -v "^#"

# Option B: Use a .pgpass file with the correct format
echo "localhost:5432:*:postgres:password" > /var/lib/pgsql/.pgpass
chmod 600 /var/lib/pgsql/.pgpass

# Option C: Use environment variable for password
# Update the script to use PGPASSWORD
export PGPASSWORD=$(cat /var/lib/pgsql/.password_file)

# Option D: Use peer authentication for local connections
# Update pg_hba.conf to allow local peer auth for postgres user
```

**Phase 4: Run the Missing Backups**

```bash
# Run the backup manually for the missing days
su - postgres -c "/opt/scripts/pgbackup.sh --full" 2>&1

# Or run pg_dump directly
pg_dump -U postgres -Fc -f /tmp/backup_$(date +%Y%m%d).dump production_db

# Upload to S3
aws s3 cp /tmp/backup_*.dump s3://prod-backups/daily/

# Clean up WAL files
# Check WAL retention
du -sh /var/lib/pgsql/data/pg_wal/
```

**Phase 5: Prevent Recurrence**

```bash
# Fix cron output to go to a monitored email/Slack
# In /etc/cron.d/pgbackup:
0 2 * * * postgres /opt/scripts/pgbackup.sh 2>&1 | logger -t pgbackup

# Add a monitoring script
# /opt/scripts/check_backup.sh
#!/bin/bash
LAST_BACKUP=$(aws s3 ls s3://prod-backups/daily/ --recursive | sort | tail -1 | awk '{print $4}')
LAST_DATE=$(echo $LAST_BACKUP | grep -oP '\d{8}')
TODAY=$(date -d "yesterday" +%Y%m%d)
if [ "$LAST_DATE" != "$TODAY" ]; then
    echo "CRITICAL: Backup from yesterday is missing" | mail -s "Backup Alert" ops@example.com
fi

# Add to cron for daily verification
0 8 * * * postgres /opt/scripts/check_backup.sh
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Cron Job Execution Flow                  │
│                                                          │
│  crond service (PID 1000)                                │
│       │                                                  │
│       ├── Reads /etc/cron.d/pgbackup                     │
│       │   Schedule: 0 2 * * * postgres                   │
│       │                                                  │
│       ├── Executes: /opt/scripts/pgbackup.sh             │
│       │   As user: postgres                              │
│       │   Environment: minimal (PATH=/usr/bin)           │
│       │                                                  │
│       ├── Script calls: pg_dump -U postgres production_db│
│       │   Uses: ~/.pgpass for authentication             │
│       │   Problem: pg_hba.conf changed to scram-sha-256 │
│       │   pgpass still configured for md5                │
│       │                                                  │
│       ├── pg_dump fails: authentication error            │
│       │   Error output goes to: postgres's mailbox       │
│       │   Mailbox: /var/mail/postgres (nobody checks)    │
│       │                                                  │
│       └── Result: Silent failure × 3 days               │
│                                                          │
│  Missing backups: Oct 11, 12, 13                         │
│  WAL files accumulating: 15GB                            │
│  Retention cleanup not running                           │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check cron service**: Run `systemctl status crond` to confirm the cron daemon is running.
2. **Check cron logs**: Run `grep CRON /var/log/cron | tail -50` to see if the job was scheduled and attempted.
3. **Check the cron entry**: Run `cat /etc/cron.d/pgbackup` to verify the syntax and user context.
4. **Check the backup script**: Run `cat /opt/scripts/pgbackup.sh` to understand what it does and look for errors.
5. **Run the script manually**: Run `su - postgres -c "/opt/scripts/pgbackup.sh"` to reproduce the failure.
6. **Check PostgreSQL authentication**: Check `pg_hba.conf` for recent changes and test authentication.
7. **Check cron environment**: Cron runs with a minimal environment. Compare `env` output between manual and cron execution.
8. **Check for cron output**: Check `/var/mail/postgres` or `/var/spool/mail/postgres` for error output.

## Commands

```bash
# 1. Check cron service
systemctl status crond
systemctl is-active crond

# 2. Check cron job entries
cat /etc/cron.d/pgbackup
crontab -l -u postgres
ls -la /etc/cron.d/
ls -la /etc/cron.daily/

# 3. Check cron logs
grep CRON /var/log/cron | grep -i postgres | tail -30
grep CRON /var/log/syslog | tail -30  # Debian/Ubuntu

# 4. Check cron permissions
cat /etc/cron.allow 2>/dev/null
cat /etc/cron.deny 2>/dev/null

# 5. Run the script manually
su - postgres -c "/opt/scripts/pgbackup.sh" 2>&1

# 6. Check PostgreSQL authentication
cat /var/lib/pgsql/data/pg_hba.conf | grep -v "^#" | grep -v "^$"

# 7. Test pg_dump directly
su - postgres -c "pg_dump -U postgres -Fc production_db > /dev/null" 2>&1

# 8. Check cron environment
su - postgres -c "env" 2>&1 > /tmp/user_env.txt
# In a cron script, capture the environment:
# #!/bin/bash
# env > /tmp/cron_env.txt
# Then compare the two

# 9. Check for cron output/mail
cat /var/mail/postgres 2>/dev/null | tail -50
ls -la /var/spool/mail/postgres 2>/dev/null

# 10. Check S3 backup status
aws s3 ls s3://prod-backups/daily/ --recursive | sort -k2 | tail -10

# 11. Verify backup recency
LATEST=$(aws s3 ls s3://prod-backups/daily/ --recursive | sort | tail -1 | awk '{print $3}')
echo "Latest backup: $LATEST"

# 12. Fix cron to log to syslog
# Instead of mailing output, pipe to logger
# In /etc/cron.d/pgbackup:
# 0 2 * * * postgres /opt/scripts/pgbackup.sh 2>&1 | logger -t pgbackup

# 13. Test cron syntax
crontab -T /etc/cron.d/pgbackup 2>&1  # Doesn't exist on all systems
# Or just verify the format manually
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| PostgreSQL auth method changed | `pg_hba.conf` shows `scram-sha-256` | Update pgpass file or change auth method for local connections |
| Cron syntax error | `crontab -l` shows syntax error | Fix the cron entry syntax |
| Cron service not running | `systemctl status crond` shows inactive | `systemctl start crond && systemctl enable crond` |
| User permission denied | Cron logs show " Permission denied" | Check `/etc/cron.allow` and user permissions |
| Script has a bug | Manual execution fails | Debug and fix the script |
| Missing environment variables | Script works manually but not in cron | Add `PATH` and other env vars to the cron entry |
| Disk full preventing backup | Script can't write to backup directory | Free disk space, fix the disk issue |
| Network issue uploading to S3 | Script can't reach S3 | Fix network connectivity, check firewall |

## Immediate Mitigation

1. **Fix the authentication issue**: Update `pg_hba.conf` or the `.pgpass` file to allow the backup script to authenticate.
2. **Run the backup manually**: Execute the backup script now to catch up on the missing days.
3. **Set up proper logging**: Change cron output to go to syslog instead of the mailbox.
4. **Verify the backup**: Test that the backup can be restored successfully.
5. **Clean up WAL files**: If they're accumulating, run a checkpoint or adjust `wal_keep_size`.

## Permanent Fix

1. **Implement backup monitoring**: Create a script that checks backup freshness daily and alerts if missing.
2. **Fix cron output handling**: Pipe cron output to `logger` so it goes to syslog, which is monitored.
3. **Add backup verification**: Implement a daily restore test to verify backup integrity.
4. **Document the dependency**: Document that the backup depends on pg_hba.conf configuration.
5. **Implement backup alerts**: Set up alerts via PagerDuty/Slack when backups fail.

## Monitoring

- **Backup existence**: Check S3 for yesterday's backup every morning at 8 AM.
- **Backup size**: Alert if backup size deviates significantly from baseline.
- **Backup duration**: Alert if backup takes significantly longer than expected.
- **WAL accumulation**: Monitor WAL directory size and alert if growing unbounded.
- **Cron job execution**: Monitor syslog for cron-related messages.
- **Restore testing**: Automate weekly restore tests to verify backup integrity.

## Security

- **Backup encryption**: Ensure backups are encrypted at rest (S3 SSE) and in transit.
- **Credentials in scripts**: Never hardcode passwords in scripts. Use `.pgpass` or a secrets manager.
- **S3 bucket policies**: Ensure the backup S3 bucket has appropriate access controls.
- **Audit logging**: Log all backup operations for compliance and incident investigation.

## Production Considerations

- **RPO impact**: 3 days of missing backups means 3 days of data at risk. This is a serious RPO violation.
- **Backup window**: The backup job should complete well before business hours.
- **Storage costs**: S3 storage for PostgreSQL backups can be significant. Implement lifecycle policies.
- **Cross-region replication**: Consider replicating backups to a different region for disaster recovery.
- **Backup verification**: Regularly test that backups can actually be restored.

## Senior-Level Answer

"I'd check `systemctl status crond` and `grep CRON /var/log/cron` to determine if the job was running but failing, or not running at all. In this case, the job was running but failing silently due to a PostgreSQL authentication change. I'd fix the `.pgpass` file, run the missing backups manually, and set up monitoring that alerts when backups are missing. The permanent fix includes piping cron output to syslog instead of a mailbox, implementing a daily backup freshness check, and adding automated restore testing."

## Architect-Level Answer

"This incident reveals three systemic gaps: no backup monitoring, no change management for database authentication, and no backup verification process. First, we need automated backup monitoring that alerts within hours, not days, of a missed backup. Second, database configuration changes (especially pg_hba.conf) should go through a change management process that checks dependencies — in this case, the backup system. Third, we need regular restore testing as part of our DR process, not just during quarterly tests. I'd recommend implementing a backup orchestration system (like pgBackRest or Barman) that provides built-in monitoring, incremental backups, and point-in-time recovery. The entire backup pipeline should be codified as infrastructure-as-code so changes are reviewed and tested before deployment."

## Follow-Up Questions

1. "What's the difference between `pg_dump`, `pg_basebackup`, and WAL archiving? When would you use each?"
2. "How does PostgreSQL's `scram-sha-256` authentication differ from `md5`? What are the implications for automation?"
3. "Design a comprehensive backup strategy for a PostgreSQL database with a 1-hour RPO and 15-minute RTO."
4. "How would you implement automated backup verification and restore testing?"
5. "What is the difference between a cron job in `/etc/cron.d/` and one in a user's crontab? When would you use each?"
