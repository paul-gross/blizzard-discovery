# The runner loop

The supervisor is a deterministic reconciliation loop — a few hundred lines — run under systemd as `blizzard-runner`. Its loop holds **no state of its own**: every fact lives in the daemon's embedded [store](./store.md) (D-023/D-028). This is what makes it `kill -9` safe. Killing the supervisor loses nothing, and startup recovery is simply the reap step running first.

The runner is machine-level and **operator-directed**: it *acquires* work from the hub (D-024), it is never pushed work. The operator can pause it, start it, and choose which work it slots — filtering or pinning what FILL may lease. Pausing stops new leases; in-flight chunks run to completion. Pause has two surfaces — the fleet's brake and the runner's own, either of which stops new leases (D-105, [domain/fleet.md](../domain/fleet.md)).

## No state of its own

Because every fact lives in the runner store and every status is derived, the supervisor can be killed at any instant and restarted with no special recovery logic. Its startup pass is its normal REAP step: expire whatever leases went stale while it was down, then continue. There is no in-memory queue to rebuild, no checkpoint to reconcile.

## The loop

Each tick — roughly every 30 seconds — the supervisor runs these steps in order: **REAP → PULL → FILL → ADVANCE**. Every step is idempotent, so a crash mid-tick followed by a restart re-runs the tick harmlessly.

(When the batching planner re-enters the design, its PLAN phase slots between PULL and FILL — parked on the batching × workflow-graph [open question](../../decisions/open-questions.md).)

### REAP

Expire leases whose holder is gone.

- A lease is dead if its heartbeat is stale **or** its pid is dead.
- Check **pid AND recorded process start time** together. A bare pid check is unsafe because the OS reuses pids — a different process may now hold the old pid. Comparing the recorded process start time against the live process's start time survives pid reuse.
- A **stalled-but-alive** worker is caught here too: heartbeats are emitted by tool use, so a worker that stops progressing stops heartbeating, its lease goes stale, and it is reaped like any dead one — there is no separate stall detector. When the reaped lease's process is somehow still alive, REAP also kills it — best-effort hygiene; the epoch fence, not the kill, is what guarantees a zombie cannot deliver.
- On expiry: if the chunk's current node has been retried fewer times than the node's `retries.max` ([graph.md](../domain/graph.md)), requeue it at that node — a retry is a new attempt: new session, new lease, fresh epoch (D-082). Otherwise report it `needs-human` and park it. Retries count **execution-attempt failures only** — reaps, crashes, verdict-less exits (D-078); a judged failure choice traversing an edge never consumes one, and max-visits cycle caps are future scope. The default cap when a node doesn't specify one is an [open-question constant](../../decisions/open-questions.md).
- Reap ends the attempt, never the chunk's tenure: the chunk stays held by this runner and its environments stay bound (D-082/D-083). The staleness threshold is deliberately conservative (~1–2 h default, tracked with the constants): heartbeats ride tool calls, so the threshold is bounded below by the longest tool call a healthy worker makes — one long test run must never read as a stall.

### PULL

Exchange state with the hub (outbound-only, D-012; eventually reachable — buffer and flush through outages). Chunk acquisition itself is FILL's claim-by-route (D-080); PULL is the fact channel.

- Flush buffered facts upward: transitions, judgements, submitted artifacts, questions, escalations — the outbound buffer drains FIFO with per-runner seq idempotency, whether or not an outage preceded (store-and-forward always, D-069; [store.md](./store.md)). The runner never talks to the PM system — its reads of PM items go through the hub's pass-through API, and nothing is written back to the PM item in the MVP (D-047).
- Pick up delivered answers, and read the fleet's brake — adhere to it (D-043). FILL stops on it or on the runner's own brake, whichever is set (D-105).

### FILL

Keep the fleet busy. Since D-080, FILL is also where work is claimed — PULL no longer acquires chunks.

- Compute open agent slots: `slots = MAX_AGENTS − running`.
- For each open slot:
  1. **Peek** the hub's ready queue read-only for the next chunk (the hub's ordering steers what comes next) and determine its environment count.
  2. **Acquire the environments** from the workspace provider — opaque env ids with their workdirs, clean by contract (D-021), the currently-held set passed in (D-062). All-or-nothing: if the provider cannot satisfy the chunk's environment count, release and skip it this tick.
  3. **Claim by route** (D-080): POST the complete route — chunk, runner, workspace, env ids — as one write, and record the chunk→env binding as a chunk-tenure fact (D-083). A `409` is race-second-place — another runner claimed it first: release the bindings and move on; the next tick peeks fresh. The accepted claim's response carries the chunk's first node envelope.
  4. Mint the node-step's lease and **spawn** a headless worker via the adapter (session identity per D-092), primed with the node envelope plus the runner's machine-local preamble — the held env ids and their workdirs (D-063). The spawn wrapper records its own pid + process start time from inside the child, before the harness runs, so no window exists where a live worker is unrecorded.

The recorded binding is what makes recovery a no-op: re-running FILL for a chunk that already has a binding reads it back from the runner's own facts instead of re-acquiring — the provider, which keeps no allocation state (D-062), is never even called. Nothing is duplicated (D-021 supersedes the earlier deterministic-env-name scheme), and a crash between the provider's clean and the binding write is absorbed by reset-on-acquire.

### ADVANCE

Judge finished workers and move their chunks through the graph (D-025/D-027).

- A worker exiting is its **done declaration** (two-phase prompting, D-038) — *unless* an open **ask fact** was recorded for its lease during the session (`blizzard runner ask` hits the runner's local API before the worker exits — see [ask-answer.md](../ask-answer.md)), in which case the chunk parks as `waiting_on_human` instead. Exit *is* the done declaration (D-055): any verdict-less exit without an open ask receives the judgement resume, and the judgement reply — or its absence (D-009) — is what tells a done apart from a crash; a session that cannot answer its judgement fails, and the attempt counts against the node's retries.
- Deliver the node's **judgement prompt** into the same session (the adapter's resume-with-message operation, D-038 — the same lease, D-082) to elicit the structured verdict — the engine-generated `<Choice>{name}</Choice>` selection over the node's choices (D-042) plus the assessment payload, which reports the results of the node's checks the worker ran in-session (D-077). The verdict is the judgement (MVP form); engine-run deterministic checks are post-MVP. Judgement prompt delivered, no parseable `<Choice>` back → failure (D-009).
- Submit the node-step's completion to the hub as **one atomic write** (D-036): the judgement choice and check results together with the step's **artifacts**, branch/commit pointers (pushed to the forge first) and assets (D-026); the write carries the lease's epoch, and the hub rejects stale ones (D-007). The completion travels through the outbound buffer like every hub-bound fact (D-069), and the hub's **apply-response** — the transition recorded, any pending migration applied atomically with it — returns the chunk's next node envelope (D-072); a lost response cannot wedge the boundary — a retried flush acked as already-applied sends the runner to the idempotent envelope read (`GET /chunks/{id}/envelope`, D-090). Then, per that response:
  - **next runner node** → continue in place: same chunk, same environment, no re-queue — spawn (or resume) the next node's worker with the returned envelope, artifacts already at hand. A migration is visible here as the envelope's new graph/node ids — discovered, never inferred. Hub unreachable → the chunk waits at the boundary, completion durable in the buffer; the worker mid-node is never affected.
  - **hub node** (the default graph's deliver node, D-030) → the runner's part is done: pointers are pushed and submitted; the hub merges through its merge queue. The runner holds the chunk's environments until the hub reports the terminal outcome, including across a PR-mode wait for an external merge (D-066) — a landed (or externally closed, D-065) delivery releases them back to the workspace provider; a conflict routes back through the graph into the same warm environments (D-058).
  - **failure** → requeue at the node or escalate per the node's `retries` config ([graph.md](../domain/graph.md)).

## Workers heartbeat as a side effect of working

Progress detection requires **no cooperation from the agent**. The worker heartbeats via a `PostToolUse` hook — every tool call the agent makes emits a heartbeat. An agent that is genuinely working cannot help but heartbeat; an agent that has stalled stops heartbeating on its own. The supervisor never has to trust the agent to self-report.

## Verdicts

A verdict is elicited, never volunteered (D-038/D-042): when the worker declares done, the runner resumes the session with the node's judgement prompt — the authored prose plus the engine-generated elicitation tail — and parses the reply's `<Choice>{name}</Choice>` against the node's choice set.

A **missing or unparseable Choice is treated as failure** (D-009). The system fails safe: absence of a clear success signal is never read as success. The verdict is the agent-supplied half of the node judgement; the node's deterministic checks are the other half (D-025). The assessment payload rides harness-native structured output (`--output-format json` / `--output-schema`), normalized by the adapter's `verdict()` — no file convention (D-056).

## Idempotency and recovery

Every step is idempotent and every chunk→env binding is a durable runner-store fact, so recovery re-runs no-ops:

- Killed mid-FILL → the lease is intact; acquisition is idempotent per chunk, so FILL re-reads (or re-requests) the same env binding.
- Killed mid-ADVANCE → the session holding the judgement reply is still on disk (re-delivering the judgement prompt is idempotent — same session, same choices), the atomic transition+artifacts write is idempotent and epoch-fenced (D-036), and the transition is recorded exactly once — re-running advances once.
- Killed entirely → restart runs REAP first and continues.

`kill -9` at any step boundary is a supported, tested operation, not an edge case.
