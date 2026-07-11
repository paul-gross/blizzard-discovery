# The runner store

The runner's embedded database: a sqlite (WAL) database **inside `blizzard-runner`**, which is its only reader and writer (D-023). It is an implementation detail of the runner, not a component anyone talks to — worker hooks and humans reach the runner's local API through the `blizzard` CLI (verb table in [contracts.md](./contracts.md)), and the daemon reaches the store in-process, so the schema stays private.

The store holds exactly the machine-local fast path in the [division of truth](../architecture.md#division-of-truth): claims, leases, heartbeats, epochs, pids, and env bindings. Everything fleet-visible lives at the hub (D-022/D-024); claim facts are both — minted and leased here, reported to the hub as the fence's input (D-044).

## Why sqlite

The claim protocol needs an atomic compare-and-set, and sqlite gives it for free:

- **Atomic claims.** A claim is a single `UPDATE ... WHERE unclaimed` statement. ACID CAS comes out of the box — no hand-rolled locking, no read-modify-write window.
- **Crash durability.** WAL mode gives durable writes that survive a hard crash.
- **Queryability.** `blizzard status`'s machine-local view is a `SELECT`. State is inspectable with ordinary SQL, which makes the derived-status model practical.

sqlite runs in WAL mode. It is more than sufficient behind a single owning daemon — which is exactly how it is deployed on both sides (D-023): embedded in the runner for the fast path, embedded in `blizzard-hub` for the fleet facts (see [the hub store](../hub/store.md)).

### Alternatives considered

| Option | Why not |
|--------|---------|
| Flat JSON + `flock` (as in claude_code_agent_farm) | You hand-roll atomicity. sqlite already provides ACID CAS; there is no reason to reimplement it on top of a lock file. |
| Redis / Postgres | A separate database server is overkill: the runner daemon already exists, and sqlite embeds in it with no extra process to operate. |
| Git-native (gnap-style) | Too slow for second-granularity heartbeats — a git commit per heartbeat does not scale to the tick rate. |
| GitHub issue assignment as the claim | GitHub's rate limits do not fit second-granularity heartbeats — and issue assignment is not the claim at any level: cross-machine arbitration is the hub's queue (D-024); assignment is at most a reflection of hub state. |

## What it records

Per the fleet-wide [facts-only principle](../architecture.md#store-facts-derive-status), the store records only things that definitely happened at a definite time:

- claim created
- environment binding (chunk → env ids, from the workspace provider)
- pid + process start time
- last heartbeat
- last tool activity
- verdict
- open asks — recorded before the asking worker exits ([ask-answer.md](../ask-answer.md)); the fact that distinguishes a parked chunk from a dead one
- escalations — recorded before the resume command is printed, so no supervisor races the human
- the outbound buffer — facts awaiting flush to an unreachable hub (transitions, questions, events)

A chunk's *status* — running, stalled, waiting-on-human, done, failed — is never a stored column; it is derived by query. This is what makes the runner's crash recovery correct rather than aspirational (see [loop.md](./loop.md)).

## Epochs (fencing tokens)

Each claim carries an incrementing **epoch**, minted here and reported to the hub with the claim fact (D-044). Transition submission carries it, and the hub rejects anything stale (D-007/D-036). A reaped-but-still-running agent therefore cannot clobber its successor's work: the successor claimed with a newer epoch, and the zombie's submission is fenced out. See [concurrency-model.md](../concurrency-model.md) for the general fencing model.
