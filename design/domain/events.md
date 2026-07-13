# Facts and events

The canonical fact vocabulary (D-067): every fact a derived status computes from, the store that owns it, and the derivation queries themselves — so no implementer improvises a status column (D-004). Part of the [domain model](./index.md). The [status enum](./work.md#status-enum) summarizes what these derive; this file is the authority on *how*.

Fact names double as event names: the SSE stream (`GET /events/stream`) re-broadcasts hub-landed facts under the same `noun.verb` names, and the batched push (`POST /events`) carries runner-minted facts up under them too — one vocabulary, no separate wire naming. (Batching, dedup, and idempotent ingestion keys for the push remain part of the runner↔hub protocol [open question](../../decisions/open-questions.md).)

## Hub-owned facts

Landed at the hub by its API routes ([api.md](../hub/api.md)); the hub store is their only owner. Runner-authored writes in this list (transitions, decisions, questions) are hub facts — the runner authors them, but they exist only once the hub accepts them.

| Fact | Landed by | Notes |
|------|-----------|-------|
| `chunk.minted` | `POST /chunks` | Ingestion by PM pointer (D-047); carries the graph pin (D-033). |
| `chunk.grouped` | `POST /chunks/{id}/group` | The merged-away chunks are discarded into the combined one (D-048). |
| `chunk.discarded` | `DELETE /chunks/{id}` | Unacquired chunks only (D-047). |
| `chunk.stopped` | `POST /chunks/{id}/stop` | Operator abandonment, terminal (D-067): legal at any point after acquisition, artifacts and history retained. Terminality is its own fence — every later state-advancing write for the chunk is rejected regardless of epoch. |
| `route.created` | `POST /routes` | Acquisition (D-021/D-024): chunk → runner → workspace → env. |
| `lease.minted` | `POST /events` | Runner-minted (D-044), reported up with its epoch — the fence's input, and what closes an open escalation by supersession. |
| `transition.recorded` | `POST /chunks/{id}/transitions` | Accepted, epoch-fenced, atomic with its artifacts (D-027/D-036). Current node derives from the newest one. |
| `decision.submitted` | `POST /chunks/{id}/decisions` | A gate's parking row (D-045); open while no transition references it. |
| `decision.resolved` | `POST /decisions/{id}/resolution` | First-write-wins; the resolving transition follows from the runner (D-027). |
| `question.asked` | `POST /questions` | The durable question row ([ask-answer.md](../ask-answer.md)); open while no answer row exists. |
| `question.answered` | `POST /questions/{id}/answer` | First-write-wins CAS; this row alone flips the chunk out of `waiting_on_human`. |
| `delivery.landed` | deliver node | The merge queue landed the chunk's git-commit artifacts across all its repos (D-030/D-057). Terminal. |
| `delivery.conflicted` | deliver node | Merge/rebase conflict; the routing it causes lands as its own migration or transition fact (D-058). |
| `pr.opened` | deliver node | PR mode (D-059); the board's "awaiting external merge" detail derives from this plus the absence of `pr.closed`. |
| `pr.closed` | deliver node / `POST /chunks/{id}/check-delivery` | Detected by poll or on-demand check (D-065); carries `merged` and the actually-landed commit where one exists. Terminal either way. |
| `migration.requested` / `migration.applied` | `POST /chunks/{id}/migrations` / apply time | Intent vs record (D-037); pending derives as request-without-record. |
| `runner.registered` / `runner.paused` / `runner.resumed` | `POST /runners` / `PATCH /runners/{id}` | Registry facts; `paused` derives from the newest pause/resume fact (D-043). `last_seen_at` is a refreshed timestamp, not an event. |
| `graph.minted` / graph metadata facts | `POST /graphs` / `PATCH /graphs/{id}` | Definition immutable (D-033); `enabled`, `auto_migrate`, `migration_target` derive from appended metadata facts (D-039/D-040). No status derives from these. |

## Runner-minted facts

Minted in the [runner store](../runner/store.md) (D-023). Three travel to the hub through the outbound buffer and `POST /events`, because fleet-visible derivations consume them; the rest never leave the machine.

| Fact | Travels? | Notes |
|------|----------|-------|
| `lease.minted` | → hub (D-044) | One lease per node-step attempt (D-035), epoch attached. |
| `escalation.recorded` | → hub | Retries exhausted, or a worker died without a verdict past the retry cap (D-009). Recorded before the resume command is printed. |
| `answer.delivered` | → hub | The resume-with-answer executed ([ask-answer.md](../ask-answer.md)); restarts the reap clock. Status flipped already at `question.answered`. |
| `takeover.started` / `takeover.ended` | → hub | A human entered/left a parked chunk's session ([cli.md](../cli.md)); board detail (human-in-session), no status derives from it. |
| env binding, pid + start time, heartbeat, tool activity, verdict, local open-ask copy | never | Machine-local execution truth ([store.md](../runner/store.md)); the worker heartbeat in particular never leaves the box, which is why `stalled` is a runner-local derivation, not a fleet status. |

## Chunk status derivation

The queries behind the [status enum](./work.md#status-enum), evaluated at the hub over the facts above. **Precedence: first match wins**, top to bottom — a chunk with facts matching several rows has exactly one status.

| Status | Query |
|--------|-------|
| `stopped` | A `chunk.stopped` fact exists. |
| `done` | A `delivery.landed` or `pr.closed` fact exists (either disposition — D-065). |
| `needs_human` | An open escalation: `escalation.recorded` with no later `lease.minted` for the chunk (D-067 supersession — requeue and takeover both mint a lease, which closes it by construction; there is no resolution fact). |
| `waiting_on_human` | An open question (`question.asked` without `question.answered`) or an open decision (`decision.submitted` no transition references). The reap clock is stopped. |
| `delivering` | The newest accepted transition's `to_node` is a hub node (D-030) — queued, merging, or awaiting an external merge (`pr.opened` without `pr.closed`); the runner holds environments throughout (D-066). |
| `running` | A `route.created` exists. |
| `ready` | The chunk is minted and none of the above — not routed, not grouped away, not discarded. |

`chunk.discarded` and `chunk.grouped` remove the chunk from every listing rather than deriving a status: the PM item is the durable referent and a discarded chunk is simply gone (D-047).
