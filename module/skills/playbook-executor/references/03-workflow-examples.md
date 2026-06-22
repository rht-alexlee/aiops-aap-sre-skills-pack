# Step 03: Workflow Examples

Read this reference when demonstrating end-to-end workflow patterns.

## Example 1: Match Existing Template (Mode A)

**User Request**: "We have an alert for high disk usage on the web tier — run the existing disk cleanup playbook."

**Skill Response**:

1. **List Job Templates** (Mode A):
   - Call `job_templates_list(search="disk")` → 3 candidates returned.
   - Call `job_templates_retrieve(id)` for each.
   - Best match by Name/Description: **"Disk Cleanup — Web Tier"** (ID: 22).

2. **Inline Template Validation (Step 1.4)**:
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
   - `job_templates_launch_retrieve(id=22, requestBody={job_type: "run", limit: "prod-web-01,prod-web-02,prod-web-03", extra_vars: {issue_id: "INC-20260224-001"}})`
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

## Example 2: Ad-hoc Generated Playbook (Mode B with `adhoc-fix-job`)

**User Request**: "We have an unusual incident — generate a fix playbook and run it."

**Skill Response**:

1. **Agent context**: A new playbook was just produced by `playbook-generator` (e.g. `playbooks/remediation/fix-INC-20260224-002.yml`).

2. **Select `adhoc-fix-job`** (Mode B):
   - Call `job_templates_list(search="adhoc-fix-job")` → ID 7.
   - Call `job_templates_retrieve(id="7")`.

3. **Inline Template Validation (Step 1.4)**:
   - Required ✓, Recommended ✓ (especially `ask_variables_on_launch: true` so we can pass the playbook reference at launch).
   - Result: **✓ PASSED**.

4. **Final Confirmation** → user replies `execute`.

5. **Launch Job**:
   ```json
   {
     "id": "7",
     "requestBody": {
       "job_type": "run",
       "extra_vars": {
         "issue_id": "INC-20260224-002",
         "remediation_mode": "automated",
         "playbook_ref": "playbooks/remediation/fix-INC-20260224-002.yml"
       },
       "limit": "prod-app-01,prod-app-02"
     }
   }
   ```

6. **Monitor** → COMPLETED.

7. **Generate Report**: Success template with full per-host details.

8. **Step 3.3 Validation**: ✓ Job log references `INC-20260224-002` and shows the expected remediation tasks.

9. **Suggest Verification**:
   ```
   Recommend: Run remediation-verifier to confirm the underlying symptom is resolved.
   ```

## Example 3: Handle Execution Failure (Partial Success)

**User Request**: "Execute the remediation playbook for INC-20260224-003."

**Skill Response**:

1–4. [Same as Example 1 or 2 through launch.]

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

2. **Inline Template Validation (Step 1.4)**:
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

1. **Search**: `job_templates_list(search="memory")` → 3 candidates.

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

   Select template (1 or 2), or "adhoc" to generate a new fix playbook.
   ```

4. User selects `2` → continue with Final Confirmation → Launch → Monitor → Report.
```

