# Repository Bootstrap Instructions

Workflow entrypoint:
- Read only `.codex/agents/instructions/root_orchestrator.md` first.
- Do not perform repo-wide discovery before reading the root orchestrator instructions.
- After loading root orchestrator instructions, follow its phase/file allowlists for all further reads.

- Do not spawn a `root_orchestrator` subagent.
- Treat `.codex/agents/instructions/root_orchestrator.md` as the orchestration prompt source.
- Treat `.codex/protocols/*.json` and `.codex/protocols/artifacts/*.json` as the canonical contract source.

Contract precedence (highest to lowest):
1. `.codex/protocols/*.json` and `.codex/protocols/artifacts/*.json`
2. `.codex/agents/instructions/root_orchestrator.md`
3. Other instruction markdown files (`AGENTS.md`, subagent instruction files)

If instruction text conflicts with schema contracts, schemas win.
If `AGENTS.md` conflicts with `root_orchestrator.md`, `root_orchestrator.md` wins.

Repo-wide guardrails:
- Do not bypass approval gates, service confirmations, or validator pass requirements for Azure-mutating work.
- Do not treat prompt text as the source of truth when it conflicts with schema contracts or approved workflow artifacts.
- Keep subagent interactions envelope-based; do not pass raw conversation transcripts as implementation handoff.
- Never use `fork_context`; keep `fork_context=false` for all orchestrated agent interactions.
- Scope reads by role: root orchestrator reads should stay minimal and orchestration-focused, while skill internals under `.codex/skills/**` are subagent-owned (`code`/`validate`) except explicit debugging.
- Prohibit reads from `.codex/protocols/tests/**` for both root orchestrator and all subagents.
- Root orchestrator progress lines must be emitted only from `.codex/protocols/progress-event-catalog.v2.json`; ad-hoc progress text is not allowed.
