# Step 01: Execution Report Templates

Read this reference when generating Phase 3 execution reports or output templates.

## Phase 3: Job Details (JSON Examples)

### jobs_retrieve Expected Output

```json
{
  "id": 1235,
  "name": "fix-linux-disk-cleanup",
  "status": "successful",
  "started": "2026-02-24T15:35:02Z",
  "finished": "2026-02-24T15:40:25Z",
  "elapsed": 323.45,
  "job_template": 10,
  "inventory": 1,
  "limit": "prod-web-01,prod-web-02,prod-web-03",
  "playbook": "playbooks/remediation/fix-disk-usage.yml"
}
```

### jobs_job_host_summaries_list Expected Output

```json
{
  "results": [
    {
      "host_name": "prod-web-01",
      "ok": 8,
      "changed": 3,
      "failed": 0,
      "unreachable": 0
    },
    {
      "host_name": "prod-web-02",
      "ok": 8,
      "changed": 3,
      "failed": 0,
      "unreachable": 0
    },
    {
      "host_name": "prod-web-03",
      "ok": 5,
      "changed": 0,
      "failed": 1,
      "unreachable": 0
    }
  ]
}
```

## Comprehensive Report Template

```markdown
# Playbook Execution Report

## Job Summary
**Job ID**: 1235
**Status**: ✅ Successful
**Duration**: 5m 23s
**Started**: 2026-02-24 15:35:02 UTC
**Completed**: 2026-02-24 15:40:25 UTC
**Job Template**: fix-linux-disk-cleanup
**Playbook**: playbooks/remediation/fix-disk-usage.yml
**Issue**: INC-20260224-001 — High disk usage on web tier
**AAP URL**: [View in AAP](https://aap.example.com/#/jobs/playbook/1235)

## Per-Host Results
| Host | OK | Changed | Failed | Unreachable | Status |
|------|-----|---------|--------|-------------|--------|
| prod-web-01 | 8 | 3 | 0 | 0 | ✅ Success |
| prod-web-02 | 8 | 3 | 0 | 0 | ✅ Success |
| prod-web-03 | 8 | 3 | 0 | 0 | ✅ Success |

**Summary**: 3 of 3 hosts successfully remediated

## Task Timeline
1. ✅ Gather Facts (2s)
2. ✅ Pre-flight checks (1s)
3. ✅ Backup configuration (3s)
4. ✅ Apply remediation action (45s)
   - prod-web-01: Cleared 4.2 GB from /var/log
   - prod-web-02: Cleared 3.8 GB from /var/log
   - prod-web-03: Cleared 5.1 GB from /var/log
5. ✅ Restart affected service (15s)
6. ✅ Verify service status (2s)
7. ✅ Update audit log (1s)

## Full Console Output
<details>
<summary>Click to expand (187 lines)</summary>

[Full stdout from jobs_stdout_retrieve]

</details>

## Job Log Issue Validation (Step 3.3)
✓ Job log confirms issue INC-20260224-001 was addressed

*(Or: ⚠️ Job log did not show clear evidence of issue handling—verify manually or use remediation-verifier)*

## Next Steps
1. ✅ All systems successfully remediated
2. ☐ Verify remediation with remediation-verifier skill
3. ☐ Update incident tracking system
4. ☐ Schedule follow-up health check in 24-48 hours

---

**Recommendation**: Run remediation-verifier skill to confirm the issue is fully resolved.
```

## Output Templates

### Success Template

```markdown
✅ Playbook Execution Successful

Job ID: 1235
Duration: 5m 23s
Systems Remediated: 3 of 3

View full report above for details.

Next Steps:
- Run remediation-verifier skill to confirm issue resolution
- Update incident tracking system
- Monitor systems for 24-48 hours

AAP URL: https://aap.example.com/#/jobs/playbook/1235
```

### Partial Success Template

```markdown
⚠️ Playbook Execution Completed with Failures

Job ID: 1235
Duration: 2m 45s
Systems Remediated: 2 of 3
Failed Systems: prod-web-03

See error details above for troubleshooting steps.

Options:
- Relaunch for failed hosts
- Manual remediation
- Skip failed hosts

AAP URL: https://aap.example.com/#/jobs/playbook/1235
```

### Failure Template

```markdown
❌ Playbook Execution Failed

Job ID: 1235
Duration: 1m 15s
Systems Remediated: 0 of 3

Critical errors prevented execution.
See error details above for troubleshooting.

AAP URL: https://aap.example.com/#/jobs/playbook/1235
```
```
