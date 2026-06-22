# AIOps SRE Skill Pack

A Lola-packaged AI Context Module that gives an LLM agent a four-specialist
SRE team. Each specialist diagnoses problems in its domain by running
**Ansible playbooks through AAP** via a single shared `playbook-executor`
skill, then writes findings to disk.

```
ops_orchestrator
├── sre-linux        — RHEL / generic Linux
├── sre-windows      — Windows Server
├── sre-kubernetes   — K8s / OpenShift (via bastion)
└── sre-networking   — DNS, L2, L3
```

All AAP execution flows through `module/skills/playbook-executor/` —
sub-agents **never** call AAP MCP tools directly. The executor only launches
**pre-registered AAP job templates**, matched by name + description against
the AAP catalog. If no template matches the sub-agent's intent, it stops and
asks an operator to register one (using the `diag-<domain>-<purpose>` /
`fix-<domain>-<purpose>` naming convention).

## Layout

```
aiops-aap-sre-skills-pack/
├── README.md                  ← this file
├── marketplace.yaml           ← Lola marketplace entry
├── subagents.yaml             ← LangChain Deep Agents wiring
├── AGENTS.md                  ← top-level orchestrator persona
├── .ansible-lint              ← lint config (production profile)
└── module/                    ← Lola AI Context Module root
    ├── AGENTS.md              ← module-level description (Lola spec)
    ├── mcps.json              ← AAP MCP server config
    ├── memory/
    │   └── conventions.md     ← shared facts (inventories, change mgmt)
    ├── agents/                ← persona files
    │   ├── sre-orchestrator.md
    │   ├── sre-linux.md
    │   ├── sre-windows.md
    │   ├── sre-kubernetes.md
    │   └── sre-networking.md
    ├── commands/              ← slash commands
    │   ├── diagnose.md
    │   └── remediate.md
    └── skills/
        ├── log-analysis/SKILL.md                  ← shared
        ├── playbook-executor/                     ← AAP launcher
        │   ├── SKILL.md
        │   └── references/{01,02,03}-*.md
        ├── linux-diagnostics/{SKILL.md,playbooks/*.yml}
        ├── windows-diagnostics/{SKILL.md,playbooks/*.yml}
        ├── kubernetes-troubleshooting/{SKILL.md,playbooks/*.yml}
        ├── networking-dns/{SKILL.md,playbooks/*.yml}
        ├── networking-layer2/{SKILL.md,playbooks/*.yml}
        └── networking-layer3/{SKILL.md,playbooks/*.yml}
```

This follows the [Lola AI Context Module spec](https://github.com/RedHatProductSecurity/lola/blob/main/docs/concepts/skills-and-modules.md):
`module/AGENTS.md` + `module/skills/<name>/SKILL.md` + `module/commands/` +
`module/agents/` + `module/mcps.json`.

## Bundled playbooks

| Skill | Playbooks |
|-------|-----------|
| `linux-diagnostics` | `rhel_health_check`, `rhel_service_diagnose`, `rhel_disk_diagnose`, `rhel_perf_snapshot` |
| `windows-diagnostics` | `windows_health_check`, `windows_service_diagnose`, `windows_eventlog_collect`, `windows_disk_diagnose` |
| `kubernetes-troubleshooting` | `k8s_cluster_health`, `k8s_pod_diagnose`, `k8s_node_pressure` |
| `networking-dns` | `dns_resolution_check`, `dns_resolver_config`, `dns_zone_query` |
| `networking-layer2` | `l2_link_check`, `l2_arp_audit`, `l2_bridge_vlan` |
| `networking-layer3` | `l3_reachability`, `l3_firewall_audit`, `l3_mtu_probe`, `l3_routing_protocols` |

All playbooks are **read-only** diagnostics. They pass `ansible-lint` at the
production profile:

```bash
ansible-lint module/skills/*/playbooks/*.yml
```

## Required collections

```bash
ansible-galaxy collection install ansible.windows kubernetes.core
```

## Uploading the playbooks to AAP

The agents only execute playbooks that are registered as **job templates**
in AAP. AAP itself pulls the playbook files from this Git repo via a Project;
you just need to register the project and create one template per playbook.

A one-shot bootstrap playbook + a manual walkthrough live in
[`docs/upload-playbooks-to-aap.md`](docs/upload-playbooks-to-aap.md).
Short version:

```bash
ansible-galaxy collection install awx.awx
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='...'

ansible-playbook aap-bootstrap/register_in_aap.yml \
  -e organization=Default \
  -e project_scm_url=https://github.com/rht-alexlee/aiops-aap-sre-skills-pack.git \
  -e project_scm_branch=main \
  -e ssh_credential_name=linux-ssh \
  -e winrm_credential_name=windows-winrm
```

That creates the `aiops-sre-skills` project, the three inventories, and 21
job templates named `diag-<domain>-<purpose>` — exactly what
`playbook-executor` searches for.

## Required MCP servers

Configured in `module/mcps.json`:

- `aap-mcp-job-management`
- `aap-mcp-inventory-management`

Both need the `AAP_MCP_SERVER` and `AAP_API_TOKEN` environment variables.

---

## Using the pack in each tool

### 1. Claude Code

Install via Lola:

```bash
# Marketplace pointer for your team
lola market add sre file:///path/to/aiops-aap-sre-skills-pack/marketplace.yaml
# Or just install directly from a clone:
lola mod add ./module
lola install aiops-aap-sre-skills-pack -a claude-code
```

Lola lays the module out under `~/.claude/skills/` (skills),
`~/.claude/commands/` (commands), `~/.claude/agents/` (sub-agents), and merges
`mcps.json` into `~/.claude.json`. Then in a Claude Code session:

```
/diagnose pod web-api-7d9 in prod-web CrashLoopBackOff since 14:30
/remediate web-api-crashloop-2026-06-22
```

The `sre-orchestrator` agent classifies the symptom and invokes
`sre-kubernetes` (via the Claude Code Agent tool). The sub-agent loads its
skills, picks the right playbook, calls the `playbook-executor` skill which
in turn talks to the AAP MCP servers.

### 2. Cursor

Lola installs the same module into Cursor's `~/.cursor/rules/` and
`~/.cursor/mcp.json`:

```bash
lola install aiops-aap-sre-skills-pack -a cursor
```

In Cursor you invoke a sub-agent via `@sre-linux …` or use the slash
commands the module shipped. The skills bundled under each agent's persona
load on demand. MCP servers are registered automatically from `mcps.json`.

### 3. Open WebUI

Open WebUI loads "tools" and "models" (system prompts) from per-workspace
directories. There is no native Lola adapter today, but the module is plain
markdown + JSON so the wiring is mechanical:

1. **MCP**: register both AAP MCP servers in Open WebUI's connections panel
   using the URLs from `module/mcps.json`.
2. **Personas**: paste the contents of each `module/agents/*.md` file into a
   new Open WebUI **Model** as the system prompt; name the model
   `sre-linux`, `sre-windows`, etc.
3. **Skills**: paste the matching `module/skills/<name>/SKILL.md` content
   into the model's "Knowledge" panel (or attach the whole `skills/`
   directory as a Knowledge source). The persona references the skill by
   name; the model loads it via RAG.
4. **Commands**: implement `/diagnose` and `/remediate` as Open WebUI
   **Prompt** templates whose body is the instructions in
   `module/commands/*.md`.

A community-maintained Lola adapter for Open WebUI is on the Lola roadmap;
until then this manual wiring is one-time per workspace.

### 4. LangChain Deep Agents

`subagents.yaml` at the repo root is the wiring file. A minimal loader:

```python
# ops_manager.py
from pathlib import Path
import yaml
from deepagents import create_deep_agent, SubAgent
from langchain_anthropic import ChatAnthropic

# AAP MCP tools loaded via langchain-mcp-adapters
from langchain_mcp_adapters.client import MultiServerMCPClient

ROOT = Path(__file__).parent
cfg  = yaml.safe_load((ROOT / "subagents.yaml").read_text())

mcp_client = MultiServerMCPClient(yaml.safe_load(
    (ROOT / "module/mcps.json").read_text())["mcpServers"])
aap_tools = mcp_client.get_tools()    # job_templates_list, ...

def load_skill_files(paths):
    """Inline SKILL.md (+ playbooks/) content as system prompt context."""
    chunks = []
    for p in paths:
        for skill in Path(ROOT, p).rglob("SKILL.md"):
            chunks.append(skill.read_text())
        for pb in Path(ROOT, p).rglob("playbooks/*.yml"):
            chunks.append(f"### playbook {pb.name}\n```yaml\n{pb.read_text()}\n```")
    return "\n\n---\n\n".join(chunks)

subagents = [
    SubAgent(
        name=sa["name"],
        description=Path(ROOT, sa["persona"]).read_text().split("---")[1],
        prompt=Path(ROOT, sa["persona"]).read_text() + "\n\n" +
               load_skill_files(sa["skills"]),
        tools=aap_tools,
        model=ChatAnthropic(model="claude-opus-4"),
    )
    for sa in cfg["subagents"]
]

ops = create_deep_agent(
    instructions=(ROOT / "AGENTS.md").read_text(),
    subagents=subagents,
    tools=aap_tools,
    model=ChatAnthropic(model="claude-opus-4"),
)

if __name__ == "__main__":
    import sys
    print(ops.invoke({"messages": [{"role": "user", "content": sys.argv[1]}]}))
```

```bash
uv run python ops_manager.py "Pods in prod-web CrashLoopBackOff after deploy"
```

The Deep Agents `ops_manager` reads `AGENTS.md`, delegates via its `task`
tool to the right `SubAgent`, and the sub-agent loads its skills (inlined
in the system prompt) before invoking the AAP MCP tools.

---

## Validating the pack

```bash
# Collections
ansible-galaxy collection install ansible.windows kubernetes.core

# Lint
ansible-lint module/skills/*/playbooks/*.yml

# (Optional) try a diagnostic playbook against a host
ansible-playbook -i inventory \
  module/skills/linux-diagnostics/playbooks/rhel_health_check.yml \
  -e incident_slug=smoke-test
```

## Extending

**Add a new domain (e.g. `sre-database`):**

1. Create `module/skills/database-diagnostics/SKILL.md` + `playbooks/*.yml`.
2. Add a `module/agents/sre-database.md` persona.
3. Add an entry to `subagents.yaml` (for the LangChain wiring).
4. Update `module/AGENTS.md` and `AGENTS.md` so the orchestrator routes to it.
5. `ansible-lint` the new playbooks.

**Add a new playbook to an existing skill:**

1. Drop the playbook in `module/skills/<skill>/playbooks/`.
2. Add a row to the **Playbooks** table in that skill's `SKILL.md`.
3. Lint.
4. Register a matching AAP job template using the
   `diag-<domain>-<purpose>` or `fix-<domain>-<purpose>` naming convention
   so `playbook-executor`'s keyword search will pick it up.

## License

Apache-2.0 (matching the upstream `playbook-executor` skill).
