# Requirements Subagent Retired

The dedicated `requirements` subagent has been retired.

Requirements intake is now owned directly by the root orchestrator, which:

- collects the 6 deterministic intake fields in root CLI
- validates the normalized payload against `.codex/protocols/requirements.v2.schema.json`
- writes `output/<run_id>/requirements.v2.json` using `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`

Active contract sources for requirements intake are:

- `.codex/agents/instructions/root_orchestrator.md`
- `.codex/protocols/requirements.v2.schema.json`
- `.codex/protocols/artifacts/requirements-artifact.v2.schema.json`

Do not spawn or configure a `requirements` subagent for normal workflow execution.
