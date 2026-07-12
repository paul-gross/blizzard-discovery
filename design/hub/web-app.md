# The web app

The hub's web front (`epic:board`, D-048): a browser surface served by `blizzard-hub` itself, giving the operator observability over the fleet and the queue-shaping controls that are clumsy as CLI verbs. It ships in `milestone:mvp` — colocated hub, local browser, authless behind Tailscale (D-018) — and the same app *is* the remote board at `milestone:centralized-hub`: going remote adds reach and roles, never a second app.

Like every client, it is a **pure client of the hub's HTTP API + SSE** ([api.md](./api.md)): no private side channel, no direct store access (D-023). And like every hub surface it is structurally unable to touch code — the hub holds pointers and assets, never worktrees or transcripts (D-012).

## MVP slice: observability + queue shaping (D-048)

**Observability** — every view reads existing routes, live over SSE:

| View | Backed by |
|------|-----------|
| Fleet view: every chunk with its derived status, current node, and runner | `GET /chunks`, `GET /runners` |
| Chunk detail: the aggregate root — node history, judgements, artifacts, questions/decisions, PM pointers, route | `GET /chunks/{id}` |
| The ready queue in its current order | `GET /chunks` |
| Open questions and decisions awaiting a human | `GET /questions` |
| The merge queue: queued, landing, landed | `GET /merge-queue` |

**Controls** — the two operator actions that shape the queue rather than execute work:

- **Prioritize.** Reorder the ready queue. With id-addressed ingestion (D-047) ordering is a hub-side property, and the web app is the surface where the operator sets it; the ordering mechanism itself (explicit rank, aging, starvation protection) is the queue-ordering [open question](../../decisions/open-questions.md).
- **Group.** Merge unacquired chunks into one: the combined chunk carries the union of their PM pointers (D-047), and the merged-away chunks are discarded — they are ephemeral. Solo execution still holds: one environment, one agent per chunk; grouping widens what a single agent takes on, it never parallelizes a chunk. Manual grouping is not batching — the heuristic packer and LLM planner (`epic:batching`) stay parked.

Whether the MVP web app also answers questions and resolves gate decisions — `blizzard answer` is the MVP acceptance path — or renders them read-only is part of the web-app-controls [open question](../../decisions/open-questions.md), as are the rank and group route shapes ([api.md](./api.md)).

## Remote slice (`milestone:centralized-hub`)

What going remote adds — reach, never architecture: phone-friendly PWA packaging, the **viewer**/**operator** role split (operator control scope [open](../../decisions/open-questions.md)), auth (D-018: authless behind Tailscale locally, OAuth2-class identity for cloud-hosted hubs), and question notifications with remote ask/answer fan-out ([ask-answer.md](../ask-answer.md)).

## Shape

Server-rendered pages plus SSE for live updates — deliberately boring; the PWA shell arrives with the remote slice. Implementation details (framework, how the hub serves it) are unshaped, deliberately, during discovery.
