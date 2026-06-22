---
description: Windows Server diagnostic playbooks for the `sre-windows` sub-agent. Covers host health, services, Event Log, disk, and performance using the `ansible.windows` collection. All playbooks are read-only.
---

# Windows Diagnostics

Diagnostic playbooks for the `sre-windows` sub-agent. The sub-agent picks the
playbook and invokes `playbook-executor` to run it in AAP.

## Workflow

1. Identify the Windows host(s) and the symptom.
2. Pick the matching playbook below.
3. Hand off to `playbook-executor`.
4. Parse Event Log IDs and Service status from the returned stdout.

## Diagnostic Commands (reference)

```powershell
Get-Service | Where-Object Status -ne 'Running'
Get-EventLog -LogName System -EntryType Error -Newest 50
Get-PSDrive -PSProvider FileSystem
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 1 -MaxSamples 5
```

## Playbooks

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/windows_health_check.yml` | first-pass Windows host triage | `diag-windows-health-check` |
| `playbooks/windows_service_diagnose.yml` | a named Windows service is failing or stuck | `diag-windows-service` |
| `playbooks/windows_eventlog_collect.yml` | Event Log spike or alert references a specific channel | `diag-windows-eventlog` |
| `playbooks/windows_disk_diagnose.yml` | drive full / low free space on a Windows host | `diag-windows-disk` |

## Output

Save findings to `investigations/<incident-slug>/windows-diag.md` with
symptom, findings, root cause, and remediation (with rollback).
