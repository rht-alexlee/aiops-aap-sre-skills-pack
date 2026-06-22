---
description: RHEL / generic Linux diagnostic playbooks for the `sre-linux` sub-agent. Covers host health, systemd services, disk/filesystem, and performance. All playbooks are read-only; remediation is generated separately and run via `adhoc-fix-job`.
---

# Linux Diagnostics

Bundled diagnostic playbooks for the `sre-linux` sub-agent. The sub-agent
picks the playbook that matches the symptom, then invokes the
`playbook-executor` skill to run it in AAP.

## Workflow

1. **Gather context**: what symptom triggered this? (alert, user report, monitor)
2. **Check system health**: load avg, memory pressure, disk, OOM kills
3. **Examine services**: systemd unit status, failed units, recent restarts
4. **Review logs**: journalctl in the incident window, dmesg for kernel issues
5. **Profile if needed**: CPU, memory, disk, network

## Diagnostic Commands (reference)

```bash
# Quick health
uptime && free -h && df -h && systemctl --failed

# Recent kernel
dmesg --since "1 hour ago" | tail -50

# Service investigation
systemctl status <unit> && journalctl -u <unit> --since "1 hour ago" --no-pager

# Performance
vmstat 1 5 && iostat -xz 1 5
```

## Playbooks

All playbooks live under `playbooks/`. Match the symptom to the playbook,
then invoke `playbook-executor`.

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/rhel_health_check.yml` | first-pass triage of an unknown host issue | `diag-linux-health-check` |
| `playbooks/rhel_service_diagnose.yml` | a named systemd unit is failing | `diag-linux-service` |
| `playbooks/rhel_disk_diagnose.yml` | `df` alarms, filesystem pressure, inode exhaustion | `diag-linux-disk` |
| `playbooks/rhel_perf_snapshot.yml` | high load avg, CPU/IO contention without obvious cause | `diag-linux-perf` |

If no AAP template matches the situation precisely, generate a fresh
playbook on disk and run it via `adhoc-fix-job` (Mode B in `playbook-executor`).

## Output

Save findings to `investigations/<incident-slug>/linux-diag.md`:

- **Symptom** — what was observed
- **Findings** — what diagnostics revealed (cite stdout excerpts + AAP job URL)
- **Root cause** — confirmed or suspected
- **Remediation** — exact commands / playbook to fix, with rollback
