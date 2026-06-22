---
description: Layer-2 network diagnostic playbooks for the `sre-networking` sub-agent. Diagnoses link state, VLAN config, ARP/NDP tables, and bridge topology from a Linux host. Read-only.
---

# Layer 2 Networking

## Workflow

1. **Link state**: interface up/down, speed/duplex.
2. **VLAN config**: tagged/untagged ports, trunk setup.
3. **ARP/NDP tables**: MAC resolution sanity.
4. **Spanning tree / bridge**: blocked ports, topology changes.
5. **MAC anomalies**: floods, loops, duplicates.

## Diagnostic Commands (reference)

```bash
ip link show
ethtool <iface>
ip -d link show | grep vlan
bridge vlan show
ip neigh show
bridge fdb show
```

## Playbooks

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/l2_link_check.yml` | suspected link / NIC issue | `diag-net-l2-link` |
| `playbooks/l2_arp_audit.yml` | unreachable LAN peer, duplicate IP, stuck ARP | `diag-net-l2-arp` |
| `playbooks/l2_bridge_vlan.yml` | bridge / VLAN / trunk misconfig | `diag-net-l2-bridge` |

## Output

Save findings to `investigations/<incident-slug>/net-diag.md`.
