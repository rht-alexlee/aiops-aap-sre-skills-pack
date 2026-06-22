---
description: Shared log analysis workflow for every SRE sub-agent. Parses stdout returned by AAP playbooks (journalctl, container logs, Windows Event Log, syslog) — time-bounds the window, filters by severity, correlates across sources, extracts patterns.
---

# Log Analysis (shared)

Used by every SRE sub-agent on the stdout returned by `playbook-executor`.

## Workflow

1. **Identify log sources** in the stdout: journalctl blocks, container logs,
   Windows Event Log dumps, syslog, application logs.
2. **Time-bound the search**: focus on `incident_window_start − 15min` to
   `incident_window_end + 5min`.
3. **Filter for severity**: errors and warnings first, then info for context.
4. **Correlate across sources**: match timestamps between system and
   application logs.
5. **Extract patterns**: repeated errors, rate changes, new error types.

## Techniques

```bash
# Time-bounded journalctl
journalctl --since "2026-06-22 14:00" --until "2026-06-22 15:00" -p err

# Frequency analysis
journalctl -u <unit> --since "1 hour ago" --no-pager \
  | grep -i error | sort | uniq -c | sort -rn | head -20

# Container logs with timestamps
kubectl logs <pod> -n <ns> --timestamps --since=1h \
  | grep -iE "error|warn|fatal"

# Windows Event Log (from an ansible.windows.win_eventlog_info dump)
# Look for: EventID, Source, TimeGenerated, Message
```

## Output

When invoked by a sub-agent, return a section to be embedded in the
`<domain>-diag.md` report:

- **Time window**: exact range analyzed
- **Log sources**: which logs were checked
- **Key findings**: error messages with timestamps and frequency
- **Correlation**: how findings relate to the reported symptom
