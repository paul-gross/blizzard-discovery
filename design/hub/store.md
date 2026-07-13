# The hub store

The hub's embedded database, symmetric to the [runner store](../runner/store.md): a sqlite (WAL) database **inside `blizzard-hub`**, which is its only reader and writer (D-023). Clients — the CLI's fleet verbs, the board, the chat bot, and the runners themselves — see only the HTTP API + SSE; the schema stays private. The same rationale that picked sqlite for the runner holds here (embedded, ACID, no separate database server to operate), and both stores obey the same [facts-only principle](../architecture.md#store-facts-derive-status): durable facts in, status derived by query, never written as a column (D-004).

The store holds the top two tiers of the [division of truth](../architecture.md#division-of-truth): the fleet's view of the work (chunks) and the record of its movement. Nothing machine-local (leases, heartbeats, pids, env bindings) lives here — that is the runner store's territory; lease facts and their epochs, minted runner-side, are the one exception, because the fence consumes them (D-044) — and **never code, never transcripts** (D-012): a pointer to code is not the code.

## What it records

- **Chunks** — the hub-minted id wrapping the PM back-reference(s) (D-024); the exact identity scheme is an [open question](../../decisions/open-questions.md).
- **Workflow definitions** — the YAML graphs (D-025), and per chunk the transition record: every judgement and node transition the holding runner reports (D-027), so the hub always knows which node every chunk is at.
- **Artifacts** — pointers (branch/commit per repo, pushed to the forge before submission) and assets (text or blobs), accumulated per chunk (D-026). Size limits, blob storage, and retention are an [open question](../../decisions/open-questions.md).
- **Questions and answers** — durable rows born here ([ask-answer.md](../ask-answer.md)); a question's answered-ness is derived from the presence of its answer row, with first-write-wins CAS on answering.
- **Routes** — chunk C → runner R → workspace W → environment E (D-021), so every chunk is locatable.
- **Lease facts** — each lease mint (chunk, epoch, runner), reported by runners (D-035/D-044); a chunk's latest epoch derives from these — the input the transition fence checks stale submissions against (D-007/D-036).
- **The fleet registry** — registered runners (runner id + workspace id) with their operational state: pause/resume facts append, `paused` derives (D-004/D-043), read back by runners on their outbound connection; a runner-level liveness signal for the board is an [open question](../../decisions/open-questions.md).
- **The merge queue** — the deliver node's serialized delivery facts (D-030): strict FIFO, one chunk at a time across all its repos (D-057); externally-merged PRs detected by poll or on-demand check (D-065).

## What stays open

The wire protocol that feeds this store (event batching, dedup, idempotent ingestion keys for flush-after-outage), the artifact storage details, chunk identity, and prioritization/ordering of the ready queue are all tracked in [open questions](../../decisions/open-questions.md). The fact vocabulary the store records and the status derivations over it are canonical in [events.md](../domain/events.md) (D-067).
