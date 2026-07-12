# The environment model

How execution environments are named, allocated, leased, cleaned, and returned — the detail behind D-021. This is the working sketch of the **workspace seam contract**; the [open question](../../decisions/open-questions.md) tracks what is still unsettled.

## The contract, in full

The agreement between a runner and its workspace is keyed on one primitive: the **environment id**, an arbitrary opaque string. The runner never knows what an environment *is* — a worktree, a winter feature environment, a container — only which ids it holds.

| Operation | Semantics |
|-----------|-----------|
| `acquire(chunk_id, count)` → env ids | Allocate `count` environments for a chunk. **Idempotent per chunk id**: re-asking for the same chunk returns the same ids (this is what makes crash recovery re-read instead of double-allocate). Returns fewer-than-asked / refusal if capacity is short. |
| `release(env_id)` | Return an environment to the workspace. Never fails; releasing an unknown or already-released id is a no-op. |
| *(implicit)* clean-by-contract | Every acquired environment is **clean**. How is the provider's business — see the bindings below. There is no `reset` verb in the contract; cleanliness is a property of `acquire`. |

Everything above the seam is provider-agnostic: the runner records the chunk→env binding as a runner-store fact, spawns (or restarts) agents *against an env id*, and reports the route upward. An agent can always be re-associated to its environment by id — restart, resume, and takeover all address `(chunk, env id)`, never a path the runner computed itself.

**The agent, not the runner, populates the environment (D-053).** The node envelope is pure-LLM: artifacts are presented in the prompt — pointer artifacts as their commit hash and feature branch name — and the agent sets up its own workspace from them. Agents manage environments, not the other way around. A provider *may* pre-materialize (check out branches, seed files) as a configurable property of this seam, but the engine never requires it.

## Two bindings, worked

**Plain git (the simplest possible binding).** The runner acquires for chunk `x7`; the provider creates a worktree `feature-x7` and returns the id `feature-x7`. Freshly minted **is** clean — the mint is the reset. Capacity is effectively unbounded (disk); `release` deletes the worktree. Idempotency is free: `acquire(x7)` re-returns `feature-x7` if it exists.

**Winter.** The provider fronts an existing workspace with environments `alpha`, `beta`, `gamma`, … — provisioned once by the operator (deps installed, ports allocated, db slices carved; D-019). The provider keeps its own allocation table: it knows alpha/beta/gamma are free and delta/epsilon are held. `acquire` picks a free env, **cleans it first** (branch reset, `winter ws init` idempotent re-apply, re-carve per-env db state), records it against the chunk, and returns the id. Capacity is the pool size; when every env is held, `acquire` refuses and the chunk waits. `release` just marks the slot free — cleaning happens on the *next* acquire.

The ephemeral-vs-pooled distinction lives entirely below the seam. The runner code is identical against both.

## Why reset-on-acquire, not reset-on-release

A chunk that crashes mid-work leaves its environments mid-surgery: half-finished branches, uncommitted files, dirty db rows, orphaned services. Reset-on-release can be **skipped by exactly that crash** — the cleanup step dies with the process that owed it. Reset-on-acquire cannot be skipped: it runs at the start of the next lease, it is idempotent, and it erases whatever corpse the previous holder left, however it died. Cleaning is therefore always on the acquiring side of the boundary, which is also why the contract states cleanliness as a property of `acquire` rather than as a verb.

## Environments ride the node-step lease

An environment binding is not a new mechanism — it rides the chunk's existing lease:

- The binding `(chunk → env ids)` is a **runner-store fact**, written in the same transaction that mints the chunk's lease.
- The binding's lifetime is the lease's lifetime: the lease's heartbeat and epoch cover its environments. There is no separate env heartbeat.
- **REAP frees environments** the same way it frees chunks: when a lease is reaped, its bindings are released back to the provider, and the (dirty) environments are safe precisely because of reset-on-acquire.
- **All-or-nothing acquisition.** A chunk needing 3 envs when only 2 are free does not take 2 and wait — it doesn't lease at all (FILL releases the lease and skips it that tick). No partial holds means no hold-and-wait, which means **no deadlock** between agents competing for environments, structurally.

One consequence worth stating: because only `blizzard-runner` drives environment lifecycle (acquire, release — one loop, one tick at a time), env operations are naturally serialized. Under the winter binding, the shared git object store never sees two concurrent worktree surgeries from the fleet.

## Three capacities, independently exhaustible

The runner arbitrates three resources, and they bind independently:

| Capacity | Limit | Exhausted looks like |
|----------|-------|----------------------|
| **Work** | what PULL acquired | fleet idle, everything else free |
| **Agent slots** | `MAX_AGENTS` (e.g. 5) | agent-bound: 5 solos running, envs idle |
| **Environment slots** | the provider's capacity (e.g. 10) | env-bound: 2 flurries holding 8 envs, agent slots idle |

Example, one machine: `MAX_AGENTS = 5`, winter pool of 10. Three solo chunks (3 agents, 3 envs) plus one 10-item batch chunk (1 agent, 4 envs — batching is parked, but the capacity math is the point) = 4 agents and 7 envs in use; the runner can still slot one more small chunk. Whether the fleet tends agent-bound or env-bound is a tuning fact `persona:harness-engineer` reads off real runs — both limits are `epic:config` constants, and both utilizations belong on the board.

The env count a chunk *wants* is set by whoever shaped it — 1 for a solo, K for a batch (batching parked; see open questions). The 1:1 fallback is just the degenerate case: 1 chunk, 1 agent, 1 env.

## Open edges

Tracked in [open questions](../../decisions/open-questions.md):

- **Capacity signaling.** Refuse-on-acquire is the minimum; a queryable free-count (the winter link *knows* alpha/beta/gamma are free) would let FILL plan a tick instead of probing. Undecided which the contract requires.
- **Environment affinity.** Ids are not perfectly fungible — an env whose deps are warm for repo X is cheaper for repo-X work. A later packer may prefer affine envs; the contract may need an optional hint.
- **Grow-on-demand.** The winter binding could mint new environments (`winter ws init zeta`) when the pool is dry, accepting provisioning cost at growth time. Deliberately out of the MVP: operator-provisioned capacity only.
