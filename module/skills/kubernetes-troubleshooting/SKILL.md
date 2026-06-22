---
description: Kubernetes / OpenShift diagnostic playbooks for the `sre-kubernetes` sub-agent. Runs `kubernetes.core` modules from a bastion host to inspect cluster health, pods, nodes, and scheduling. All playbooks are read-only.
---

# Kubernetes Troubleshooting

Diagnostic playbooks for `sre-kubernetes`. Playbooks target the `k8s-bastion`
inventory (the bastion holds the kubeconfig); resource targets go in
`extra_vars`.

## Workflow

1. Identify scope: pod / deployment / node / cluster-wide.
2. Pick the playbook that matches.
3. Hand off to `playbook-executor` with `limit: k8s-bastion-1` (or your
   bastion host name) and the resource details in `extra_vars`.
4. Parse the structured output for events, container status, and node
   conditions.

## Diagnostic Commands (reference)

```bash
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -20
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous --tail=50
kubectl describe node <node> | grep -A5 Conditions
kubectl top pods -n <ns> --sort-by=memory
```

## Common patterns

- **CrashLoopBackOff** → check previous logs + look for OOMKilled
- **Pending** → events for scheduling failures, node resources, taints
- **ImagePullBackOff** → image name, registry auth, network
- **Evicted** → node memory/disk pressure

## Playbooks

| Playbook | Use when | Expected AAP template |
|----------|----------|------------------------|
| `playbooks/k8s_cluster_health.yml` | unknown cluster issue, broad triage | `diag-k8s-cluster-health` |
| `playbooks/k8s_pod_diagnose.yml` | one or more pods failing in a namespace | `diag-k8s-pod` |
| `playbooks/k8s_node_pressure.yml` | NodeNotReady, pressure conditions, evictions | `diag-k8s-node` |

## Output

Save findings to `investigations/<incident-slug>/k8s-diag.md`.
