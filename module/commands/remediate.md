---
description: Propose and (after explicit approval) execute a remediation playbook in AAP for a previously diagnosed incident.
argument-hint: "<incident-slug>"
---

# /remediate

Propose and run a remediation. **Requires explicit user approval** before any
state-changing playbook executes.

## User-provided arguments

> $ARGUMENTS

## Instructions

1. Treat `$ARGUMENTS` as the `<incident-slug>`. If empty, list the slugs under
   `investigations/` and ask the user to choose one.
2. Read `investigations/<slug>/*.md`. If no diagnosis exists, stop and tell
   the user to run `/diagnose` first.
3. Delegate to the specialist sub-agent that owns this domain. The specialist
   will ask `playbook-executor` to match the diagnosis against a pre-built
   remediation template in the AAP catalog (`fix-<domain>-<purpose>`). If no
   suitable template exists, the specialist must STOP and tell the user that
   an operator needs to register one — do not improvise.
4. The specialist MUST present the proposed template + target hosts to the
   user and wait for an explicit "yes" / "execute" before invoking
   `playbook-executor`.
5. After execution, save the report to
   `investigations/<slug>/<domain>-remediation.md` and link the AAP job URL.
