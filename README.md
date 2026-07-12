# blizzard-discovery

Codename **blizzard**: an interoperable orchestration platform for autonomous fleets of coding agents.

Blizzard runs a fleet of coding agents continuously on local machines: work flows in from a backlog, each task executes unattended in an isolated environment, finished work lands — merged to main by default, parked at a human gate where one is configured — and a human is interrupted only when an agent is genuinely blocked. The platform's core is **interoperability** — the workspace framework, the forge, the project-management system, and the coding harness are all pluggable seams, not assumptions (see the [mission](./product/mission.md)). The **reference stack** for the first implementation binds those seams to [winter](https://github.com/paul-gross/winter) feature environments (polyrepo-native isolation), GitHub issues and pull requests, and Claude Code; the architecture documents describe that stack concretely. Blizzard itself contributes the fleet layer: a work-orchestrating hub — colocated with the runner or remote — that wraps the project-management system and runs a dynamic workflow engine, a reconciliation supervisor that advances work through that engine's graphs, harness adapters, and remote visibility with phone-based control.

## Status: DISCOVERY PHASE

**This repository contains no code.** It is a discovery, research, and planning corpus. It exists so that the design can be iterated on in writing, and cross-checked cold by other AI harnesses (Codex, OpenCode) and by human reviewers, before any implementation begins.

Everything here is a proposal. Settled choices live in the [decision log](./decisions/log.md) with their rationale; everything still soft lives in [open questions](./decisions/open-questions.md) — so a fresh reader can evaluate and challenge the design rather than inherit it as settled.

## How this repository is used

- **Ideation.** Capture and refine the requirements, design, and roadmap for the blizzard fleet.
- **Cross-checking.** Hand the corpus to other AI harnesses and reviewers to stress-test the design from a cold start — no shared session history, only what is written down.
- **Convergence.** Drive the open questions toward decisions (graduating them into the decision log), then graduate the design into an implementation plan.

## Reading order

For a cold, front-to-back read: **product** (mission → personas → mvp → vision → roadmap; epics and the story map are reference) → **design** (architecture → workflow-engine → concurrency-model → runner → harness-adapters → hub → ask-answer → planner) → **research** → **decisions**. The [glossary](./glossary.md) is a companion throughout.

## The corpus

The four layers have different lifecycles: product documents should barely change once settled, design documents churn throughout discovery, research is a point-in-time record, and decisions only grow.

### `product/` — what and why

| Document | What it answers |
|----------|-----------------|
| [product/mission.md](./product/mission.md) | The scope filter: what blizzard is for, the success criteria, and the explicit non-goals. |
| [product/personas.md](./product/personas.md) | Who blizzard serves — `persona:senior-se`, `persona:application-architect`, `persona:product-manager`, `persona:product-owner`, `persona:harness-engineer`, and the non-human `persona:fleet-agent` — and which parts of the design exist for each. |
| [product/mvp.md](./product/mvp.md) | What *done* means for `milestone:mvp`: the acceptance journey, testable criteria, and the deliberate cut list. |
| [product/epics.md](./product/epics.md) | The `epic:<slug>` registry — the canonical feature-epic ids that scope statements (MVP, roadmap, decisions) cite. |
| [product/storymap.md](./product/storymap.md) | The story map: one box per epic, arrows pointing to the next work in release order. |
| [product/vision.md](./product/vision.md) | The narrative: the problem, why winter is the key workspace binding, the deterministic-shell / intelligent-core philosophy, why nothing off-the-shelf fits. |
| [product/roadmap.md](./product/roadmap.md) | The named-milestone rollout (`milestone:mvp` → `milestone:centralized-hub`) and the unscheduled Later pool. |

### `design/` — how

| Document | What it answers |
|----------|-----------------|
| [design/architecture.md](./design/architecture.md) | The map: every running piece (hub, runner, workers, CLI, board), where each runs, how they interact, the division of truth, and the facts-only principle — with diagrams. Start here before any deep dive. |
| [design/workflow-engine.md](./design/workflow-engine.md) | The dynamic workflow engine: chunks, the hub-defined YAML node graphs, judgements, artifacts (pointers + assets), delivery-as-a-node, and the default graph. |
| [design/concurrency-model.md](./design/concurrency-model.md) | The three distinct concurrency problems, leases and fencing, the hub's merge queue, and structural contention reduction. |
| [design/runner/](./design/runner/index.md) | The runner component: the reconciliation loop, the embedded store (sqlite rationale, epochs), the environment model (opaque env ids, reset-on-acquire, the three capacities), and the tracked contracts for every API the runner speaks. |
| [design/harness-adapters.md](./design/harness-adapters.md) | The adapter interface, the per-harness comparison, the enforcement wrapper, hooks, and human-takeover flows. |
| [design/domain/](./design/domain/index.md) | The app-wide domain models — chunk, transition, migration, artifact, question/answer, route, runner, graph — with per-model property tables and the derived status enum. |
| [design/hub/](./design/hub/index.md) | The hub component: the canonical responsibility list (PM wrapper, workflow record, artifacts, merge queue, registry), the colocated/remote topology, outbound-only runners, the board and chat clients, the hub store, and the API route table. |
| [design/ask-answer.md](./design/ask-answer.md) | The ask/answer protocol: ask-and-exit, parking, fan-out, first-write-wins answers, resume-with-answer. |
| [design/planner.md](./design/planner.md) | The LLM work-shaping planner (parked pending the workflow-graph interplay): execution units, the deterministic validator, conflict packing, batch failure semantics, and the rollout ladder. |

### `research/` — what we learned from others

| Document | What it answers |
|----------|-----------------|
| [research/ecosystem.md](./research/ecosystem.md) | The mid-2026 ecosystem survey — what exists, what to steal, and why blizzard is still worth building. |
| [research/hard-problems.md](./research/hard-problems.md) | The four hard problems surfaced by studying Agent Orchestrator, and blizzard's answers. |

### `decisions/` — what is settled vs. still soft

| Document | What it answers |
|----------|-----------------|
| [decisions/log.md](./decisions/log.md) | The settled decisions, each with its rationale — considered-and-rejected made distinguishable from never-considered. |
| [decisions/open-questions.md](./decisions/open-questions.md) | The open questions still to be resolved; each resolution graduates into the log. |

## Guiding philosophy

**Deterministic shell, intelligent core** — all loop, arbitration, and recovery logic is boring deterministic code; LLMs supply judgment only at well-defined points. The full treatment is in [product/vision.md](./product/vision.md).
