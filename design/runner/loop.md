# The runner loop

The supervisor is a deterministic reconciliation loop — a few hundred lines — run under systemd as `blizzard-runner`. Its loop holds **no state of its own**: every fact lives in the daemon's embedded [store](./store.md) (D-023/D-028). This is what makes it `kill -9` safe. Killing the supervisor loses nothing, and startup recovery is simply the reap step running first.

The runner is machine-level and **operator-directed**: it *acquires* work from the hub (D-024), it is never pushed work. The operator can pause it, start it, and choose which work it slots — filtering or pinning what FILL may lease. Pausing stops new leases; in-flight chunks run to completion.

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
- On expiry: if the chunk's current node has been retried fewer times than the node's `retries.max` ([graph.md](../domain/graph.md)), requeue it at that node. Otherwise report it `needs-human` and park it. Reap-driven and judgement-fail retries consume the same per-node count; the default cap when a node doesn't specify one is an [open-question constant](../../decisions/open-questions.md).

### PULL

Exchange state with the hub (outbound-only, D-012; eventually reachable — buffer and flush through outages).

- Acquire ready chunks from the hub when there is capacity to fill (D-024) — each arrives with its **node envelope**: the current node's prompt, config, and the chunk's relevant artifacts.
- Flush buffered facts upward: transitions, judgements, submitted artifacts, questions, escalations. The runner never talks to the PM system — its reads of PM items go through the hub's pass-through API, and nothing is written back to the PM item in the MVP (D-047).
- Pick up delivered answers, and read the runner's own operational state — adhere to `paused` (D-043).

### FILL

Keep the fleet busy.

- Compute open agent slots: `slots = MAX_AGENTS − running`.
- For each open slot:
  1. Atomically lease the next acquired chunk in the runner store.
  2. Acquire the chunk's environment(s) from the workspace provider — opaque env ids, clean by contract (D-021) — and record the chunk→env binding as a runner-store fact. If the provider cannot satisfy the chunk's environment count, release the lease and skip it this tick.
  3. Spawn a headless worker with a **pre-assigned session id**, pointed at the chunk's env ids and primed with the node envelope.
  4. Record the worker's pid + process start time.

The recorded binding is what makes recovery a no-op: acquisition is keyed by chunk id and idempotent on the provider side, and re-running FILL for a chunk that already has a binding reads it back instead of re-acquiring, so nothing is duplicated (D-021 supersedes the earlier deterministic-env-name scheme).

### ADVANCE

Judge finished workers and move their chunks through the graph (D-025/D-027).

- A worker exiting is its **done declaration** (two-phase prompting, D-038) — *unless* an open **ask fact** was recorded for its lease during the session (`blizzard ask` hits the runner's local API before the worker exits — see [ask-answer.md](../ask-answer.md)), in which case the chunk parks as `waiting_on_human` instead. What exactly "declares done" is — and how a done-exit is told apart from a crash — is an [open question](../../decisions/open-questions.md).
- Deliver the node's **judgement prompt** into the same session (the adapter's resume-with-message operation, D-038) to elicit the structured verdict — the engine-generated `<Choice>{name}</Choice>` selection over the node's choices (D-042) plus the assessment payload. Run the node's configured deterministic checks. Verdict + checks are the judgement (MVP form). Judgement prompt delivered, no parseable `<Choice>` back → failure (D-009).
- Submit the node-step's completion to the hub as **one atomic write** (D-036): the transition — judgement plus the edge taken — together with its **artifacts**, branch/commit pointers (pushed to the forge first) and assets (D-026); the write carries the lease's epoch, and the hub rejects stale ones (D-007). Then:
  - **next runner node** → continue in place: same chunk, same environment, no re-queue — spawn (or resume) the next node's worker with its envelope, artifacts already at hand.
  - **hub node** (the default graph's deliver node, D-030) → the runner's part is done: pointers are pushed and submitted; the hub merges through its merge queue. The runner holds the chunk's environments until the hub reports the outcome — a landed delivery releases them back to the workspace provider; a failed one routes back through the graph.
  - **failure** → requeue at the node or escalate per the node's `retries` config ([graph.md](../domain/graph.md)).

## Workers heartbeat as a side effect of working

Progress detection requires **no cooperation from the agent**. The worker heartbeats via a `PostToolUse` hook — every tool call the agent makes emits a heartbeat. An agent that is genuinely working cannot help but heartbeat; an agent that has stalled stops heartbeating on its own. The supervisor never has to trust the agent to self-report.

## Verdicts

A verdict is elicited, never volunteered (D-038/D-042): when the worker declares done, the runner resumes the session with the node's judgement prompt — the authored prose plus the engine-generated elicitation tail — and parses the reply's `<Choice>{name}</Choice>` against the node's choice set.

A **missing or unparseable Choice is treated as failure** (D-009). The system fails safe: absence of a clear success signal is never read as success. The verdict is the agent-supplied half of the node judgement; the node's deterministic checks are the other half (D-025). How the assessment payload rides alongside the Choice — `.fleet/result.json`, harness-native structured output, or both — is the verdict-format [open question](../../decisions/open-questions.md).

## Idempotency and recovery

Every step is idempotent and every chunk→env binding is a durable runner-store fact, so recovery re-runs no-ops:

- Killed mid-FILL → the lease is intact; acquisition is idempotent per chunk, so FILL re-reads (or re-requests) the same env binding.
- Killed mid-ADVANCE → the session holding the judgement reply is still on disk (re-delivering the judgement prompt is idempotent — same session, same choices), the atomic transition+artifacts write is idempotent and epoch-fenced (D-036), and the transition is recorded exactly once — re-running advances once.
- Killed entirely → restart runs REAP first and continues.

`kill -9` at any step boundary is a supported, tested operation, not an edge case.
