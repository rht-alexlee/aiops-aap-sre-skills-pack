---
description: Layer-3 network diagnostic playbooks for the `sre-networking` sub-agent. Diagnoses IP routing, reachability, firewall rules, MTU/path issues, and dynamic-routing neighbour state. Read-only.
---

# Layer 3 Networking

## Workflow

1. **IP config**: addresses, subnets, default gateway.
2. **Reachability**: ping, traceroute — isolate where traffic drops.
3. **Routing table**: expected routes, correct next hops.
4. **Firewall**: iptables/nftables, security groups.
5. **Routing protocols**: OSPF/BGP neighbour status (if applicable).

## Diagnostic Commands (reference)

```bash
ip addr show
ip route show
ip route get <destination>
traceroute -n <destination>
iptables -L -n -v
nft list ruleset
vtysh -c "show ip ospf neighbor"
vtysh -c "show bgp summary"

# MTU
ping -M do -s 1472 <destination>
tracepath <destination>
```

## Playbooks

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/l3_reachability.yml` | unreachable destination, asymmetric routing, slow path | `diag-net-l3-reachability` |
| `playbooks/l3_firewall_audit.yml` | suspected firewall drop on a host | `diag-net-l3-firewall` |
| `playbooks/l3_mtu_probe.yml` | MTU blackhole symptoms (small packets work, large hang) | `diag-net-l3-mtu` |
| `playbooks/l3_routing_protocols.yml` | BGP/OSPF flap on a routing host (FRR/quagga) | `diag-net-l3-protocols` |

## Output

Save findings to `investigations/<incident-slug>/net-diag.md`.
