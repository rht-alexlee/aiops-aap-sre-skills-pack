# AIOps SRE Module

This module provides an orchestrator and four specialist SRE sub-agents that
diagnose and remediate infrastructure incidents through Ansible Automation
Platform (AAP).

## When to use

- **Generic infrastructure incident**: invoke `sre-orchestrator` which
  classifies the incident and delegates to the right specialist.
- **Symptom is clearly scoped to one domain**: invoke the specialist directly:
  - `sre-linux` — RHEL hosts, systemd, performance, OOM, disk pressure
  - `sre-windows` — Windows services, event log, disk, performance
  - `sre-kubernetes` — pods, nodes, scheduling, image pulls, evictions
  - `sre-networking` — DNS resolution, L2 link/VLAN/ARP, L3 routing/firewall
- **Slash commands**:
  - `/diagnose <symptom>` — read-only investigation across the relevant domain
  - `/remediate <incident-id>` — propose + execute a remediation playbook
    (requires explicit user approval)

## Skills bundled

| Skill | Purpose |
|-------|---------|
| `playbook-executor` | Launches AAP job templates; **all** AAP execution flows through this skill |
| `log-analysis` | Shared log analysis workflow used by every specialist |
| `linux-diagnostics` | RHEL diagnostic playbooks |
| `windows-diagnostics` | Windows diagnostic playbooks |
| `kubernetes-troubleshooting` | K8s/OpenShift diagnostic playbooks |
| `networking-dns` | DNS resolution diagnostic playbooks |
| `networking-layer2` | L2 link/VLAN/ARP diagnostic playbooks |
| `networking-layer3` | L3 routing/firewall diagnostic playbooks |

## MCP servers required

See `mcps.json`. Both `aap-mcp-job-management` and
`aap-mcp-inventory-management` must be configured with `AAP_MCP_SERVER` and
`AAP_API_TOKEN` environment variables.

## Memory

Module-level facts (AAP base URL, inventory naming, change-management notes)
live in `memory/conventions.md`. Sub-agents load this on every invocation.
