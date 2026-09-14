# 6. Application Failing Due to File Permission Error

## Scenario
It's 9:00 AM on a Friday. A deployment was completed last night at 11 PM. This morning, the support team reports that the application is failing with "Permission denied" errors. The application is a Python Flask app running under the `appuser` user account via Gunicorn. The config file `/etc/myapp/config.yaml` was updated during the deployment by a CI/CD pipeline running as root. The application logs show: `[ERROR] Permission denied: /etc/myapp/config.yaml (errno 13)`. The deployment also updated the SSL certificate files in `/etc/myapp/certs/`. You need to fix the permissions without rolling back the deployment.

## Interviewer Question
"After a deployment, the application starts failing with 'Permission denied' errors. The application runs as a specific user but config files were updated by root. How do you diagnose and fix this without rolling back?"

## What I Should Think About
- **File ownership**: `chown` to change owner/group
- **File permissions**: `chmod` to set read/write/execute
- **Directory traversal**: Application needs execute permission on all parent directories
- **umask**: What umask was the CI/CD pipeline running with?
- **SELinux/AppArmor**: Could be mandatory access control, not just DAC
- **ACLs**: Could be POSIX ACLs overriding standard permissions
- **Temporary fix vs permanent fix**: How to fix now and prevent recurrence
- **Symlinks**: Are there symlinks involved that have their own permissions?

## Ideal Answer

**Phase 1: Diagnose the Issue**

```bash
# Check the application logs for the exact error
tail -50 /var/log/myapp/error.log

# Check the current permissions on the config file
ls -la /etc/myapp/config.yaml
stat /etc/myapp/config.yaml

# Check who the application runs as
ps aux | grep gunicorn | head -5

# Verify the user exists
id appuser

# Check directory permissions
ls -la /etc/myapp/
ls -la /etc/myapp/certs/

# Check if SELinux is blocking
getenforce 2>/dev/null
ausearch -m avc --recent 2>/dev/null | tail -20
```

The output shows:
```
-rw------- 1 root root 2048 Oct 13 23:15 /etc/myapp/config.yaml
```

The config file is owned by `root:root` with `600` permissions (read/write for owner only). The application runs as `appuser` and can't read it.

**Phase 2: Fix the Immediate Issue**

```bash
# Fix ownership
chown appuser:appuser /etc/myapp/config.yaml

# Or if it should be readable by a group
chown root:appgroup /etc/myapp/config.yaml
chmod 640 /etc/myapp/config.yaml

# Fix the certs directory too
chown -R appuser:appuser /etc/myapp/certs/
chmod 600 /etc/myapp/certs/*

# Check directory traversal permissions
chmod 755 /etc/myapp/
chmod 755 /etc/

# Verify the fix
ls -la /etc/myapp/config.yaml
su - appuser -c "cat /etc/myapp/config.yaml"
```

**Phase 3: Verify the Application Recovers**

```bash
# Restart the application (or just the worker)
systemctl restart myapp
# OR
kill -HUP $(pgrep -f gunicorn)  # Graceful reload

# Verify it's running
systemctl status myapp
curl -s http://localhost:8080/health

# Check logs for errors
tail -20 /var/log/myapp/error.log
```

**Phase 4: Fix the CI/CD Pipeline**

The root cause is that the CI/CD pipeline runs as root and sets root:root ownership. Fix the deployment scripts:

```yaml
# In the deployment playbook/script
- name: Deploy config file
  copy:
    src: config.yaml
    dest: /etc/myapp/config.yaml
    owner: appuser
    group: appuser
    mode: '0640'
  become: yes
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Deployment Flow                          │
│                                                          │
│  CI/CD Pipeline (runs as root)                           │
│       │                                                  │
│       ├──▶ Config File: root:root 600 ← WRONG           │
│       │    /etc/myapp/config.yaml                        │
│       │                                                  │
│       ├──▶ Cert Files: root:root 600 ← WRONG            │
│       │    /etc/myapp/certs/server.key                   │
│       │    /etc/myapp/certs/server.crt                   │
│       │                                                  │
│  Application (runs as appuser)                           │
│       │                                                  │
│       ├──▶ Tries to read config.yaml → DENIED (errno 13)│
│       ├──▶ Tries to read certs/ → DENIED                │
│       └──▶ Application fails to start                   │
│                                                          │
│  Fix:                                                    │
│       chown appuser:appuser /etc/myapp/config.yaml       │
│       chmod 640 /etc/myapp/config.yaml                   │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check application logs**: Run `tail -50 /var/log/myapp/error.log` to find the exact permission error and file path.
2. **Check file permissions**: Run `ls -la /etc/myapp/config.yaml` and `stat /etc/myapp/config.yaml` to see current ownership and mode.
3. **Check application user**: Run `ps aux | grep gunicorn` to confirm which user the application runs as.
4. **Check directory permissions**: Run `ls -la /etc/myapp/` — the application needs execute permission on all parent directories to traverse the path.
5. **Check for SELinux**: Run `getenforce` and `ausearch -m avc --recent` — SELinux can deny access even if DAC permissions are correct.
6. **Check for ACLs**: Run `getfacl /etc/myapp/config.yaml` — POSIX ACLs can override standard permissions.
7. **Test as the application user**: Run `su - appuser -c "cat /etc/myapp/config.yaml"` to reproduce the issue.
8. **Check the deployment logs**: Review the CI/CD pipeline execution to see what commands were run and what umask was in effect.

## Commands

```bash
# 1. Check current permissions
ls -la /etc/myapp/config.yaml
stat /etc/myapp/config.yaml

# 2. Check who the app runs as
ps aux | grep gunicorn
cat /etc/systemd/system/myapp.service | grep User

# 3. Check directory traversal permissions
namei -l /etc/myapp/config.yaml

# 4. Test access as the app user
su -s /bin/bash appuser -c "cat /etc/myapp/config.yaml" 2>&1

# 5. Check for SELinux denials
getenforce
ausearch -m avc --start recent 2>/dev/null | tail -20
ls -Z /etc/myapp/config.yaml

# 6. Check for POSIX ACLs
getfacl /etc/myapp/config.yaml

# 7. Fix ownership
chown appuser:appuser /etc/myapp/config.yaml

# 8. Fix permissions (readable by owner, group-readable)
chmod 640 /etc/myapp/config.yaml

# 9. Fix entire directory
chown -R appuser:appuser /etc/myapp/
chmod -R u=rwX,g=rX,o= /etc/myapp/

# 10. Fix certs specifically (should be more restrictive)
chown root:appuser /etc/myapp/certs/server.key
chmod 640 /etc/myapp/certs/server.key
chown root:appuser /etc/myapp/certs/server.crt
chmod 644 /etc/myapp/certs/server.crt

# 11. Fix directory traversal
chmod 755 /etc /etc/myapp

# 12. Verify the fix
su -s /bin/bash appuser -c "cat /etc/myapp/config.yaml" 2>&1
su -s /bin/bash appuser -c "ls /etc/myapp/certs/" 2>&1

# 13. Fix SELinux context if needed
chcon -t etc_t /etc/myapp/config.yaml
restorecon -Rv /etc/myapp/

# 14. Restart the application
systemctl restart myapp
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| CI/CD deploys files as root:root | `ls -la` shows root ownership | Fix deployment script to set correct owner with `chown` or use `become: appuser` |
| umask too restrictive in deployment | Files created with 600 or 700 | Set umask to 0022 in CI/CD pipeline |
| Directory not traversable | `namei -l` shows missing `x` permission on parent | `chmod 755` on parent directories |
| SELinux blocking access | `ausearch -m avc` shows denial | Set correct SELinux context with `chcon` or `semanage fcontext` |
| ACL overriding permissions | `getfacl` shows deny entries | Remove or modify ACL with `setfacl` |
| Symlink pointing to wrong location | `ls -la` shows symlink | Fix symlink target permissions |

## Immediate Mitigation

1. **Fix ownership**: `chown appuser:appuser /etc/myapp/config.yaml /etc/myapp/certs/*`
2. **Fix permissions**: `chmod 640 /etc/myapp/config.yaml` and `chmod 600 /etc/myapp/certs/*.key`
3. **Fix directory traversal**: `chmod 755 /etc/myapp/`
4. **Verify as application user**: `su -s /bin/bash appuser -c "cat /etc/myapp/config.yaml"`
5. **Restart the application**: `systemctl restart myapp` or `kill -HUP $(pgrep gunicorn)`
6. **Confirm recovery**: Check application health endpoint and logs for errors.

## Permanent Fix

1. **Fix the CI/CD pipeline** to set correct ownership and permissions after deployment:
```yaml
# Ansible example
- name: Set config file permissions
  file:
    path: /etc/myapp/config.yaml
    owner: appuser
    group: appuser
    mode: '0640'
```
2. **Use a deployment user**: Run the deployment as the application user where possible, or explicitly set ownership in the pipeline.
3. **Implement permission validation**: Add a post-deployment check that verifies file permissions.
4. **Use configuration management**: Ansible/Chef/Puppet should manage file permissions as part of desired state.
5. **Document the expected permissions**: Create a manifest of file paths, owners, and permissions.

## Monitoring

- **Application health checks**: Alert immediately when the application fails to start.
- **File permission monitoring**: Use tools like AIDE or custom scripts to detect permission changes.
- **Deployment verification**: Add post-deployment smoke tests that verify application functionality.
- **Log monitoring**: Alert on "Permission denied" errors in application logs.
- **Change detection**: Monitor `/etc/myapp/` for unexpected changes using `inotifywait` or `auditd`.

## Security

- **Principle of least privilege**: Config files should be readable only by the application user and necessary groups.
- **Sensitive data in configs**: If the config contains credentials, ensure permissions are restrictive (600 or 640).
- **SELinux/AppArmor**: Consider using mandatory access control for defense in depth.
- **Audit file access**: Use `auditd` to log all access to sensitive configuration files.
- **Secrets management**: Move sensitive configuration to a secrets manager (Vault, AWS Secrets Manager) instead of file-based configs.

## Production Considerations

- **Rollback vs fix-forward**: In this case, fix-forward (correcting permissions) is faster than rollback.
- **Deployment window**: If this happens during business hours, communicate the issue and fix timeline.
- **Multiple servers**: If this deployment affected multiple servers, you'll need to fix permissions on all of them — use Ansible or similar.
- **Change management**: Document the permission change in your change management system.
- **Testing**: Add permission checks to your deployment pipeline to catch this before it reaches production.

## Senior-Level Answer

"I'd diagnose this by checking `ls -la` on the affected files and `ps aux | grep gunicorn` to confirm the application user. I'd use `namei -l` to verify directory traversal permissions and `getenforce` to rule out SELinux. The fix is `chown appuser:appuser` on the config files and `chmod 640` for appropriate access. I'd verify by testing with `su -s /bin/bash appuser -c 'cat /etc/myapp/config.yaml'`, then restart the application. The permanent fix is updating the CI/CD pipeline to set correct ownership and permissions in the deployment step, and adding a post-deployment permission validation check."

## Architect-Level Answer

"This incident reveals that file permission management is not part of our deployment contract. We need to define a permission manifest — a file that specifies every file path, owner, group, and mode that the application requires. This manifest should be validated after every deployment automatically. More broadly, I'd recommend moving sensitive configuration to a centralized configuration management system (Vault, Consul, or cloud-native parameter stores) that the application fetches at runtime, eliminating file-based configuration as a failure mode entirely. For the CI/CD pipeline, I'd implement a 'deployment contract' pattern where the pipeline explicitly declares what user context it operates under and what the resulting file state should be. This should be codified as a test in the pipeline that fails if permissions don't match the expected state."

## Follow-Up Questions

1. "What's the difference between DAC (Discretionary Access Control) and MAC (Mandatory Access Control) in Linux? How does SELinux differ from standard Unix permissions?"
2. "How do POSIX ACLs work? When would you use them instead of standard Unix permissions?"
3. "What is the `sticky bit`, `setuid`, and `setgid`? How do they affect file and directory permissions?"
4. "How would you implement a secrets management solution that eliminates the need for config files with sensitive data?"
5. "Design a deployment pipeline that automatically validates and enforces file permissions across a fleet of 200 servers."
