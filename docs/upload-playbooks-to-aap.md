# Uploading the SRE playbooks to AAP

The `playbook-executor` skill only runs **pre-registered AAP job templates**.
You don't upload playbook *files* to AAP — instead, AAP **pulls** the
playbooks from a Git repository via a **Project**, and each playbook is
exposed to the agents as a **Job Template** with a name and description that
the executor can match against by keyword.

This guide walks through that one-time setup three ways:

1. **Automated bootstrap** with `awx.awx` (recommended).
2. **Manual via the AAP Web UI.**
3. **Manual via the `awx` CLI.**

You only need to do one of them.

---

## Prerequisites

- AAP 2.4+ (Controller). The `controller_*` API parameters and the `awx.awx`
  collection both work against the upstream AWX project too.
- An AAP **Organization** (the examples below use `Default`).
- An **Execution Environment** that includes the `ansible.windows` and
  `kubernetes.core` collections. The default Red Hat AAP execution
  environment satisfies this; if you build your own, add:
  ```yaml
  collections:
    - ansible.windows
    - kubernetes.core
  ```
- **Credentials** already created in AAP:
  - `linux-ssh` — Machine credential with SSH key for RHEL hosts and the
    Kubernetes bastion.
  - `windows-winrm` — Machine credential of type "Machine" using WinRM
    (set `ansible_connection=winrm` plus auth method in the credential's
    extra vars or via a custom credential type).
- An **Inventory** (or three) for the targets. The bootstrap will create empty
  ones for you, but you still need to populate them with hosts.

---

## Option 1 — Automated bootstrap (recommended)

This repo ships a one-shot playbook that creates the Project, ensures the
inventories exist, and registers every diagnostic job template in one run.

```bash
# 1. Install the AAP/AWX automation collection
ansible-galaxy collection install awx.awx

# 2. Point the awx.awx modules at your controller
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='change-me'
# (or, instead of user/pass:)
# export CONTROLLER_OAUTH_TOKEN=...

# 3. Run the bootstrap
ansible-playbook aap-bootstrap/register_in_aap.yml \
  -e organization=Default \
  -e project_scm_url=https://github.com/rht-alexlee/aiops-aap-sre-skills-pack.git \
  -e project_scm_branch=main \
  -e ssh_credential_name=linux-ssh \
  -e winrm_credential_name=windows-winrm
```

What it does:

| Step | AAP object | Detail |
|------|------------|--------|
| 1 | **Inventories** | Creates `linux-prod`, `windows-prod`, `k8s-bastion` if they don't exist (you still populate hosts). |
| 2 | **Project** `aiops-sre-skills` | Source: this Git repo, branch `main`, `scm_update_on_launch: true` (latest playbook on every job). |
| 3 | **Job Templates** | 21 templates, one per diagnostic playbook, named `diag-<domain>-<purpose>`, with the right inventory + credential + `become_enabled` for the domain. |

Template-to-playbook mapping lives in
[`aap-bootstrap/job_templates.yml`](../aap-bootstrap/job_templates.yml) —
edit there if you want to change names, descriptions, inventories, or
credentials.

After it finishes, in AAP Web UI you should see 21 new job templates
prefixed with `diag-`. The `playbook-executor` skill will pick them up via
keyword search.

---

## Option 2 — Manual via the AAP Web UI

For each playbook, do the following once:

### Step 1. Create the Project (one-time)

Resources → Projects → **Add**

| Field | Value |
|-------|-------|
| Name | `aiops-sre-skills` |
| Organization | `Default` |
| Source Control Type | `Git` |
| Source Control URL | `https://github.com/rht-alexlee/aiops-aap-sre-skills-pack.git` |
| Source Control Branch | `main` |
| Update Revision on Launch | ✓ |
| Clean | ✓ |

Save → click **Sync** → wait for the status to go green.

### Step 2. Create / verify Inventories (one-time)

Resources → Inventories → **Add**

Create at least:
- `linux-prod` — RHEL hosts the SRE agents touch.
- `windows-prod` — Windows hosts.
- `k8s-bastion` — the host(s) holding the kubeconfig; K8s playbooks ssh
  here and run `kubernetes.core` modules against the cluster.

Add hosts to each.

### Step 3. Create one Job Template per playbook

Resources → Templates → **Add → Job Template**

Use the values from [`aap-bootstrap/job_templates.yml`](../aap-bootstrap/job_templates.yml). For each row:

| Field | Value |
|-------|-------|
| Name | `diag-<domain>-<purpose>` (e.g. `diag-linux-health-check`) |
| Description | from `job_templates.yml` (the executor matches against this) |
| Job Type | `Run` |
| Inventory | per the table |
| Project | `aiops-sre-skills` |
| Playbook | path from `job_templates.yml`, e.g. `module/skills/linux-diagnostics/playbooks/rhel_health_check.yml` |
| Execution Environment | one with `ansible.windows` + `kubernetes.core` installed |
| Credentials | `linux-ssh` for `linux-*` and `k8s-bastion`, `windows-winrm` for `windows-*` |
| Options → Privilege Escalation | ✓ for Linux + network templates; **off** for Windows + k8s |
| Options → Prompt on launch → Variables | ✓ |
| Options → Prompt on launch → Limit | ✓ |
| Options → Prompt on launch → Inventory | ✗ (locked to the inventory above) |

Repeat for all 21 playbooks.

> Tip: in the UI, click **Copy** on the first template you create and just
> change the Name, Description, and Playbook fields for the rest.

### Step 4. Smoke-test one template

Pick `diag-linux-health-check` → **Launch**. Supply a `Limit` (one host) and
no extra vars. The job should complete in seconds and dump the structured
health report to stdout. If it does, the executor skill will be able to do
the same.

---

## Option 3 — Manual via the `awx` CLI

If you'd rather script it from a shell without writing a playbook:

```bash
pip install awxkit
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='change-me'

awx login

# Project (one-time)
awx projects create \
  --name aiops-sre-skills \
  --organization Default \
  --scm_type git \
  --scm_url https://github.com/rht-alexlee/aiops-aap-sre-skills-pack.git \
  --scm_branch main \
  --scm_update_on_launch true \
  --scm_clean true \
  --wait

# One job template (repeat per playbook)
awx job_templates create \
  --name diag-linux-health-check \
  --description "RHEL host triage — uptime, memory, disk, failed units, OOMs." \
  --job_type run \
  --inventory linux-prod \
  --project aiops-sre-skills \
  --playbook module/skills/linux-diagnostics/playbooks/rhel_health_check.yml \
  --ask_limit_on_launch true \
  --ask_variables_on_launch true \
  --become_enabled true

# Attach the credential
awx job_templates associate \
  --credential linux-ssh \
  diag-linux-health-check
```

If you're going to do this for all 21 playbooks, just use Option 1 — it's
the same logic, expressed once.

---

## Naming conventions (so the executor finds them)

The `playbook-executor` skill searches AAP with the sub-agent's keywords —
typically `<domain> <symptom>` (e.g. `"linux disk"`, `"kubernetes pod"`).
Match probability goes up sharply when:

- The **Name** starts with `diag-<domain>-` or `fix-<domain>-`.
- The **Description** mentions the symptom in plain English (e.g.
  "disk pressure", "CrashLoopBackOff", "DNS resolution", "MTU blackhole").

If you add your own playbooks/templates later, follow the same convention
and the agents will pick them up without code changes.

---

## What about remediation (`fix-*`) templates?

This pack only ships **read-only diagnostic** playbooks (`diag-*`).
Remediation templates are intentionally **out of scope** — they're
environment-specific and should be reviewed by your change-management
process before they ever run.

When you're ready, create them in AAP using the same pattern:

- Name `fix-<domain>-<purpose>` (e.g. `fix-linux-disk-cleanup`).
- Description mentioning the symptom you're fixing.
- Same project, inventories, and credentials.
- `become_enabled: true` (most fixes need root).
- `ask_limit_on_launch: true` so the agent can scope the blast radius.

The `playbook-executor` skill's keyword search will then surface them on
`/remediate`, gated on explicit user approval.

---

## Re-syncing after you change a playbook

`scm_update_on_launch: true` on the Project means every job pulls the
latest commit from `main` before running. After you push a change:

```bash
git push origin main
```

…the next agent-launched job will use it. If you want to force a sync
manually (e.g. to validate before an oncall handoff):

AAP Web UI → Projects → `aiops-sre-skills` → **Sync**.

Or via CLI:

```bash
awx projects update aiops-sre-skills --wait
```
