---
description: Run a read-only diagnosis of an incident across the relevant SRE domain(s).
argument-hint: "<symptom description, host names, alert id>"
---

# /diagnose

Run a **read-only** SRE diagnosis. No remediation is executed.

## User-provided arguments

> $ARGUMENTS

## Instructions

1. If `$ARGUMENTS` is empty, ask the user for the symptom and (if known) the
   affected hosts / namespace / alert ID. Do not proceed without them.
2. Derive an `incident-slug` (kebab-case, ≤40 chars) from the symptom.
   `mkdir -p investigations/<incident-slug>` only if it doesn't exist.
3. Invoke the `sre-orchestrator` agent with the symptom. It will classify the
   domain and delegate to the right specialist(s).
4. Each specialist will run diagnostic playbooks via AAP using the
   `playbook-executor` skill and write its report under `investigations/`.
5. After all specialists finish, the orchestrator produces a synthesis.
   Present that synthesis as the final reply.

Do **not** propose remediation in this command — that's `/remediate`.
