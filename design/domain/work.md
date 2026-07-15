# Work models

The work lifecycle: the chunk at the center, its derived statuses, the transition record it moves by, and the migration fact that re-pins it across graphs. Part of the [domain model](./index.md).

## Status enum

The derived chunk statuses (D-067). Per D-004 none of these is ever a stored column — each is the result of a query over recorded facts. The canonical fact vocabulary and the derivation queries, including their precedence order, live in [events.md](./events.md); this table is the summary.

| Status | Derived from |
|--------|--------------|
| `not_ready` | Ingested but not yet promoted (D-103) — a `chunk.promoted` fact is absent and no higher-precedence fact matches. The resting state ingest mints by default: visible on the board but never claimed by a runner. `POST /chunks/{id}/promote` appends the fact that flips it to `ready`. |
| `ready` | Ingested by id into a chunk (D-024/D-047) **and promoted** (D-103), no live route — never routed, or released back by detach (D-088) — sitting in the hub's ready queue. |
| `running` | A live route exists — `route.created` with no later `route.released` (D-021/D-088) — and no higher-precedence fact matches. |
| `delivering` | The newest accepted transition entered a hub node (D-030) — queued, merging, or awaiting an external merge (D-065); the runner holds environments throughout (D-066). |
| `waiting_on_human` | An open ask — a question row with no answer row ([ask-answer.md](../ask-answer.md)) — or an open decision — a gate's decision row no transition references (D-045); the reap clock is stopped. The answer row or the resolution flips it back. |
| `needs_human` | An open escalation — retries exhausted or a worker died without a verdict past the retry cap (D-009) — with no later lease fact: requeue or takeover closes it by supersession, never a resolution fact (D-067). |
| `stopped` | An operator stop fact (D-067) — terminal abandonment at any point after acquisition, artifacts and history retained. |
| `done` | The deliver node landed the chunk's git-commit artifacts through the merge queue (D-030), or its PR reached a terminal state externally (D-065). |

## Chunk

The unit of work that travels the workflow graph (D-024, D-025). The center of the model: everything else hangs off it. It is also **ephemeral** (D-047): the PM item is the durable referent, so any unacquired chunk may be discarded, and re-ingesting the same pointer mints a fresh chunk.

| Property | Notes |
|----------|-------|
| `chunk_id` | Hub-minted prefixed ULID — `ch_<ulid>` (D-075): type-evident, lexically creation-ordered, the convention every hub entity id follows. |
| PM pointers | An array of `{provider, url}` pointers (D-075), one per wrapped PM item (D-047) — a GitHub issue in the reference binding; ingest mints one chunk per item, plural pointers arrive by manual grouping in the web app (D-048) and later automated batching (open). A pointer already held by a live chunk cannot be re-ingested (D-093). Contents are never stored — read pass-through via the hub. |
| `graph_id` | The immutable graph pinned at mint — the hub-configured default graph at ingest (D-033/D-040/D-081) — travels with the chunk to completion; changes only when a migration record lands. |
| current node | **Derived** from the transition record (D-027), never stored — always a node of the pinned graph. |
| status | **Derived** — see the [status enum](#status-enum) above. |
| transitions | Append-only record reported by the holding runner (D-027) — see Transition. |
| latest epoch | **Derived** from the chunk's newest lease fact — one lease per node-step attempt (D-035) — the fence every state-advancing write is checked against (D-007). |
| artifacts | An append-only versioned key-value store, `{node}.{artifact-name}.{epoch} → artifact_id` (D-036) — see [artifacts.md](./artifacts.md). |
| questions | The chunk's asks, open and answered — see [questions.md](./questions.md). |
| route | Where it is being worked — see [fleet.md](./fleet.md). |

## Transition

One entry in a chunk's append-only movement record (D-027): the fact that a judgement was made at a node's exit and the chunk moved along an edge. The chunk's current node is derived from its newest **accepted** transition — which is why transitions are fenced: a zombie processor reporting a transition under a stale epoch must be rejected, or the record would advance a chunk on a dead leaseholder's say-so (D-007).

The transition is also the **artifact commit** (D-036): a node-step's transition and its artifacts are submitted as one atomic, epoch-fenced write — a rejected transition's artifacts never enter the chunk's store. At a human gate there is **no transition until the human decides**: the node-step's completion lands as an open **Decision** (below, D-045), and the resolving choice writes the ordinary transition referencing it. Every transition is therefore fully formed — its judgement always exists, because the judgement is what selects the edge.

| Property | Notes |
|----------|-------|
| `chunk_id` | The chunk that moved. |
| `from_node_id` / `to_node_id` | Exact node references ([graph.md](./graph.md)) — both nodes of the chunk's pinned graph (D-033), so a transition always resolves against a graph that still exists. |
| judgement | What selected the edge: the worker's verdict plus the node's deterministic check results (D-025), or the human's chosen choice resolving a Decision at a gate (D-045); a missing verdict is a failure (D-009). |
| `decision_id` | Gates only (D-045): the Decision this transition resolves. |
| artifacts | The artifact entries committed atomically with this transition (D-036) — see [artifacts.md](./artifacts.md). Empty on a gate-resolving transition: the artifacts already landed with the Decision. |
| `epoch` | The executing lease's fencing epoch — one lease per node-step attempt (D-035) — checked against the chunk's latest epoch; stale transitions are rejected, not recorded (D-007). |
| `runner_id` | The reporting author: the holding runner (D-027) — or the hub's coordinator for a hub-node exit (D-079), which mints those steps' leases and epochs itself. |
| `recorded_at` | When the record landed at the hub. |

A migration between graphs is **never a transition** — see Migration.

## Decision

A human decision to be made (D-045) — the gate's parking row, kin to a question ([questions.md](./questions.md)): a durable multiple-choice ask whose resolution moves the chunk. Written by the runner as the node-step's completion — one atomic, epoch-fenced write carrying the step's artifacts, exactly where a worker-judged node would have written its transition (D-036 atomicity intact). **The runner authors both outcomes and the hub validates which is legal**: at a human-judged node a submitted transition is rejected (human signoff required), and a runner may submit a decision in place of a transition for *any* node — that is the runner-config gate (D-032), human checks added to an existing workflow by configuration alone. The chunk derives `waiting_on_human` from an unresolved decision and holds no live lease while parked (D-035); **pending is derived** — a decision no transition references (the D-037 pattern). Resolution — a person picking one of the choices (the board's and chat bot's buttons, D-042) — is recorded at the hub first-write-wins like an answer; the holding runner picks it up over its outbound pull and records the resolving Transition above (D-027 intact: the runner advances the chunk).

| Property | Notes |
|----------|-------|
| `decision_id` | What the resolving transition's `decision_id` points back at. |
| `chunk_id` | The parked chunk. |
| `node_id` | The gate node awaiting the decision — its judgement spec supplies the choices ([graph.md](./graph.md)). |
| choices | The selectable outcomes (D-042), rendered as buttons. Later extensible with override options — route to an arbitrary node past the graph's edges, or stop the chunk — without touching the transition model (D-045). |
| artifacts | The step's artifacts, committed atomically with the decision (D-036) — visible to the deciding human. |
| `epoch` | The submitting lease's fencing epoch — stale decisions are rejected like stale transitions (D-007). |
| `submitted_at` / resolution | When it parked; the resolving choice + resolver + time live on the resolving transition, and resolved-ness derives from that reference. |

## Migration request

The **intent** to move a chunk between graphs (D-037) — never the record of it happening; that is the Migration record below. A chunk's pending targeted migration is **derived**: a request with no migration record referencing it (D-004). Requests append, and the **newest unapplied request wins**: a chunk whose target changes twice before it executes again simply has its pending intent superseded — one migration applies, one record is written, and superseded requests never produce a record. (Auto-migration needs no update at all: the derived pending always follows the pinned graph's current `migration_target` pointer, one hop per transition — D-040.)

| Property | Notes |
|----------|-------|
| `request_id` | What the eventual migration record's `caused_by` points back at. |
| `chunk_id` | The chunk to re-pin. |
| `to_graph_id` | The target graph. |
| mode | `deferred` or `forced` (D-034). Deferred: applied by the hub at the chunk's next transition ingestion, atomically with the transition (D-072); the runner discovers the swap in the apply-response's envelope. Forced: apply immediately — the hotfix path. |
| `requested_at` / `triggered_by` | When and by whom. |

Standing intent is **not** a request: "always drift onward" lives on each graph as its `auto_migrate` policy plus `migration_target` pointer ([graph.md](./graph.md), D-040), and a chunk's pending auto-migration is derived — its pinned graph has `auto_migrate: deferred` and an **enabled** (D-039) `migration_target`. No per-chunk rows are fanned out when a new graph is published. A parked chunk never transitions, so it never auto-migrates until resumed — "next convenient time" by construction.

## Migration record

The fact that a chunk **did** move between graphs (D-033/D-034/D-037) — written only at apply time, immutable, and its own record, never a transition: a transition moves along an edge *within* one pinned graph, while a migration re-pins the chunk across graphs, where no edge exists (D-040). The chunk's pinned `graph_id` derives from its newest migration record (its mint fact when none exists).

| Property | Notes |
|----------|-------|
| `chunk_id` | The chunk that was re-pinned. |
| `caused_by` | A `request_id`, or the pinned graph's `auto_migrate` policy (D-037/D-040). |
| `from_graph_id` / `to_graph_id` | The graphs on either side; both immutable (D-033). |
| `from_node_id` | The node (in `from_graph_id`) the chunk sat at when the migration applied. |
| `to_node_id` | The node (in `to_graph_id`) the chunk re-pinned to, resolved by **name-match** against `from_node_id`'s name (D-034); when no name matches, the target graph's entry node (D-051). |
| `epoch` (forced only) | The new fencing epoch minted by lease revocation — the existing epoch check (D-007) then rejects the abandoned in-flight work, which is redone under the new graph. |
| `applied_at` | When it applied. |
