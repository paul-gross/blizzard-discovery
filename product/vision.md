# Vision

## Problem statement

We want an autonomous development fleet: one or more coding agents — Claude Code today, OpenCode and Codex later — running continuously on local machines, pulling work from a shared queue, executing each unit of work in an isolated environment, delivering results — merged to the main branch in the baseline, parked at a human-review gate where one is configured — and escalating to a human only when blocked.

The fleet should keep working overnight and unattended. It should recover cleanly from crashes and reboots. It should let a human step in with full context when an agent gets stuck, and get out of the way otherwise. It should scale from a single developer running a handful of agents on one machine to a small team coordinating across several machines.

## Why winter is the key

Blizzard itself is workspace-agnostic. The workspace is a seam, like the forge and the harness ([mission](./mission.md)); the orchestrator never binds itself to a specific workspace, workflow, or project-management system. But an orchestrator is only as capable as the work chunks its agents can safely execute — and that is why winter is the key binding.

Winter is the best workspace solution for making a single agent powerful. A winter feature environment composes one git worktree per project repository, all on a shared branch, with per-environment ports and per-environment service orchestration. That is the implementation behind the flexible work chunk: it turns "1 agent" into **1 agent → many tasks → many environments → many repositories** — one agent takes a chunk of one or more tasks, fans out subagents across one or more feature environments, and touches one or more repos, with isolation guaranteed at every level. Two agents on different chunks never share a working tree, never collide on ports, never step on each other's running services — so there is nothing to lock and nothing to serialize.

The rest of the ecosystem cannot express this. Off-the-shelf orchestrators universally assume **single-repo git-worktree isolation** — one repository, one worktree, one branch, one agent: the rigid 1:1:1:1 shape. Winter is what makes the many:many:many chunk executable at all. That makes it the reference workspace binding — not a foundation blizzard is welded to, but the workspace that gives blizzard's agents the most power.

## Deterministic shell, intelligent core

The single most important design principle runs through every document in this corpus:

> All loop, arbitration, and recovery logic is boring deterministic code. LLMs supply judgment only at well-defined points.

The control plane — the queue, the acquisition and lease protocol, the reconciliation loop, the fencing logic, the crash recovery — is ordinary deterministic software. It is testable, it is `kill -9` safe, and it never depends on an LLM behaving correctly.

LLMs are invoked only where judgment is genuinely required:

- **Doing the work** — the agent that actually writes the code in a feature environment.
- **Reviewing / judging the work** — the agent reviewers and e2e testers that verify a chunk before it advances (the review node of the workflow graph; human gates optional per the [mission](./mission.md)).
- **Planning batches** — an optional planner that shapes open tasks into execution units (see [design/planner.md](../design/planner.md)).
- **Rendering verdicts** — the structured success/failure output a finished worker emits.

Everywhere else, the system is deterministic by construction. An LLM can be wrong in judgment, but it is never handed a lever that lets it be wrong in arithmetic.

## Why not off-the-shelf (summary)

The ecosystem was surveyed in mid-2026; [research/ecosystem.md](../research/ecosystem.md) carries the full survey and the steal-list. The short version:

- **Nothing off-the-shelf is polyrepo-native.** Every mature orchestrator models a session as `repo = branch = PR`, which cannot express a unit of work that spans several repos.
- **The best projects are worth stealing from, not adopting.** Agent Orchestrator contributes its durable-facts / derived-state architecture and its feedback-loop idea; claude_code_agent_farm contributes proven file-based broker semantics; kodo contributes independent verification; Archon contributes deterministic validation gates and chat triggering. None of them fills the polyrepo niche.
- **The niche is unfilled.** A workspace-agnostic orchestrator with flexible work shapes — even mere polyrepo support — is an open space. Winter's strengths make it the reference workspace to prove it on. Blizzard itself is a standalone tool (D-019); at most, the winter *binding* might someday ship as a winter extension.
