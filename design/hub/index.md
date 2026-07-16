# The hub (`blizzard-hub`)

The **work orchestrator**: the fleet's shared memory and the human's front door. It ships in `milestone:mvp` (D-022) — colocated on the runner machine in the solo setup, on a VPS / cloud / home server from `milestone:centralized-hub`. Where it runs is a deployment choice; how it is spoken to never changes. It is not the runner's store relocated: the runner keeps its own embedded database for the machine-local fast path (leases, heartbeats, epochs, env bindings), while the hub owns everything fleet-visible — see the [division of truth](../architecture.md#division-of-truth).

| Document | What it covers |
|----------|----------------|
| [store.md](./store.md) | The hub store: the fleet facts it records, its symmetry with the runner store, what stays open. |
| [api.md](./api.md) | The HTTP API + SSE surface: the route table every client speaks. |
| [web-app.md](./web-app.md) | The hub's web front: MVP observability + queue shaping (prioritize, group — D-048), and the remote board slice. |
| [sample-graph.yaml](./sample-graph.yaml) | An illustrative workflow graph definition — not schema; makes the node/judgement/edge concepts concrete. |

## Responsibilities

This is the canonical list — other documents point here rather than restating it.

- **Wraps the project-management layer (D-024/D-047).** The PM binding — GitHub issues is one — plugs into the hub, and work enters by **explicit, id-addressed ingestion** via `blizzard hub ingest` / `POST /chunks` (D-074): ingesting an item stores a **pointer** — `{provider, url}`, the binding tag plus the item's canonical web URL (D-075) — and mints a chunk by default, one per item. Item contents are read **pass-through**: anything that needs the guts (the build node included) fetches them through the hub's API — body and comment thread, vendor-native format (D-074) — and the hub calls the PM system with its own credentials, a per-provider token in hub config (OAuth/app registration waits for the identity spike) — PM credentials never reach a runner; the payload's consumers are LLMs. Selection — which items are fleet-ready — happens outside the hub: humans or agents ingest specific ids (the agent-native flow is a documented pattern, not shipped machinery); there is no label sweep. Chunks are ephemeral: any unacquired chunk may be discarded — real deletion; acquired-with-history chunks are stopped instead (D-067/D-074) — and re-ingesting the same pointer mints a fresh chunk only once every prior holder is terminal: a pointer with a live chunk is rejected at ingest, 409 with the existing chunk id (D-093). The MVP writes nothing back to the PM item. Runners never speak to the PM system; when a runner wants work, it asks the hub. Queue **prioritization is a hub-side property** — mechanism [open](../../decisions/open-questions.md). Hub-side routing (pinning work to a runner) also lives here, post-MVP.
- **Owns the workflow definitions and the record (D-025/D-027).** The YAML graphs live here; every judgement and transition is recorded here — the hub always knows which node every chunk is at. Hub configuration names the **default graph** — by name, resolving to its newest enabled mint — that every ingested chunk pins (D-081); the first graph is provisioned manually via `POST /graphs`, and per-chunk graph selection is post-MVP. See [workflow-engine.md](../workflow-engine.md).
- **Executes hub nodes through the coordinator — delivery first (D-030/D-079).** The **coordinator** — a singleton, runner-like internal component — acquires the chunk's hub node-step, executes it, and records the exit transition under a lease and epoch it mints itself. The deliver node runs there: branch artifacts, already pushed to the forge, are merged to main (or opened as a PR) through an internal **merge queue**, one chunk at a time, with stale-epoch submissions rejected before they enter it — serially per repo, best-effort, with per-repo landed facts and reconciliation on retry (D-091). The queue is strict FIFO with no train batching (D-057); conflicts route intra-graph in the MVP (D-086), to the conventional `merge` graph once migration ships (D-058); PR merging is capability-gated on the delivery binding (D-059), externally-merged PRs detected by polling plus the on-demand check (D-065), environments held until the terminal outcome (D-066/D-083).
- **Stores chunk state and artifacts (D-026).** Per chunk: current node, history, questions, verdicts, and artifacts — branch/commit pointers and assets. Plus **routes**: chunk C is being worked by runner R, in workspace W, on environment E (D-021) — every chunk is locatable.
- **Is the ask/answer rendezvous.** Question and answer rows are *born* here; the hub applies first-write-wins CAS on answers and emits the events that resume dormant agents. Full protocol: [ask-answer.md](../ask-answer.md).
- **Keeps the fleet registry.** Runners register (runner id + workspace id) and connect outbound-only; each registry entry carries the runner's declarative operational state — `hub_paused` now, routing knobs post-MVP (D-043) — set via the API and read back by the runner on its outbound pull. It also holds the runner's *own* brake (`locally_paused`), which the hub never sets and only learns by report (D-105) — see [domain/fleet.md](../domain/fleet.md).
- **Serves the clients.** HTTP API + SSE for the CLI's fleet verbs and the hub-served web app from the MVP (D-048), the chat bot later; the auth seam (D-018) guards this surface.

Two structural constraints hold throughout: it **never stores code or transcripts** (D-012) — pointers and assets, not worktrees; it cannot leak what it does not hold — and its sqlite is encapsulated by the daemon (D-023); clients see only the HTTP API.

For most chunk facts the hub is the authority because they are born there or reported to it as the single record; the runner store is authoritative only for the machine-local execution facts that never leave the box. The wire is settled: store-and-forward from each runner with per-runner sequence idempotency (D-069), and a dedicated runner-liveness heartbeat (D-070).

## Topology: colocated in the MVP, remote from `milestone:centralized-hub`

The hub/runner separation is the architecture from day one (D-022): the runner speaks the same outbound-only protocol whether the hub is local or remote, so moving it off-machine — and registering multiple runners — changes nothing architecturally. What `milestone:centralized-hub` adds is *reach*: the remote hub, the board, remote ask/answer fan-out, and chat + auth.

A runner's connection is **outbound-only** (D-012): runners poll the hub or subscribe to its SSE stream; **nothing dials into the dev box** — the same pattern as Anthropic's self-hosted sandbox workers, where the sensitive machine only makes outbound connections. The hub is **eventually reachable**: during an outage, runners keep working on already-acquired chunks, buffer events locally, and flush when it returns.

The runners themselves — the supervisor, workers, environments, and the per-runner workspace binding (D-019) — are the runner component's territory: [runner/](../runner/index.md).

## Clients: the web app and the chat bot

The **web app** ships in the MVP, served by the hub itself (D-048): fleet observability plus ready-queue prioritization and chunk grouping — [web-app.md](./web-app.md). At `milestone:centralized-hub` it grows its remote slice — phone-friendly PWA, and the role split: **viewer** (sees everything, touches nothing) and **operator** (answers questions and drives limited controls; the exact control set is an [open question](../../decisions/open-questions.md)).

The **chat bot** is Telegram (D-031), carrying question notifications with inline answer buttons — see [ask-answer.md](../ask-answer.md).

**Auth is configurable, and the first hub ships with none** (D-018): anyone who can reach the app can do anything, with Tailscale as the network perimeter. A cloud-hosted hub needs real identity (OAuth2-class) and per-user permissions — that design is a dedicated post-MVP spike, not something bolted on early.

"You can't jump into the code" is **structural**, not a missing feature: a resume command only works on the machine that holds the environment, and the hub never holds environments. The remote surface can direct the fleet but can never itself touch a worktree.
