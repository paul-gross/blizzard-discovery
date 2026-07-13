# The environment model

How execution environments are named, allocated, leased, cleaned, and returned — the detail behind D-021. This is the working sketch of the **workspace seam contract**; the [open question](../../decisions/open-questions.md) tracks what is still unsettled.

## The contract, in full

The agreement between a runner and its workspace is keyed on one primitive: the **environment id**, an arbitrary opaque string. The runner never knows what an environment *is* — a worktree, a winter feature environment, a container — only which ids it holds.

| Operation | Semantics |
|-----------|-----------|
| `acquire(chunk_id, count, held_ids)` → `(env id, workdir)` pairs | Allocate `count` environments for a chunk, picking from the provider's pool **minus the held set the runner passes in** (D-062 — the provider keeps no allocation state of its own, see below). Each id comes back with its **working directory** — the one path fact the runner never computes itself; under a lazy binding the workdir may not exist yet (the agent materializes it, D-053/D-063). Returns fewer-than-asked / refusal if capacity is short. |
| `release(env_id)` | Return an environment to the workspace. Never fails; releasing an unknown or already-released id is a no-op. |
| *(implicit)* clean-by-contract | Every acquired environment is **clean**. How is the provider's business — see the bindings below. There is no `reset` verb in the contract; cleanliness is a property of `acquire`. |

Everything above the seam is provider-agnostic: the runner records the chunk→env binding — id and workdir — as a runner-store fact, spawns (or restarts) agents *against an env id*, and reports the route upward. An agent can always be re-associated to its environment by id — restart, resume, and takeover all address `(chunk, env id)`, never a path the runner computed itself; the workdir those operations need is the provider-returned fact read back from the binding.

**The agent, not the runner, populates the environment (D-053).** The node envelope is pure-LLM: artifacts are presented in the prompt — pointer artifacts as their commit hash and feature branch name — and the agent sets up its own workspace from them. Agents manage environments, not the other way around. A provider *may* pre-materialize (check out branches, seed files) as a configurable property of this seam, but the engine never requires it.

**And the runner tells the agent which environments it holds (D-063).** The hub composes the node envelope at PULL and at each transition apply (D-072), but env ids exist only after FILL acquires — so the ids reach the worker as a machine-local **spawn preamble** adjacent to the envelope: `Environments: alpha (/ws/alpha), beta (/ws/beta)` — plus `BLIZZARD_ENV_IDS` in the spawn process environment for hooks and tooling, riding the same injection as `BLIZZARD_LEASE_ID`/`BLIZZARD_SESSION_ID`. The preamble presents each id as a *name grant*: under a pre-materializing binding the agent works in what exists; under a lazy binding it creates the environment itself from the grant plus the presented artifacts.

## Two bindings, worked

**Plain git (the simplest possible binding).** The runner acquires for chunk `x7`; the provider creates a worktree `feature-x7` and returns the id `feature-x7`. Freshly minted **is** clean — the mint is the reset. Capacity is effectively unbounded (disk); `release` deletes the worktree. Idempotency is free: `acquire(x7)` re-returns `feature-x7` if it exists.

**Winter.** The provider fronts an existing workspace with environments `alpha`, `beta`, `gamma`, … — provisioned once by the operator (deps installed, ports allocated, db slices carved; D-019). The pool is the provider's static config; which envs are held it learns from the `held_ids` the runner passes in (D-062). `acquire` picks a free env, **cleans it first** (branch reset, `winter ws init` idempotent re-apply, re-carve per-env db state), and returns the id with its workdir. Capacity is the pool size; when every env is held, `acquire` refuses and the chunk waits. `release` just marks nothing — cleaning happens on the *next* acquire, and the hold itself was never the provider's fact to clear.

The ephemeral-vs-pooled distinction lives entirely below the seam. The runner code is identical against both.

## Allocation truth lives in the runner store (D-062)

The provider is **allocation-stateless**: it keeps no durable table of who holds what. The chunk→env binding fact is already written in the same transaction that mints the lease, and REAP already releases it — so "held" is derivable from facts the runner owns, which is the store-facts-derive-status principle (D-004) applied to allocation. `acquire` receives the held set as an argument and picks from the remainder; *which* id it picks (and, later, affinity) stays below the seam.

What this buys, concretely:

- **One source of truth.** A provider-owned table would be a second store with a real desync window between the provider's write and the runner's — and every BYO binding would have to implement crash-safe storage correctly. Statelessness keeps the hard part in the engine (D-064's exec seam depends on this).
- **Idempotent re-acquire gets simpler.** The runner answers a re-acquire for a chunk that already has a binding from its own facts, without calling the provider at all — the provider sees only genuinely-new allocations. Crash recovery is a re-read, exactly as before, just owned by the store that already guarantees it.
- **The mid-acquire crash is harmless by construction.** Provider cleaned `gamma`, runner died before the binding fact landed: `gamma` stays free per the runner's facts, the next acquire picks whatever it picks and cleans it, and the orphaned clean cost nothing but the clean — reset-on-acquire absorbs it.
- **Native state is substrate, not allocation truth.** A worktree's existence, winter's env-index — providers may consult their own substrate (plain git's idempotency *is* "the worktree exists"), but drift reconciles one way, runner → provider: anything that looks held below the seam with no binding fact above it gets released.

## Packaging: a capability slot with an exec seam (D-064)

The provider binds like winter's own `[capabilities]` slots: `workspace = "plain-git" | "winter" | <path to executable>` in the runner's config, one binding per runner (D-019). The two reference bindings compile into the one `blizzard` binary (D-061 — no second artifact, no version skew), and a **bring-your-own provider is an invoked executable** behind a versioned exec protocol: the contract's verbs as argv (`<provider> acquire --chunk x7 --count 1 --held alpha,beta`), results as JSON on stdout. Arbitrary code, deliberately — the contract is two verbs and the engine holds the durable truth, so a BYO provider can be a fifty-line script. The exact wire shape is an open edge below.

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

- **Capacity signaling.** Refuse-on-acquire is the minimum; a queryable pool size would let FILL plan a tick instead of probing (with D-062 the runner knows the held count already — pool size is the missing half of free-count). Undecided whether the contract requires it.
- **Exec wire format.** D-064 fixes the shape (argv verbs, JSON out, versioned); the exact schema — field names, error/refusal encoding, protocol version negotiation — is unshaped.
- **Environment affinity.** Ids are not perfectly fungible — an env whose deps are warm for repo X is cheaper for repo-X work. A later packer may prefer affine envs; the contract may need an optional hint.
- **Grow-on-demand.** The winter binding could mint new environments (`winter ws init zeta`) when the pool is dry, accepting provisioning cost at growth time. Deliberately out of the MVP: operator-provisioned capacity only.
