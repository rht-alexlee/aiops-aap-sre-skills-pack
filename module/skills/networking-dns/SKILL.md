---
description: DNS troubleshooting playbooks for the `sre-networking` sub-agent. Diagnoses name resolution failures (NXDOMAIN, SERVFAIL, slow resolution, split-horizon) from a Linux host or CoreDNS pod. Read-only.
---

# DNS Troubleshooting

## Workflow

1. **Test resolution** from the affected host (`dig`, `nslookup`).
2. **Check resolver config**: `/etc/resolv.conf`, `systemd-resolved`.
3. **Trace the query path** through each nameserver.
4. **Verify zone data**: SOA, NS, TTLs, DNSSEC.
5. **Check split-horizon** if internal vs external answers differ.

## Diagnostic Commands (reference)

```bash
dig <domain> A +short
dig <domain> @<nameserver> A +trace
cat /etc/resolv.conf
resolvectl status
dig -x <ip>
# In Kubernetes:
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
```

## Playbooks

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/dns_resolution_check.yml` | host can't resolve a name, or resolves to the wrong IP | `diag-net-dns-resolution` |
| `playbooks/dns_resolver_config.yml` | verify resolver config + systemd-resolved status | `diag-net-dns-config` |
| `playbooks/dns_zone_query.yml` | suspect zone data is wrong (SOA serial, NS, DNSSEC) | `diag-net-dns-zone` |

## Output

Save findings to `investigations/<incident-slug>/net-diag.md` (combined with
L2/L3 findings if multi-layer).
