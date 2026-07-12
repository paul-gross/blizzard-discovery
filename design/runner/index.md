# The runner (`blizzard-runner`)

The machine-level fleet agent-of-agents: one runner per machine, bound to one prepared workspace (D-019), operator-directed. It **acquires** work — it is never pushed work — and arbitrates three capacities (work, agent slots, environment slots) while keeping its loop stateless: every fact lives in the [runner store](./store.md) — sqlite embedded in the daemon behind its local API (D-023/D-028) — so `kill -9` at any instant is a supported operation.

Responsibilities, in one pass: acquire ready chunks from the hub (D-024 — never from the PM system); acquire clean environments by id from the workspace provider; spawn workers through harness adapters, one node-step at a time; heartbeat-watch them; judge node exits (verdict + checks) and advance chunks through the hub's workflow graph without re-queuing (D-025/D-027); push branch artifacts and submit them to the hub (D-026) — delivery itself executes at the hub (D-030); escalate with a literal resume command; record every transition and event to the hub (colocated in the solo setup, D-022); serve the `blizzard` CLI through its local API. The operator can pause it, start it, and choose which work it slots.

| Document | What it covers |
|----------|----------------|
| [store.md](./store.md) | The embedded database: why sqlite, what it records, epochs — the runner's private state behind the local API. |
| [loop.md](./loop.md) | The reconciliation loop — REAP / PULL / FILL / ADVANCE — crash recovery, idempotency, verdicts, operator controls. |
| [environments.md](./environments.md) | The environment model: opaque env ids, the workspace-seam contract sketch, reset-on-acquire, environments riding the lease, the three capacities, worked winter and plain-git bindings. |
| [contracts.md](./contracts.md) | Every API surface the runner speaks — local API, workspace provider, harness adapter, hub, operator — tracked as explicit contracts with maturity markers. |
| [api.md](./api.md) | The local API's route table — the resource surface worker hooks and the CLI's local verbs speak, the CLI-verb mapping, wire conventions, and what is deliberately not a route. |

Context above this component: [the architecture map](../architecture.md) for how the runner sits among the other pieces and the facts-only principle its store follows; [harness-adapters.md](../harness-adapters.md) for the adapter interface it drives.
