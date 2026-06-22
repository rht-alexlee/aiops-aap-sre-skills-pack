# SRE Module Conventions

Shared facts every sub-agent should assume unless told otherwise.

## AAP

- AAP Web UI base URL is read from `AAP_BASE_URL`; link every job report to
  `${AAP_BASE_URL}/#/jobs/playbook/<job-id>`.
- The catch-all template for newly generated remediation playbooks is
  `adhoc-fix-job`. It MUST have `ask_variables_on_launch=true` and
  `ask_limit_on_launch=true`.
- Pre-built diagnostic templates follow the naming pattern
  `diag-<domain>-<purpose>` (e.g. `diag-linux-health-check`).
- Pre-built remediation templates follow `fix-<domain>-<purpose>`.

## Inventories

- Linux inventories: `linux-prod`, `linux-staging`
- Windows inventories: `windows-prod`, `windows-staging`
- Kubernetes control hosts: `k8s-bastion` (the bastion runs `kubectl` against
  the cluster; diagnostic playbooks ssh to it, they do not `kubectl` from the
  AAP control plane)
- Network devices: `net-core`, `net-edge`

## Output directory

All investigation findings go to `investigations/<incident-slug>/`:
- `<domain>-diag.md` — diagnostic report
- `<domain>-remediation.md` — proposed fix + approval trail
- `aap-job-<id>.txt` — full job stdout (also retrievable from AAP)

## Change management

- Remediation in `*-prod` inventories requires explicit user "yes" + reason.
- Outside the change window (Mon–Fri 09:00–17:00 local), warn the user before
  running remediation against prod.
