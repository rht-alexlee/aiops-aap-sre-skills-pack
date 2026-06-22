---
description: Execute Ansible playbooks via AAP. The ONLY path AIOps SRE sub-agents use to run AAP job templates — never call AAP MCP tools directly. Selects the best-matching pre-built template or runs newly generated playbooks via `adhoc-fix-job`.
---

# AAP Playbook Executor

This skill is the single entry point for **all** AAP execution in the AIOps
SRE skill pack. Sub-agents (`sre-linux`, `sre-windows`, `sre-kubernetes`,
`sre-networking`) MUST invoke this skill — they MUST NOT call
`job_templates_launch_retrieve` or any other `aap-mcp-*` tool directly.

## Two selection modes

- **Mode A — Match Existing Template**: when the sub-agent's intent maps to a
  pre-built template (e.g. `diag-linux-health-check`, `fix-rhel-disk-cleanup`),
  search the AAP catalog by keyword and pick the best match.
- **Mode B — Ad-hoc Generated Playbook**: when the sub-agent generated a
  fresh remediation playbook for this incident, always launch via the
  reserved `adhoc-fix-job` template.

## Prerequisites

**Required MCP servers**: `aap-mcp-job-management`, `aap-mcp-inventory-management`
(declared in `module/mcps.json`).

**Required MCP tools**:

| Server | Tool |
|--------|------|
| aap-mcp-job-management | `job_templates_list`, `job_templates_retrieve`, `job_templates_launch_retrieve`, `jobs_retrieve`, `jobs_stdout_retrieve`, `jobs_job_events_list`, `jobs_job_host_summaries_list`, `jobs_relaunch_retrieve`, `projects_list` |
| aap-mcp-inventory-management | `inventories_list`, `hosts_list` |

**Environment**: `AAP_MCP_SERVER`, `AAP_API_TOKEN`, optionally `AAP_BASE_URL`
for UI links.

## How sub-agents invoke this skill

A sub-agent calls this skill with a structured intent:

```yaml
intent:
  action: diagnose            # or "remediate"
  domain: linux               # linux | windows | kubernetes | networking
  symptom_keywords: ["disk", "full", "/var/log"]
  target:
    inventory: linux-prod
    limit: web-01,web-02
  extra_vars:                 # optional, merged into job launch
    incident_slug: web-disk-full-2026-06-22
  generated_playbook_path: null   # set to a path for Mode B; null for Mode A
```

This skill:
1. **Selects** the template (Mode A keyword search, or Mode B = `adhoc-fix-job`).
2. **Validates** the template (Step 1.4 below).
3. **Confirms** with the user before launch (mandatory).
4. **Launches** the job, **polls** until terminal, **collects** stdout +
   host summaries + events.
5. **Reports** back to the calling sub-agent with a structured result.

## Workflow

### Phase 1: Job Template Selection

#### Step 1.1: Decide Selection Mode

Use Mode B if `intent.generated_playbook_path` is set. Otherwise Mode A.

#### Step 1.2: List and Search Templates (Mode A)

**MCP Tool**: `job_templates_list` (from `aap-mcp-job-management`)

**Parameters**:
- `page_size`: 50
- `search`: a single string formed by joining `intent.symptom_keywords`
  (and `intent.domain` as a prefix, e.g. `"linux disk"`).

For each candidate template in the result:
1. Call `job_templates_retrieve(id)` to get full details.
2. Score by name + description match against the intent.
3. Run the inline validation (Step 1.4).
4. Present the top 3 to the user.

#### Step 1.3: Select `adhoc-fix-job` (Mode B)

**MCP Tool**: `job_templates_list` with `search: "adhoc-fix-job"`.
Retrieve by id, validate, then proceed.

The `adhoc-fix-job` template MUST be configured with
`ask_variables_on_launch: true` and `ask_limit_on_launch: true`.

#### Step 1.4: Inline Job Template Validation

Read-only checks. Replaces any external validator.

##### Required (must all pass)

| Requirement | Field / Logic | Why |
|-------------|---------------|-----|
| Inventory | `template.inventory` non-null | Target hosts must be defined |
| Project | `template.project` non-null | Playbook source must exist |
| Playbook | `template.playbook` non-empty string | Playbook path required |
| Credentials | `summary_fields.credentials` ≥ 1 | SSH/WinRM access |
| Privilege escalation | `template.become_enabled == true` | Most fixes need root |

##### Recommended (warnings only)

| Requirement | Field |
|-------------|-------|
| Ask Variables on Launch | `ask_variables_on_launch == true` |
| Ask Limit on Launch | `ask_limit_on_launch == true` |
| Ask Inventory on Launch | `ask_inventory_on_launch == true` |

##### Context (optional)

- Project status via `projects_list` — warn if `failed`/`error`, wait if
  `pending`/`running`.
- Inventory existence via `inventories_list`.

##### Result determination

- **PASSED** — proceed.
- **PASSED WITH WARNINGS** — proceed only if user accepts.
- **FAILED** — STOP. Tell the user which check failed and how to fix the
  template in AAP Web UI.

#### Step 1.5: Present Selection to User

```
Selected job template:

→ "{template.name}" (ID: {template.id})
   Description: {template.description}
   Playbook: {template.playbook}
   Match reason: {keyword match | "ad-hoc remediation generated for this incident"}

Validation: ✓ PASSED

❓ Proceed with this template? (yes / choose another / abort)
```

Wait for explicit confirmation.

### Phase 2: Execution

#### Step 2.1: Final Confirmation

```
⚠️ Playbook Execution Confirmation

Target hosts:       {N} systems ({limit pattern})
Inventory:          {inventory.name}
Playbook:           {template.playbook OR "ad-hoc generated for incident X"}
Issue:              {symptom summary}
Job template:       {name} (ID: {id})

❓ Execute now?  (yes / execute / abort)
```

Wait for explicit "yes" or "execute".

#### Step 2.2: Launch Job

**MCP Tool**: `job_templates_launch_retrieve`

**Parameters**:
```json
{
  "id": "{template_id}",
  "requestBody": {
    "job_type": "run",
    "extra_vars": {
      "incident_slug": "{intent.extra_vars.incident_slug}",
      "remediation_mode": "automated",
      "...": "merge intent.extra_vars here"
    },
    "limit": "{intent.target.limit}"
  }
}
```

For Mode B, also pass the generated playbook reference per the
`adhoc-fix-job` template definition.

#### Step 2.3: Monitor

Poll `jobs_retrieve` every 2s and stream `jobs_job_events_list`. Render:

```
⏳ Job 1235 — running   (elapsed 1m 23s)
URL: {AAP_BASE_URL}/#/jobs/playbook/1235

- ✓ Gather facts
- ✓ Pre-checks
- ⏳ Apply fix
- ⏸  Verify
```

Stop when status is `successful`, `failed`, or `error`.

### Phase 3: Execution Report

**Read [references/01-execution-report-templates.md](references/01-execution-report-templates.md)** for the full templates.

Gather: `jobs_retrieve(id)`, `jobs_job_host_summaries_list(id)`,
`jobs_job_events_list(id)`, `jobs_stdout_retrieve(id, format="txt")`.

#### Step 3.3 — MANDATORY: Issue-handling validation

Parse stdout for:
- references to the incident ID / alert ID / target service
- task names matching the remediation intent (e.g. "Apply fix", "Restart …")
- modules that actually performed an action (`dnf`, `service`, `systemd`,
  `file`, `template`, `win_service`, `k8s`)

Report **✓ confirmed** or **⚠️ inconclusive — recommend manual verification**.

### Phase 4: Error Handling

**Read [references/02-error-handling-guide.md](references/02-error-handling-guide.md)**.

On `failed`/`error`: categorize, render the error report, offer relaunch via
`jobs_relaunch_retrieve` with `{"hosts": "failed", "job_type": "run"}`.

## Reference files

| File | Use when |
|------|----------|
| [references/01-execution-report-templates.md](references/01-execution-report-templates.md) | Phase 3 reports |
| [references/02-error-handling-guide.md](references/02-error-handling-guide.md) | Phase 4 errors / relaunch |
| [references/03-workflow-examples.md](references/03-workflow-examples.md) | End-to-end demos |

## Return value to calling sub-agent

```yaml
job:
  id: 1235
  template: diag-linux-disk
  status: successful           # | failed | error | partial
  url: https://aap.example.com/#/jobs/playbook/1235
  duration_seconds: 47
hosts:
  ok: [web-01, web-02]
  failed: []
  unreachable: []
stdout_path: investigations/<slug>/aap-job-1235.txt
issue_handled: confirmed       # | inconclusive
```

The sub-agent uses this structure to write its `<domain>-diag.md` /
`<domain>-remediation.md` report.

## Human-in-the-loop rules

1. Show execution summary (targets, playbook, intent).
2. Ask for explicit "yes" / "execute" — never assume approval.
3. Never proceed without it.

## Best practices

1. **Best-match search** on template Name + Description.
2. **`adhoc-fix-job` for generated playbooks** — always.
3. **Validate before launch** (Step 1.4).
4. **Check project sync** — stale playbooks cause stale fixes.
5. **Stream events** during execution.
6. **Full reporting** — per-host stats + task timeline + stdout.
7. **Categorize errors** and offer targeted troubleshooting.
8. **Relaunch failed hosts only** with `hosts: "failed"`.
9. **Link to AAP UI** in every report.
10. **Validate issue handling** in stdout (Step 3.3).
11. **Audit trail** — persist job ID + slug under `investigations/`.
