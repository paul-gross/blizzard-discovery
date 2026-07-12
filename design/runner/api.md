# The runner local API

The runner serves one local API to the only two clients on its machine: **worker hooks** (the heartbeat and ask paths) and the **`blizzard` CLI's local verbs** (D-020/D-023 — fleet verbs go to the hub's HTTP API instead). Like the [hub's surface](../hub/api.md), it is resource-oriented: facts are POSTed as new subordinate resources, controls are PATCHed as declarative fields on the resource they govern, and every read returns state derived by query — never a stored status column (D-004). **Maturity: draft** — this is the route-level shape of the local-API contract ([contracts.md](./contracts.md) §1) and the operator contract's local half (§6); the routes are proposals with no decision entries yet, and the transport (unix domain socket vs localhost HTTP — the hook path must stay milliseconds-cheap) is tracked in the runner↔hub protocol [open question](../../decisions/open-questions.md).

What the resource framing clarifies: the loop's own store writes — minting leases, recording env bindings, releasing, reaping — are **not routes**. The daemon reaches its store in-process (D-023), it is the machine's single writer with no rival to arbitrate against (D-049), and nothing external ever invokes those operations (a worker's *exit* is its done-declaration — nothing calls a release verb). The API is exactly what crosses the process boundary, nothing more.

| Route | Client | CLI verb | Description |
|-------|--------|----------|-------------|
| `GET /runner` | operator | `blizzard runner status` | The singleton: runner id + workspace id (D-019), derived operational state (`paused` — effective across the local flag and the hub registry flag, see the pause-reconciliation [open question](../../decisions/open-questions.md)), the three capacities with used/free counts ([environments.md](./environments.md)), hub reachability and outbound-buffer depth, last tick. |
| `PATCH /runner` | operator | `blizzard runner pause` / `blizzard runner start` | Declarative controls — the D-043 pattern applied locally, so one control model serves both surfaces: `{paused}` now, work-selection knobs (filter or pin what FILL may lease) post-MVP. Pause/start facts append; the flag derives (D-004). Works with the hub unreachable — the operator contract's standing requirement. |
| `GET /leases` · `GET /leases/{id}` | operator | `blizzard runner status` | Lease facts with derived state — running, stale, parked-on-ask ([store.md](./store.md)). Leases are minted only by FILL, in-process (D-049): there is deliberately no POST. |
| `POST /leases/{id}/heartbeats` | worker hook | `blizzard runner heartbeat` | The `PostToolUse` heartbeat ([loop.md](./loop.md)), optionally carrying `{tool}` for the last-tool-activity fact; returns `204`. A heartbeat is genuinely an appended fact, so the subordinate collection is honest REST, not RPC in disguise. The milliseconds-cheap constraint binds here. |
| `POST /leases/{id}/asks` | worker hook | `blizzard runner ask` | The ask fact — `{question, options}` — durable before the asking worker exits (the park-vs-death discriminator, [ask-answer.md](../ask-answer.md)); the runner forwards it to the hub, where the question row is born. `201` with the ask id. |
| `GET /asks?open=true` | operator | `blizzard runner status` | Machine-local open asks — the status view's question list. Fleet-wide questions are the hub's `GET /questions`. |
| `GET /chunks` · `GET /chunks/{id}` | operator | `blizzard runner status` | The machine-local chunk view: current node, lease, env bindings, derived status — a `SELECT`, never a stored column (D-004). The fleet view is the hub's `GET /chunks`. |
| `GET /environments` | operator | `blizzard runner status` | Held environment bindings (env id ← chunk, held-since), derived from lease facts ([environments.md](./environments.md)) — the env-slot utilization view. |
| `GET /escalations` | operator | `blizzard runner status` | Parked `needs-human` records, each carrying its literal resume command ([store.md](./store.md)). |
| `POST /chunks/{id}/takeovers` | operator | `blizzard runner takeover` | Begin a human takeover of a parked chunk: verifies no live lease (`409` otherwise), records the takeover fact (flushed to the hub so the board can show human-in-session), and returns the adapter-composed interactive resume command + working directory for the CLI to exec — the daemon never touches a TTY ([cli.md](../cli.md)). The CLI marks the takeover ended (`PATCH …/takeovers/{tid}`) when the interactive process exits. |
| `POST /chunks/{id}/requeues` | operator | `blizzard runner requeue` | The takeover hand-back: appends the fact that clears `needs-human` and makes the chunk leasable again at its current node next FILL; `409` while a takeover session is still open ([cli.md](../cli.md)). |
| `POST /selftests` · `GET /selftests/{id}` | operator | `blizzard runner selftest` | The adapter-drift canary as a job resource: POST `{harness}` mints a run, GET reads pass/fail — a resource with a result, not an RPC verb. |

## Worker addressing

The runner spawns every worker, so it injects the lease id and pre-assigned session id into the worker's process environment (`BLIZZARD_LEASE_ID`, `BLIZZARD_SESSION_ID`); hooks and `blizzard runner ask` read them — a worker never discovers or guesses its own identity. A reaped lease turns this into a cheap local tripwire: the zombie's next heartbeat or ask gets `410 Gone`, so a runaway worker learns it is fenced at its first tool call after the reap — hygiene only; the hub-checked epoch fence remains the guarantee (D-007/D-049).

## Wire conventions

- JSON bodies; every route is idempotent or CAS-atomic. Hook POSTs carry client-generated idempotency keys so a crash-retry never double-records an ask.
- `409` for conflicting writes, `410` for operations against a reaped lease, `404` for unknown resources.
- Reads compute derived state from recorded facts at query time (D-004); no route exposes or accepts a status column.

## What is deliberately not here

- **Loop-internal store writes** — lease minting, env binding, release, reap: in-process daemon writes (D-023/D-049), invisible to clients.
- **`answer`** — a fleet verb against the hub API (`POST /questions/{id}/answer`), because the question row and its first-write-wins CAS live at the hub ([ask-answer.md](../ask-answer.md)); whether a runner-local fast path exists is the local-ask-answer [open question](../../decisions/open-questions.md).
- **`verdict`** — elicited by the adapter's resume-with-message and parsed from the session by the runner, recorded in-process (D-038/D-042). If the verdict-format [open question](../../decisions/open-questions.md) lands on a hook-reported result, it enters this surface as `PUT /leases/{id}/verdict` — single-slot, idempotent.
- **Anything that pushes into the box** — the hub's controls arrive over the runner's outbound pull (D-012/D-043); this API listens only on the local socket.
