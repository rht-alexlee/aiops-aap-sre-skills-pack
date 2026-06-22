# Step 02: Error Handling Guide

Read this reference when generating Phase 4 error reports or troubleshooting.

## Error Categories

**Parse error output** from `jobs_stdout_retrieve` for these common patterns:

1. **Connection Failures**: SSH timeout, host unreachable, authentication failed
2. **Permission Errors**: sudo required, insufficient privileges, SELinux denials
3. **Package Manager Issues**: repo unavailable, package not found, dependency conflicts
4. **Service Failures**: service not found, restart failed, timeout
5. **Disk Space**: insufficient space for operations
6. **General Failures**: playbook syntax errors, task failures, unexpected exceptions

## Error Report Template

```markdown
# Playbook Execution Failed

## Job Summary
**Job ID**: 1235
**Status**: ❌ Failed
**Duration**: 2m 45s
**Started**: 2026-02-24 15:35:02 UTC
**Failed At**: 2026-02-24 15:37:47 UTC
**Job Template**: adhoc-fix-job
**Issue**: INC-20260224-001 — High disk usage on web tier
**AAP URL**: [View in AAP](https://aap.example.com/#/jobs/playbook/1235)

## Per-Host Results
| Host | OK | Changed | Failed | Unreachable | Status |
|------|-----|---------|--------|-------------|--------|
| prod-web-01 | 8 | 3 | 0 | 0 | ✅ Success |
| prod-web-02 | 8 | 3 | 0 | 0 | ✅ Success |
| prod-web-03 | 5 | 0 | 1 | 0 | ❌ Failed |

**Summary**: 2 of 3 hosts succeeded, 1 failed

## Failed Tasks Details

### Host: prod-web-03

**Task**: Restart affected service
**Error**: "Failed to restart httpd.service: Unit httpd.service not found."

**Error Category**: Service Failure

**Root Cause**: The target service is not installed or not recognized by systemd.

**Troubleshooting Steps**:
1. Check if the service unit exists:
   ```bash
   ssh prod-web-03 'systemctl list-unit-files | grep <service>'
   ```
2. Verify the package providing the service is installed:
   ```bash
   ssh prod-web-03 'rpm -qa | grep <package>'
   ```
3. Check systemd service status:
   ```bash
   ssh prod-web-03 'systemctl status <service>'
   ```
4. Review system journal for related errors:
   ```bash
   ssh prod-web-03 'journalctl -xe --no-pager | tail -100'
   ```

**Recommended Action**:
- Verify the affected service is installed on prod-web-03
- Check whether a previous step (e.g. package install) silently failed
- Manually install/repair the service if needed
- Relaunch job for failed host only

## Console Output (Last 50 Lines)
<details>
<summary>Click to expand error context</summary>

[Relevant error output from jobs_stdout_retrieve]

</details>

## Relaunch Options

Would you like to:
1. **Relaunch for failed hosts only** - Run job again with hosts="failed"
2. **Fix issues manually and relaunch** - Resolve problems first, then relaunch
3. **View full job output** - See complete execution logs
4. **Abort** - Stop remediation workflow

Please choose an option (1-4):
```

## Relaunch Parameters

**MCP Tool**: `jobs_relaunch_retrieve` (from aap-mcp-job-management)

**Parameters**:
```json
{
  "id": "1235",
  "requestBody": {
    "hosts": "failed",
    "job_type": "run"
  }
}
```

This relaunches the job for only the failed hosts.

## Common Error Patterns and Mitigations

| Pattern | Likely Cause | First Mitigation |
|---------|--------------|------------------|
| `Permission denied (publickey)` | SSH key/credential issue | Verify the Machine credential attached to the template; confirm key on target |
| `sudo: a password is required` | Become password missing or wrong | Set become method/password on the credential; ensure NOPASSWD where appropriate |
| `Could not resolve host` | DNS / inventory error | Check inventory host entry; verify DNS or `/etc/hosts` |
| `No package <name> available` | Repo not enabled / wrong package name | Enable required repo; verify package name for the OS version |
| `Disk full` / `No space left on device` | Insufficient disk on target or controller | Free space or add a pre-check task; relaunch on failed hosts |
| `Timeout (12s) waiting for privilege escalation prompt` | Slow sudo or wrong become | Increase `timeout`; verify become method |
| `unreachable` | Host down or network issue | Validate connectivity from execution node; consider excluding host |
```
