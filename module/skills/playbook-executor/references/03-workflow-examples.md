# Step 03: Workflow Examples

Read this reference when demonstrating end-to-end workflow patterns.

## Example 1: Match Existing Template

**User Request**: "We have an alert for high disk usage on the web tier — run the existing disk cleanup playbook."

**Skill Response**:

1. **List Job Templates**:
   - Call `job_templates_list(search="linux disk")` → 3 candidates returned.
   - Call `job_templates_retrieve(id)` for each.
   - Best match by Name/Description: **"Disk Cleanup — Web Tier"** (ID: 22).

2. **Inline Template Validation (Step 1.3)**:
   - Required checks: Inventory ✓, Project ✓, Playbook ✓, Credentials ✓, become_enabled ✓
   - Recommended: ask_variables_on_launch ✓, ask_limit_on_launch ✓
   - Project status: `successful` ✓
   - Result: **✓ PASSED**

3. **Present Selection**:
   ```
   Selected: "Disk Cleanup — Web Tier" (ID: 22)
   Match reason: Description mentions "/var/log cleanup and log rotation"
   Validation: ✓ PASSED
   Proceed? yes
   ```

4. **Final Confirmation** → user replies `execute`.

5. **Launch Job**:
   - `job_templates_launch_retrieve(id=22, requestBody={job_type: "run", limit: "prod-web-01,prod-web-02,prod-web-03", extra_vars: {incident_slug: "web-disk-full-2026-02-24"}})`
   - Job ID 1234 created.

6. **Monitor** → COMPLETED, status `successful`.

7. **Generate Report**:
   ```
   ✅ 3 of 3 hosts successfully remediated
   Duration: 5m 23s
   ```

8. **Validate Issue Handling in stdout (Step 3.3)**:
   - ✓ Job log shows /var/log cleanup tasks executed on all hosts.

9. **Suggest Verification**:
   ```
   Recommend: Run remediation-verifier to confirm disk usage is back to normal.
   ```

## Example 2: No Matching Template — Stop and Refer to Operator

**User Request**: "We have an unusual incident — run a fix."

**Skill Response**:

1. **Search** the catalog with the available keywords:
   `job_templates_list(search="kubernetes ingress cert renewal")` → 0 candidates.

2. **Stop and report**:
   ```
   ✗ No pre-registered AAP job template matches this intent.

   Searched: "kubernetes ingress cert renewal"
   Candidates: 0

   This executor only runs templates that already exist in the AAP catalog.

   Next steps:
   - Ask an operator to register a template using the convention
     fix-<domain>-<purpose>  (e.g. fix-k8s-ingress-cert-renewal).
   - Re-run /diagnose once the template is registered.
   ```

3. **Do NOT launch any playbook.** Wait for the operator to add the template
   to AAP, then retry.

## Example 3: Handle Execution Failure (Partial Success)

**User Request**: "Execute the remediation playbook for INC-20260224-003."

**Skill Response**:

1–4. [Same as Example 1 through launch.]

5. **Execute Playbook**:
   - Launch job → FAILED on 1 of 3 hosts.
   - Generate error report:
     ```
     ❌ prod-web-03 failed
     Task: Restart affected service
     Error: Unit not found
     Category: Service Failure
     Troubleshooting steps provided
     ```

6. **Offer Relaunch**:
   ```
   Relaunch for failed host only? yes
   ```

7. **Relaunch Job**:
   - `jobs_relaunch_retrieve(id="1235", requestBody={hosts: "failed", job_type: "run"})`
   - Monitor → COMPLETED.
   - Final report:
     ```
     ✅ All 3 hosts successfully remediated (1 after retry)
     ```

## Example 4: Template Validation Fails — Stop Before Launch

**User Request**: "Use template ID 15 to apply the fix."

**Skill Response**:

1. **Retrieve Template**: `job_templates_retrieve(id="15")`.

2. **Inline Template Validation (Step 1.3)**:
   - `become_enabled` is `false` → **✗ FAILED** (required).

3. **Stop and Report**:
   ```
   ✗ Template validation FAILED

   Failed required check: Privilege Escalation (become_enabled is false)

   Remediation playbooks generally require sudo/root for system changes.

   Fix: AAP Web UI → Templates → [Template 15] → Edit →
        Options → ✓ Enable Privilege Escalation, then save and retry.
   ```

4. **Do NOT launch**. Wait for user to either fix the template or choose a different one.

## Example 5: Multiple Candidate Templates — User Selects

**User Request**: "There's a memory leak on app servers — run a fix."

**Skill Response**:

1. **Search**: `job_templates_list(search="linux memory")` → 3 candidates.

2. **Validate each** with the inline checks. 2 PASS, 1 FAILS (no credentials).

3. **Present passing candidates**:
   ```
   Found 2 compatible job templates for "memory" issues:

   1. "Restart App Service on High Memory" (ID: 31)
      Description: Restarts the app service when RSS exceeds threshold.
      Validation: ✓ PASSED

   2. "Memory Leak Mitigation — Worker Recycle" (ID: 44)
      Description: Recycles worker processes safely.
      Validation: ✓ PASSED WITH WARNINGS (ask_limit_on_launch: false)

   Select template (1 or 2), or "abort".
   ```

4. User selects `2` → continue with Final Confirmation → Launch → Monitor → Report.
