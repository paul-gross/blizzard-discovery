# The `blizzard` CLI

The third surface named by D-020: **one executable** (D-061) whose client verbs are **pure clients of the two daemons' APIs** (D-023) — runner verbs hit the runner's local API ([runner/api.md](./runner/api.md)), hub verbs hit the hub's HTTP API ([hub/api.md](./hub/api.md)) — and whose two `host` verbs *become* those daemons: the runner and the hub are invocation personalities of this same binary, so CLI↔daemon version skew on a machine is structurally impossible and the shared facts schema ships in one place. Two callers share the client verbs: **worker hooks**, which take their identity from the spawn-injected environment (`BLIZZARD_LEASE_ID`, `BLIZZARD_SESSION_ID` — [harness-adapters.md](./harness-adapters.md)) and pass no identity arguments; and **humans at the machine**. **Maturity: shape settled (D-061)** — the noun grouping, the `host` personalities, and one-binary packaging are decided; individual flags and output formats remain unpinned.

## Dispatch is spelled in the command

Every verb is namespaced by the system it targets (D-061): `blizzard runner <cmd>` hits the runner's local API, `blizzard hub <cmd>` hits the hub's HTTP API. No bare verbs, no dispatch table to memorize — a CLI that targets two systems says which one in the command. Worker-hook verbs are runner verbs, because their facts land in the runner store; the ask/answer pair spans the two systems deliberately — `blizzard runner ask` records the local ask fact (forwarded to the hub, where the question row is born), and `blizzard hub answer` writes the answer where that row lives. Each noun also carries the one verb that means "become it" rather than "talk to it": `host` launches that system's daemon personality (D-061), with D-020's hyphenated names (`blizzard-runner`, `blizzard-hub`) as alias entry points for systemd and `ps` legibility.

## `blizzard runner <cmd>` — the machine-local surface

| Verb | Caller | API | Notes |
|------|--------|-----|-------|
| `blizzard runner host` | systemd / operator | — becomes the daemon | Launches the `blizzard-runner` personality (D-061): the reconciliation loop plus the local API every other verb in this table speaks. `ExecStart=… blizzard runner host`; the `blizzard-runner` alias is this. |
| `blizzard runner heartbeat` | worker hook | `POST /leases/{id}/heartbeats` | Fired by the `PostToolUse` hook on every tool call; lease id from the inherited environment ([harness-adapters.md](./harness-adapters.md)). |
| `blizzard runner ask "…" --options "a\|b"` | worker | `POST /leases/{id}/asks` | Ask-and-exit (D-010/D-015, [ask-answer.md](./ask-answer.md)); the ask fact is durable before the worker exits. |
| `blizzard runner status` | human | `GET /runner` + `/chunks` + `/asks` + `/environments` + `/escalations` | The machine-local view: capacities, held environments, open asks, escalations — derived only (D-004). |
| `blizzard runner pause` / `blizzard runner start` | human | `PATCH /runner` | Declarative controls (the D-043 pattern applied locally); keep working while the hub is unreachable. |
| `blizzard runner takeover <chunk-id>` | human | `POST /chunks/{id}/takeovers`, then exec | Below. |
| `blizzard runner requeue <chunk-id>` | human | `POST /chunks/{id}/requeues` | The takeover hand-back: appends the fact that clears `needs-human`, making the chunk leasable at its current node next FILL; `409` while the takeover session's process is still alive. |
| `blizzard runner selftest <coding-harness>` | human | `POST /selftests` | Adapter-drift canary before any unattended period ([harness-adapters.md](./harness-adapters.md)). |

## `blizzard hub <cmd>` — the fleet surface

| Verb | Caller | API | Notes |
|------|--------|-----|-------|
| `blizzard hub host` | systemd / operator | — becomes the daemon | Launches the `blizzard-hub` personality (D-061): HTTP API + SSE plus the embedded web app — one binary is the whole hub deployment. The `blizzard-hub` alias is this. |
| `blizzard hub status` | human | `GET /chunks` + `GET /questions` + `GET /runners` | The fleet view: every chunk, open question, and registered runner — what the board renders, in the terminal. |
| `blizzard hub answer <question-id> "<answer>"` | human | `POST /questions/{id}/answer` | First-write-wins CAS at the hub; the loser is told who already answered ([ask-answer.md](./ask-answer.md)). |
| `blizzard hub decisions` / `blizzard hub decide <decision-id> <choice>` | human | `GET` open decisions / `POST /decisions/{id}/resolution` | The CLI half of gate surfacing (D-052) — same hub route as the board's buttons; resolution is first-write-wins like an answer (D-045). |
| `blizzard hub ingest …` | human | `POST /chunks` | Ingest PM items by pointer — `{provider, url}` per item (D-074/D-075); specific ids always, batch fine. |

## `blizzard runner takeover`

Takeover addresses the **chunk** — the id a human actually has in hand from `blizzard runner status` or the board — never a session id or a worktree path. The verb POSTs to the runner, which verifies the chunk is takeable — parked (`needs_human`, `waiting_on_human`, or at a gate) with no live lease (D-035); a running chunk draws `409` (a `--force` that reaps the live lease first is an open flag) — records the takeover fact, and returns the adapter-composed interactive command (`resume_cmd`, [harness-adapters.md](./harness-adapters.md)) with its working directory. The CLI runs that command as its own child in the operator's terminal — the daemon never touches a TTY — and marks the takeover ended on the resource when the child exits, so the runner always knows whether a human is inside the session. Takeover facts flush to the hub like any event, so the board can show human-in-session. The interactive session carries no worker hooks and no lease: nothing heartbeats it, nothing reaps it, and its exit declares nothing — exit-is-done (D-055) applies only to runner-spawned workers.

Hand-back is explicit: the human runs `blizzard runner requeue <chunk-id>` when the chunk should re-enter the fleet. Requeue refusing while the takeover process is still alive is what enforces the one-process-per-session rule at the seam where it could otherwise be broken.

## What the CLI is not

- **Never a database client** — nothing but the owning daemon opens a sqlite file (D-023); the `host` personalities *are* those owning daemons, and every other verb is a pure API client.
- **Not the loop** — lease, bind, release, reap are in-process daemon writes and never surface as verbs ([runner/api.md](./runner/api.md)).
- **Not a push channel into the box** — fleet controls travel hub → runner over the runner's outbound pull (D-012/D-043); the CLI's runner verbs keep working while the hub is unreachable.
