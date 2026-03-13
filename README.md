# codex-agentic-infraops

A contract-first, multi-agent workflow for governed Azure infrastructure delivery.
Built to run with Codex in the CLI, Codex app, and the VS Code extension.

This project turns natural-language infrastructure intent into AVM-first Bicep with strict checkpoints, explicit approvals, and schema-validated evidence artifacts.

## 🎯 Why did I create this?

Most AI infra generation is fast but hard to trust. This repo optimizes for trust and repeatability:

- 📦 schema-valid handoff envelopes between agents
- 🔐 two explicit approval gates before authoring/deployment-critical actions
- 🧩 mandatory service-option confirmations when multiple valid options exist
- 🛑 fail-closed validation with remediation loops
- 🧾 full evidence trail under `output/<run_id>/`

Common AI-generated infra challenges this **tries** to solve:

- ⚖️ consistency: the same prompt can produce different architectures between runs
- 🧱 AVM discipline: agents can drift into raw `resource` declarations instead of AVM modules
- 🛡️ AVM producer-owned capabilities: built-in options like `roleAssignments` and `privateEndpoints` are often missed

## 🗺️ Workflow

| Phase | Owner | Purpose | Output artifact |
| --- | --- | --- | --- |
| 0. Orchestration | `root_orchestrator` | Manage phase lifecycle, approvals, confirmations, and contract enforcement | `progress.events.v2.json` + gate markers |
| 1. Requirements | `requirements` agent | Normalize intent and constraints | `requirements.v2.json` |
| 2. Research | `research` agent | Gather Microsoft Learn evidence and option analysis | `research.evidence.v2.json` |
| 3. Authoring | `code` agent | Generate AVM-first Bicep after approvals | `authoring.evidence.v2.json` + `infra/` |
| 4. Validation | `validate` agent | Strict pass/fail checks and blockers | `validator.evidence.v2.json` |

## 📜 Contract-first design

We use contracts so every phase handoff is machine-checkable, reproducible, and reviewable.
In practice, that means no agent is allowed to “wing it” with informal text when a structured schema exists.

Why this matters:

- Predictable behavior: each phase must emit schema-valid envelopes and artifacts
- Better debugging: failures point to exact contract violations instead of ambiguous prompt behavior
- Auditability: you can trace decisions from requirements -> research -> authoring -> validation
- Safer automation: approval gates and validator pass/fail status are enforced by artifact state

Where to find the contracts:
  - `.codex/protocols/*`

## 🧱 Repository layout

- `.codex/agents/`: phase agent TOML configs and instruction files
- `.codex/protocols/`: envelope, compact contract, and progress catalog schemas
- `.codex/protocols/artifacts/`: persisted evidence schema contracts
- `.codex/skills/`: `bicep-avm-code` and `bicep-avm-validate` implementation skills
- `output/`: run-scoped outputs (`output/<run_id>/...`)
- `AGENTS.md`: bootstrap rules and repo guardrails

## ⚙️ Runtime prerequisites

- Bicep installed
- Codex runtime with multi-agent support
- MCP server access:
  - `microsoft_learn` (research phase)
  - `bicep` (authoring and validation phases)

Use this guide to install the [MCP servers](https://github.com/Hardstl/skills/blob/main/docs/mcp-setup.md).

## 🚦 Quick start

1. Add this project as trusted in your global Codex config:

```toml
[projects.'C:\path\to\codex-agentic-infraops']
trust_level = "trusted"
```

2. Open this repository as your Codex workspace.
3. Start with an infrastructure intent prompt, for example:

```text
Create an Azure Function App and Key Vault with private endpoints and no public access in a new virtual network with address space 10.200.0.0/24.
```

4. Follow the workflow prompts for requirements, option confirmations, and approvals.
5. Review outputs in `output/<run_id>/`.

## ✍️ Prompting

Prompt quality matters. Better prompts reduce rework in requirements/research, improve AVM module selection, and produce cleaner first-pass authoring.

Good prompts usually include:

- workload intent (what you are building)
- security/network constraints (private endpoints, no public access, etc.)
- environment context (`dev`/`test`/`prod`)
- scale/cost expectations

## ✅ What a successful run produces

Typical artifacts under `output/<run_id>/`:

- `requirements.v2.json`
- `research.evidence.v2.json`
- `service.confirmations.v2.json`
- `authoring.evidence.v2.json`
- `validator.evidence.v2.json`
- `progress.events.v2.json`
- `.gate1.approved`
- `.gate2.approved`
- `.validator.passed`
- `infra/` (generated Bicep files)
