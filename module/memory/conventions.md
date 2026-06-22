# SRE Module Conventions

Shared facts every sub-agent should assume unless told otherwise.

## AAP

- AAP Web UI base URL is read from `AAP_BASE_URL`; link every job report to
  `${AAP_BASE_URL}/#/jobs/playbook/<job-id>`.
- `playbook-executor` only runs pre-registered AAP job templates. There is no
  ad-hoc execution path — if no template matches, the sub-agent stops and
  asks an operator to register one.
- Diagnostic templates follow the naming pattern
  `diag-<domain>-<purpose>` (e.g. `diag-linux-health-check`).
- Remediation templates follow `fix-<domain>-<purpose>`
  (e.g. `fix-linux-disk-cleanup`).
- Every template MUST have credentials attached, an inventory selected,
  `become_enabled: true` for remediation, and ideally
  `ask_variables_on_launch: true` + `ask_limit_on_launch: true` so the
  executor can pass incident context and host targets at launch time.

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
