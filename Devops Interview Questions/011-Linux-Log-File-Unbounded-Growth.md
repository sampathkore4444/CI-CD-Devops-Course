# 11. Log File Growing Uncontrollably Filling Disk

## Scenario
It's 4:00 PM on a Tuesday. You receive an alert: "Disk usage critical on prod-api-02: 92%." You investigate and find that the application log file `/var/log/myapp/application.log` has grown to 50GB in the last 6 hours. A developer debugging an issue in production earlier today accidentally enabled DEBUG log level in the application's logging configuration. The log file is growing at approximately 2GB per hour. The application cannot be restarted right now because it's handling peak traffic and a restart would cause a 30-second outage affecting 10,000 concurrent users. The disk will fill up completely within the next 45 minutes at the current growth rate. You need to handle this without restarting the application.

## Interviewer Question
"A debug log level was accidentally enabled in production. The log file has grown to 50GB and is still growing, threatening to fill the disk. The application cannot be restarted right now. How do you handle this?"

## What I Should Think About
- **Immediate relief**: Truncating the log file while the app writes to it
- **The app holds the file descriptor**: Truncating works; deleting doesn't
- **logrotate vs manual**: Can we use logrotate in the current situation?
- **Runtime config change**: Can the log level be changed without restart?
- **JMX/actuator endpoints**: Spring Boot can change log levels at runtime
- **Disk monitoring**: Alert thresholds should have caught this earlier
- **Communication**: Coordinate with the team about the fix timeline

## Ideal Answer

**Phase 1: Immediate Triage — Stop the Disk from Filling**

The most critical action is to free disk space RIGHT NOW without restarting the application.

```bash
# Truncate the log file (the app keeps writing, but the file shrinks)
truncate -s 0 /var/log/myapp/application.log

# Verify the file is truncated
ls -lh /var/log/myapp/application.log
# Should show 0 bytes

# The application continues writing, but the disk space is freed
```

**Why truncate instead of delete?** If you `rm` the file, the application still holds the file descriptor. The space won't be freed until the process closes the FD (which requires a restart). But `truncate` sets the file size to 0 while the FD remains valid, so the application keeps writing to the same file and the space is immediately freed.

**Phase 2: Stop the Log Level from Being Re-enabled**

```bash
# Check the application's logging configuration
cat /etc/myapp/logback-spring.xml
cat /etc/myapp/log4j2.xml

# If it's Spring Boot with actuator, change log level at runtime:
curl -X POST http://localhost:8080/actuator/loggers/com.myapp \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "INFO"}'

# If using JMX:
# Use jconsole or jmxterm to change the log level

# If using a file-based config that's being watched:
# Update the config file back to INFO level
sed -i 's/level="DEBUG"/level="INFO"/' /etc/myapp/logback-spring.xml
```

**Phase 3: Free Additional Space**

```bash
# Check what else is consuming space
du -sh /var/log/myapp/*
du -sh /var/log/*

# Clean up old rotated logs
find /var/log -name "*.gz" -mtime +7 -delete

# Check for other large files
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# Truncate other log files if needed
for f in /var/log/myapp/*.log; do
  size=$(stat -c%s "$f" 2>/dev/null)
  if [ "$size" -gt 104857600 ]; then  # > 100MB
    truncate -s 0 "$f"
    echo "Truncated: $f"
  fi
done
```

**Phase 4: Implement Log Rotation Going Forward**

```bash
# Create a logrotate config
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 100M
    create 0644 appuser appuser
}
EOF

# Test the configuration
logrotate -d /etc/logrotate.d/myapp

# Force a rotation now
logrotate -f /etc/logrotate.d/myapp
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  prod-api-02                              │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Application (Java/Spring Boot)                     │  │
│  │ Log level: DEBUG (accidentally enabled)            │  │
│  │ Writing to: /var/log/myapp/application.log         │  │
│  │ Rate: ~2GB/hour                                    │  │
│  │ File descriptor: OPEN (app holds FD 234)           │  │
│  └────────────────────┬───────────────────────────────┘  │
│                       │ writes                           │
│  ┌────────────────────▼───────────────────────────────┐  │
│  │ /var/log/myapp/application.log                     │  │
│  │ Size: 50GB → growing at 2GB/hour                   │  │
│  │ Disk: 92% → will hit 100% in ~45 minutes          │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Fix:                                                    │
│  1. truncate -s 0 /var/log/myapp/application.log         │
│  2. Change log level to INFO (via actuator/JMX)          │
│  3. Implement logrotate                                 │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Confirm disk usage**: Run `df -h` to see the exact disk usage and how much space is left.
2. **Identify the growing file**: Run `du -sh /var/log/myapp/*` to find the specific file consuming space.
3. **Check the growth rate**: Run `ls -lh /var/log/myapp/application.log` twice, 1 minute apart, to estimate growth rate.
4. **Check what changed**: Review recent deployment or configuration changes. Look for DEBUG level in config files.
5. **Check if runtime config change is possible**: Look for actuator endpoints, JMX, or file watchers.
6. **Check if truncating will work**: Confirm the application has the file open with `lsof | grep application.log`.
7. **Calculate time to disk full**: Divide remaining disk space by growth rate.
8. **Check for other log files**: Ensure other logs aren't also growing due to the verbose logging.

## Commands

```bash
# 1. Check disk usage
df -h

# 2. Find the largest log files
du -sh /var/log/myapp/* | sort -rh | head -10

# 3. Check real-time growth
watch -n 5 "ls -lh /var/log/myapp/application.log"

# 4. Verify the application has the file open
lsof | grep application.log

# 5. TRUNCATE THE FILE (critical action)
truncate -s 0 /var/log/myapp/application.log

# 6. Verify truncation
ls -lh /var/log/myapp/application.log
df -h  # Should show freed space

# 7. Check if Spring Boot actuator is available
curl -s http://localhost:8080/actuator/loggers | python -m json.tool | head -20

# 8. Change log level via actuator (Spring Boot)
curl -X POST http://localhost:8080/actuator/loggers/com.myapp \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "INFO"}'

# 9. Change log level via config file
grep -r "level" /etc/myapp/logback-spring.xml
sed -i 's/level="DEBUG"/level="INFO"/' /etc/myapp/logback-spring.xml

# 10. Clean up old log files
find /var/log -name "*.log.*" -mtime +7 -delete
find /var/log -name "*.gz" -mtime +30 -delete

# 11. Set up logrotate
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
    maxsize 100M
}
EOF

# 12. Test and force logrotate
logrotate -d /etc/logrotate.d/myapp  # Dry run
logrotate -f /etc/logrotate.d/myapp   # Force rotation

# 13. Monitor disk usage in real-time
watch -n 10 "df -h /var/log"
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| DEBUG log level enabled in production | Config file shows DEBUG | Change back to INFO, implement change control for log levels |
| No log rotation configured | No logrotate entry exists | Implement logrotate with maxsize and daily rotation |
| No disk usage alerts | Disk grew from 40% to 92% without alert | Set up alerts at 80% and 90% |
| No log file size limits | Application writes unbounded logs | Implement max file size in logging framework |
| Debugging in production | Developer changed log level | Implement change control for production config changes |

## Immediate Mitigation

1. **Truncate the log file**: `truncate -s 0 /var/log/myapp/application.log` — this immediately frees disk space.
2. **Change log level back to INFO**: Via actuator, JMX, or config file update.
3. **Clean up old logs**: Delete rotated/compressed logs older than 7 days.
4. **Monitor disk**: Watch `df -h` to confirm disk usage is decreasing.
5. **Communicate**: Inform the team that the log level was changed back and the disk issue is resolved.

## Permanent Fix

1. **Implement logrotate** with `maxsize` and `copytruncate` for all application logs.
2. **Implement change control** for production configuration changes (especially log levels).
3. **Add disk usage alerts** at 80% (warning) and 90% (critical).
4. **Use structured logging** with log levels controlled by environment variables, not config files.
5. **Implement log rate limiting** in the application to prevent log storms.
6. **Centralize logging** to prevent local disk from filling up.

## Monitoring

- **Log file size**: Monitor log file sizes and alert when they exceed expected baselines.
- **Log growth rate**: Alert when log growth rate exceeds normal thresholds.
- **Disk usage**: Alert at 80% and 90% for all mount points.
- **Log level changes**: Audit and alert on any log level changes in production.
- **Application errors**: Monitor for "No space left on device" errors.

## Security

- **Sensitive data in logs**: DEBUG logs may contain sensitive data (PII, credentials). Ensure log access is restricted.
- **Log injection**: Attackers may inject verbose logging to trigger disk fills.
- **Audit logging**: Maintain audit trail of who changed the log level and when.
- **Log retention**: Implement proper log retention policies for compliance.

## Production Considerations

- **Zero-downtime**: The truncation approach avoids any downtime, which is critical during peak traffic.
- **Log level change control**: Production log level changes should require approval.
- **Centralized logging**: Moving to centralized logging (ELK, Loki, CloudWatch) eliminates local disk issues.
- **Cost**: Storing 50GB of logs locally has cost implications. Centralized logging may be more cost-effective.
- **Operational runbook**: This scenario should be documented for the team.

## Senior-Level Answer

"I'd immediately truncate the log file with `truncate -s 0 /var/log/myapp/application.log` to free disk space without restarting the application — the app holds the file descriptor so it continues writing to the same file. Then I'd change the log level back to INFO, ideally via Spring Boot actuator endpoints without a restart. I'd implement logrotate with `maxsize` and `copytruncate`, and set up disk usage alerts. The permanent fix includes implementing change control for production log levels and centralizing logging to prevent local disk fill issues."

## Architect-Level Answer

"This incident reveals three architectural problems: no log level governance, no local disk guardrails, and monolithic logging architecture. First, log levels in production should be managed through a centralized configuration system with audit trails and approval workflows. Second, every application should have hard limits on log file sizes through the logging framework itself (max file size, max total size), not just OS-level logrotate. Third, and most importantly, we should move to centralized logging where application logs are shipped directly to a logging platform (ELK, Loki, CloudWatch) without accumulating on local disk. This eliminates the entire class of 'disk fill from logs' incidents. I'd also recommend implementing log sampling for DEBUG-level logs in production — instead of logging every debug event, sample at 1% to preserve some debugging capability without the disk impact."

## Follow-Up Questions

1. "What's the difference between `truncate` and `rm` for a file that's currently open by a process? Why does one free space and the other doesn't?"
2. "How does `copytruncate` in logrotate work? What are the risks of losing log data during rotation?"
3. "Explain Spring Boot's logging architecture. How do logback, Log4j2, and the actuator interact?"
4. "Design a logging architecture for a microservices system that prevents local disk fill while maintaining debuggability."
5. "How would you implement log sampling in a production application to reduce log volume while preserving the ability to debug issues?"
