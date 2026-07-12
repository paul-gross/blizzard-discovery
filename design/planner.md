# The LLM planner (work-shaping)

> **Status: parked.** This document predates the workflow engine (D-025) and still speaks in execution units against a flat queue. Batching's interplay with the graph — whether a batch is one chunk with many PM references, how per-member verdicts work at a node exit — is an [open question](../decisions/open-questions.md); the design below is the input to that reconciliation, not current truth. When it is reconciled, it lands as `design/batching.md` (matching `epic:batching`).

An LLM can shape raw open tasks into a smart batch plan — bundling related work, sequencing conflicts, and choosing models and harnesses. This is feasible and valuable, but it must be quarantined: it is a **separate PLAN phase between PULL and FILL**, and it is **never inside the supervisor's control logic**. The supervisor stays deterministic; the planner is an advisory LLM whose output is validated to death before it is trusted.

## One planning call

Each planning pass makes a single LLM call. The call sees:

- the open tasks — labels, repos, size, dependencies; and
- the current fleet state.

It emits a structured plan of **execution units**, each of the shape:

```
{ type: solo | flurry,
  tasks: [...],
  envs: ...,
  coordinator/worker models: ...,
  harness: ...,
  effort: ...,
  rationale: ... }
```

## The unit is the acquirable thing

The **execution unit** replaces the individual task as the atomic acquirable object. Acquiring a `flurry` unit **atomically acquires all of its member tasks in one transaction** — the batch is all-or-nothing at acquire time, so no other worker can pick off a member mid-batch.

## A deterministic validator gates every plan

The planner can be wrong in judgment but **never in arithmetic**, because a deterministic validator gates every plan it emits. The validator checks:

- every task id exists and is unacquired;
- no task appears in two units;
- unit sizes are within the caps;
- each unit's requested environment count is within the cap the workspace provider can satisfy;
- repo membership is consistent.

If the plan fails validation, it is rejected. The **worst case is the plain 1:1 mapping** you would have had without a planner at all — the planner can only improve on the baseline or be discarded. This is the deterministic-shell / intelligent-core principle applied to batching: the LLM proposes, deterministic code disposes.

## Conflict packing — the key insight

Predicted conflicts are turned into ordering instead of contention. If two tasks touch the same files, the planner puts them **into the same unit, sequenced** — one agent does both, in order. This converts what would have been lock contention between two agents into intra-agent ordering within one agent.

> Every conflict packed into a unit is a lease never contended.

This is why the planner is worth having even beyond raw throughput: it structurally removes contention that the concurrency model would otherwise have to lock around.

## The batch that motivated this

Twenty small bugs across eight repos:

- **Naive platforms** do 1 bug = 1 env = 1 agent — twenty environment setups and twenty Opus sessions.
- **Winter's flurry pattern** does one Opus coordinator plus Sonnet subagents fixing all twenty across roughly four environments.

The cost difference is large, and it is exactly the difference a good planner captures.

## Batch failure semantics

The savings show up in ADVANCE, and so do the costs:

- **Per-task verdicts are non-negotiable.** A batch reports `{12: done, 14: failed-tests, ...}`, one verdict per member task — never a single verdict for the whole unit.
- **Wholesale unit failure explodes the batch.** If a unit fails as a whole, its members are requeued flagged `no-batch` and retried as **solos**. Batching is an optimization; the 1:1 mapping is the always-correct fallback.
- **Tighter stall detection for batches.** A batch unit reports per-member progress in its heartbeat payload, so a batch that silently stalls on one member is caught faster than a solo would be.
- **`blizzard runner ask` from a batch carries the member task id**, so a mid-batch question is routed to the right task.

## Rollout ladder

The planner is introduced cautiously, and every rung is independently useful:

1. **Strict 1:1** (the `milestone:mvp` baseline). No planner. Every task is its own unit. Always correct, no batching.
2. **Heuristic packer.** A deterministic heuristic bundles same-repo, `size:small`-labeled tasks, capped at about 5 per unit. This captures most of the savings with no LLM in the loop.
3. **LLM planner in shadow mode.** The planner runs against the heuristic without driving: log both plans, compare for a week, and only promote the LLM planner to driving once it demonstrably beats the heuristic.

## Routing doctrine

The planner's policy manual is winter-workflow's `choosing-a-build-skill.md`. The doctrine that already decides between `thaw`, `flurry`, `glacier`, `delegate`, and `blizzard` for a human is the same doctrine the planner applies when choosing a unit's type, models, and harness.
