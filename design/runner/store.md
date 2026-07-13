# The runner store

The runner's embedded database: a sqlite (WAL) database **inside `blizzard-runner`**, which is its only reader and writer (D-023). It is an implementation detail of the runner, not a component anyone talks to — worker hooks and humans reach the runner's local API through the `blizzard` CLI (route table in [api.md](./api.md)), and the daemon reaches the store in-process, so the schema stays private.

The store holds exactly the machine-local fast path in the [division of truth](../architecture.md#division-of-truth): leases, heartbeats, epochs, pids, and env bindings. Everything fleet-visible lives at the hub (D-022/D-024); lease facts are both — minted and leased here, reported to the hub as the fence's input (D-044).

## Why sqlite

The store needs atomic, durable, queryable writes behind a single daemon — sqlite gives all three for free:

- **Atomic, crash-safe writes.** A lease is a single `UPDATE ... WHERE unleased` statement, so it either happens or does not — no read-modify-write window, no half-written row a crash-restart must reconcile. The atomicity buys *self-consistency and idempotency*, not race arbitration: the runner is the sole writer on its machine (D-023), so there is no rival to beat. Cross-machine exactly-once is arbitrated by the hub's acquisition (D-024/D-049), not here — see the alternatives table below.
- **Crash durability.** WAL mode gives durable writes that survive a hard crash.
- **Queryability.** `blizzard runner status`'s machine-local view is a `SELECT`. State is inspectable with ordinary SQL, which makes the derived-status model practical.

sqlite runs in WAL mode. It is more than sufficient behind a single owning daemon — which is exactly how it is deployed on both sides (D-023): embedded in the runner for the fast path, embedded in `blizzard-hub` for the fleet facts (see [the hub store](../hub/store.md)).

### Alternatives considered

| Option | Why not |
|--------|---------|
| Flat JSON + `flock` (as in claude_code_agent_farm) | You hand-roll atomicity. sqlite already provides ACID CAS; there is no reason to reimplement it on top of a lock file. |
| Redis / Postgres | A separate database server is overkill: the runner daemon already exists, and sqlite embeds in it with no extra process to operate. |
| Git-native (gnap-style) | Too slow for second-granularity heartbeats — a git commit per heartbeat does not scale to the tick rate. |
| GitHub issue assignment as the lease | GitHub's rate limits do not fit second-granularity heartbeats — and issue assignment is not the lease at any level: cross-machine arbitration is the hub's queue (D-024); assignment is at most a reflection of hub state. |

## What it records

Per the fleet-wide [facts-only principle](../architecture.md#store-facts-derive-status), the store records only things that definitely happened at a definite time:

- lease created
- environment binding (chunk → env ids, from the workspace provider)
- pid + process start time — recorded by the spawn wrapper from inside the child, before the harness runs; the adapter-reported session id lands at spawn-return (D-092)
- last heartbeat
- last tool activity
- verdict
- open asks — recorded before the asking worker exits ([ask-answer.md](../ask-answer.md)); the fact that distinguishes a parked chunk from a dead one
- escalations — recorded before the resume command is printed, so no supervisor races the human
- the outbound buffer — every hub-bound fact, always (see below)

A chunk's *status* — running, stalled, waiting-on-human, done ([domain/events.md](../domain/events.md) is the canonical vocabulary, D-067) — is never a stored column; it is derived by query. This is what makes the runner's crash recovery correct rather than aspirational (see [loop.md](./loop.md)).

## The outbound buffer (store-and-forward, D-069)

Runner→hub is **store-and-forward always**: every hub-bound fact — transitions, decisions, questions, and the traveling runner-minted facts ([domain/events.md](../domain/events.md)) — is written to the outbound buffer at mint, stamped with a **per-runner monotonic sequence number**, even when the hub is reachable. A single flusher drains the buffer in FIFO order (D-044's ordering made structural — the buffer is the only path, so a lease fact always precedes the transitions minted under it), calling the appropriate hub route per fact kind with the `(runner_id, seq)` idempotency key. The hub keeps a per-runner **high-water mark**: a fact with seq ≤ mark is already-applied and acked without re-applying, and a *semantic* rejection — a stale-epoch transition — still advances the mark, because rejection is an outcome, not a delivery failure. The result is one code path whether the hub is up or not: an outage is just a bigger backlog, and replay-after-outage is the normal drain. Gaps in the sequence are detectable on both sides. One flush is special: a transition submission's apply-response carries the chunk's next node envelope (D-072), which is what ADVANCE waits on before continuing in place — so during an outage a chunk waits at its node boundary while everything else keeps flushing when the hub returns. A lost apply-response cannot wedge that boundary: a retried submission acked as already-applied sends the runner to the idempotent envelope read (`GET /chunks/{id}/envelope`, D-090).

## Epochs (fencing tokens)

Each lease carries an incrementing **epoch**, minted here and reported to the hub with the lease fact (D-044). Transition submission carries it, and the hub rejects anything stale (D-007/D-036). A reaped-but-still-running agent therefore cannot clobber its successor's work: the successor leased with a newer epoch, and the zombie's submission is fenced out. See [concurrency-model.md](../concurrency-model.md) for the general fencing model.
