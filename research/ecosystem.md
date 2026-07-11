# Ecosystem survey

Surveyed mid-2026. The question this survey answers: does something already exist that we should adopt instead of building, and if we do build, what should we steal? The conclusion is that **nothing off-the-shelf is polyrepo-native**, so blizzard is worth building — but several projects contribute ideas worth taking.

## The projects

### Agent Orchestrator (ComposioHQ)

- Apache 2.0; Go daemon + TypeScript desktop app; ~8k stars; [github.com/ComposioHQ/agent-orchestrator](https://github.com/ComposioHQ/agent-orchestrator).
- The **most production-grade** of the survey.
- Routes CI and review feedback loops **back into the owning sessions** — a finished agent's session is resumed with the CI or review result rather than a fresh agent being spawned.
- **Durable-facts + derived-state architecture** with change data capture (CDC).
- Supports **remote worktrees over SSH** and reportedly per-task setup/teardown scripts (worth re-checking `SETUP.md`).
- Single-user, local-first; **no shared team board**.
- **The dealbreaker: a single-repo session model** — session = repo = branch = PR. It cannot express a unit of work spanning several repos.
- Its two moats — the feedback loops and state truthfulness — both have **cheap v1 equivalents** here: escalate-with-resume-command replaces the feedback loop for v1, and the facts+derived schema replaces CDC-backed truthfulness.

### claude_code_agent_farm (Dicklesworthstone)

- MIT.
- **tmux-native swarm** with a work registry, file locks, 2-hour stale-lock detection, and a completed-work log — **proven broker semantics implemented in JSON files.**
- Oriented to **many-agents-one-codebase sweeps**, not per-task isolated environments.

### kodo (ikamensh)

- Overnight autonomy: a cheap orchestrator model directs workers.
- **Independent architect/tester verification** before accepting work — the verification loop is the idea worth copying.

### Archon (coleam00)

- MIT; ~23k stars.
- Now a **YAML workflow engine** mixing deterministic nodes (bash and tests as hard gates) with AI nodes; per-run worktrees; chat-platform adapters.
- Solves the **task layer** — which winter's build skills already occupy — not the **fleet layer**.
- Single-repo.
- Has **pivoted three times** (a stability risk if depended upon).
- **Steal:** deterministic validation gates, the run/session schema, and chat-platform triggering.

### Vibe Kanban

- **Sunsetting — do not build on it.**

### gnap

- A **git-native coordination protocol**; interesting for distributed-queue ideas, though too slow for second-granularity heartbeats (see [../design/runner/store.md](../design/runner/store.md)).

## Curated lists

- andyrewlee/awesome-agent-orchestrators — the running index of the orchestrator landscape this survey walked; re-walk it when the survey is refreshed.
- bradAGI/awesome-cli-coding-agents — the running index of harness CLIs; the watch-list for future adapter candidates beyond Claude Code / OpenCode / Codex.

## Conclusion

Nothing off-the-shelf is polyrepo-native. Blizzard's differentiator is exactly that: a **polyrepo-native agent fleet** is an unfilled niche, and it maps directly onto winter's existing feature-environment isolation. Blizzard itself is a standalone tool, not a winter extension (D-019); at most, the winter *binding* may someday ship as a winter extension.

## The steal-list, condensed

| From | Take |
|------|------|
| Agent Orchestrator | Durable-facts + derived-state schema; the resume-into-owning-session feedback-loop idea. |
| claude_code_agent_farm | Proven file-broker semantics (registry, locks, stale detection, completed log) — reimplemented on sqlite. |
| kodo | Independent architect/tester verification before accepting work. |
| Archon | Deterministic validation gates; run/session schema; chat-platform triggering. |
| gnap | Distributed-queue ideas for the eventual team mode. |
