# Agent Coordination

Canonical agent instructions live in [CLAUDE.md](CLAUDE.md). This file holds
only multi-tool coordination.

**Claim before you code — one issue = one agent.** Start an issue only if it
carries no `agent:*` label and no `*/issue-<N>-*` branch or open PR from any
tool exists. Claim it with your tool's label (`agent:claude`, `agent:codex` or
`agent:cursor`), work on a `<tool>/issue-<N>-*` branch, and remove the label
when you finish or stop. The full protocol is in `traverse-framework/.github`
`docs/ai-agent-hardening.md`.
