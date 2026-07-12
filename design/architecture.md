# Architecture — the running pieces

What actually runs, where it runs, and how the pieces talk to each other. Each piece has a deep-dive doc; this page is the map. Names per D-020: the app is `blizzard`, the hub daemon is `blizzard-hub`, the runner is `blizzard-runner` — one executable (D-061): the daemons are `host` personalities of the single binary (`blizzard runner host`, `blizzard hub host`), the hyphenated names its aliases.

## The pieces at a glance

| Piece | What it is |
|-------|-----------|
| [**`blizzard-hub`**](#blizzard-hub) | the work orchestrator: PM binding, queue, workflow record, artifacts, asks, the merge queue — HTTP API + SSE (D-022/D-024/D-030) |
| [**`blizzard-runner`**](#blizzard-runner) | the supervisor: a stateless reconciliation loop behind a local API, advancing chunks through the workflow graph |
| [**workers**](#workers) | the harness processes doing the work (`claude -p …`), one per leased chunk's node-step |
| [**`blizzard-cli`**](#blizzard-cli) | a pure client of the daemons' APIs, verbs namespaced by target (`runner heartbeat`, `runner ask`, `hub answer`, `hub status`, …) |
| [**workflow graphs**](#workflow-graphs) | hub-configured YAML: the node graph every chunk travels (D-025) |
| [**workspace provider**](#workspace-provider) | allocates clean environments by opaque **environment id** (D-021) |
| [**web app / chat bot**](#web-app--chat-bot) | fleet observability, queue shaping (prioritize / group), question answering, operator controls |

### `blizzard-hub`

The work orchestrator, running as a daemon from the MVP (D-022) — colocated on the runner machine in the solo setup, on a VPS / cloud / home server from `milestone:centralized-hub`. Where it runs is a deployment choice; how it is spoken to never changes. Wraps the PM system into chunks (D-024), owns the workflow graphs and the transition record (D-025/D-027), stores chunk state and artifacts (D-026), executes the deliver node's merge queue (D-030), and is the ask/answer rendezvous. Talks to the PM binding, the forge (delivery), the runners (they dial in), the board, and the chat bot. Deep dive: [hub/](./hub/index.md) — the canonical responsibility list lives there.

### `blizzard-runner`

Runs as a daemon under systemd, one per runner machine, exposing a **local API** that serves the `blizzard` CLI (D-023). It advances the chunks it holds through consecutive workflow nodes (D-027). Talks to:

- its [embedded store](./runner/store.md) — sqlite (WAL), in-process; only the daemon touches the file (D-023/D-028)
- the hub (outbound-only; colocated in the solo MVP setup, D-022) — chunks and answers in, transitions, artifacts, and questions out
- the workspace provider
- workers, via adapters
- git remotes — pushing branch artifacts (D-026); delivery itself executes at the hub (D-030); never the PM system (D-024)

Deep dive: [runner/](./runner/index.md).

### workers

Run as child processes of the runner, working in the chunk's leased environments on one node-step at a time, primed with the node envelope. Talk to the runner's local API via the CLI (from hooks), and to the code itself. Deep dive: [harness-adapters.md](./harness-adapters.md).

### `blizzard-cli`

Runs as short-lived invocations of the one `blizzard` binary (D-061). A pure client: runner verbs hit the runner's local API, hub verbs hit the hub's HTTP API — the client verbs never open a sqlite file (D-023); the daemons themselves are the same binary's `host` personalities. Deep dive: [cli.md](./cli.md).

### workflow graphs

**Not a process** — YAML configuration at the hub defining the node graph (build → review → deliver in the default) that every chunk travels; judgements at node exits pick the edges. The hub owns the definitions and the record; the holding runner executes the transitions (D-025/D-027). A planning station, where a graph wants one, is just a node. Deep dive: [workflow-engine.md](./workflow-engine.md). (The parked LLM *batching* planner — shaping many PM items into one chunk — is queue-shaping at the hub, not a running piece: [planner.md](./planner.md).)

### workspace provider

Allocates clean environments by opaque environment id: a plain binding mints `feature-{work-tag}` worktrees, the winter binding hands out its existing envs (alpha, beta, …). Runs as an invoked binary, one binding per runner (D-019). Talks to the filesystem / git. Deep dive: [runner/environments.md](./runner/environments.md).

### web app / chat bot

Web clients of the hub. The hub-served **web app** runs from the MVP (D-048): fleet observability plus queue shaping — prioritize the ready queue, group unacquired chunks; its remote slice (PWA reach, viewer/operator roles) and the Telegram bot (D-031, one-tap answers) arrive with `milestone:centralized-hub`. Talk to the hub only — never a runner. Deep dive: [hub/web-app.md](./hub/web-app.md).

## Inside one runner machine

```text
┌─ one runner machine ─────────────────────────────────────────────────────────────────────────────┐
│                                                                                                  │
│   ┌──────────────────────────────────────────┐             ┌───────────────────────────────────┐ │
│   │ blizzard-runner                          │             │ workers                           │ │
│   │ (daemon under systemd)                   │ spawn / kill│                                   │ │
│   │                                          │────────────▶│ one harness process per leased   │ │
│   │ tick: REAP → PULL → FILL                 │(via adapter)│ chunk, working in the chunk's     │ │
│   │       → ADVANCE                          │             │ leased env ids                    │ │
│   │                                          │             │                                   │ │
│   │ advances chunks node → node per          │             │ hooks: heartbeat on every tool    │ │
│   │ the hub's workflow graphs (D-027)        │             │ call; ask-and-exit; verdict via   │ │
│   │                                          │             │ judgement resume (D-038)          │ │
│   │ ┌─ local API (blizzard verbs) ─────────┐ │             └───────────┬───────────────────────┘ │
│   │ │ runner store — sqlite (WAL):         │ │◀────────────────────────┘                         │
│   │ │ facts only; status always derived;   │ │    blizzard <verb> (from hooks)                   │
│   │ │ only the daemon touches the file     │ │                                                   │
│   │ └──────────────────────────────────────┘ │                                                   │
│   └──────────────────────────────────────────┘                                                   │
│                                                                                                  │
│   workspace provider (winter, plain worktrees, …) — allocates clean                              │
│   environments by opaque id; acquired = clean by contract (D-021)                                │
│                                                                                                  │
└─────────────────────┬───────────────────────────────────────┬────────────────────────────────────┘
                      │                                       │
                      │  PULL (each tick, outbound-only):     │  git only:
                      │  chunks + node envelopes,             │  push branch artifacts (D-026);
                      │  answers, runner state in;            │  the hub's deliver node merges
                      │  transitions, artifacts,              │  them via its merge queue (D-030)
                      │  questions out                        │
                      ▼                                       ▼
        ┌──────────────────────────────────────┐            ┌────────────────────────────────────┐
        │ blizzard-hub — the work orchestrator │            │ forge / git remotes                │
        │ (colocated in the mvp solo setup;    │            │ (GitHub in the reference binding)  │
        │ remote in milestone:centralized-hub │            └────────────────────────────────────┘
        └──────────────────────────────────────┘
```

## The runner↔hub topology (mvp colocated, team remote)

The separation exists from `milestone:mvp` (D-022): the runner always reaches the hub through the same outbound-only protocol. In the solo MVP setup the hub simply runs on the runner machine and serves its web app to a local browser (D-048); moving it to a VPS, and adding more runners — and the web app's remote reach and the chat bot — arrive with `milestone:centralized-hub`, changing nothing architecturally.

```text
   web app (browser / PWA)         chat bot (one-tap answers)
                ▲                             ▲
                │         HTTP + SSE          │
                └──────────────┬──────────────┘
                       ┌───────┴────────┐
                       │ blizzard-hub   │   facts only —
                       │ (colocated in  │   never code,
                       │  the mvp; VPS/ │   never transcripts
                       │  cloud later)  │
                       └───────▲────────┘
                               │ outbound-only: runners dial out,
                               │ nothing ever dials into a dev box
               ┌───────────────┴───────────────┐
               │                               │
     runner machine A                 runner machine B
     full winter workspace            plain single-repo worktrees
     (everything in the                (same diagram, different
      diagram above)                   workspace binding)
```

## Interaction rules

The diagrams encode rules that hold everywhere:

- **Only a daemon opens its own sqlite file (D-023).** The runner encapsulates its store behind its local API; the hub encapsulates the fleet facts behind HTTP/SSE. The `blizzard` CLI — shared by worker hooks and humans — is a pure client of those two APIs and never touches a database.
- **All work flows through the hub (D-024).** The PM binding feeds the hub; runners acquire chunks from the hub and never speak to the PM system. The only external system a runner touches directly is git remotes — branch-artifact pushes; delivery itself (merging, or opening the PR) executes at the hub through the forge (D-030).
- **Work moves on a graph (D-025/D-027).** The hub owns the workflow definitions and the transition record; the runner holding a chunk advances it through consecutive nodes without re-queuing, and every judgement and transition lands at the hub.
- **The runner store and the hub are different stores, and both exist from the MVP (D-022).** See the [division of truth](#division-of-truth) below. Wire-level protocol details are an [open question](../decisions/open-questions.md).
- **Environments are opaque ids.** The runner asks the workspace provider to acquire; it gets back an id (`feature-x7`, `alpha`, …), records the chunk→env binding as a fact in its store, and reports the route (chunk → runner → workspace → env) to the hub. An acquired environment is clean by contract (D-021).
- **Workers never talk to the hub.** Only the runner does, outbound-only. A worker's world is its environment, its hooks, and the CLI.
- **Humans plug in at three places:** the hub (`blizzard hub status` / `blizzard hub answer`, and the web app's observability and queue shaping — D-048; in the MVP, answering a question means going to the hub), session takeover (`blizzard runner takeover`, or the pasted resume command), and — from `milestone:centralized-hub` — the remote clients: the board's PWA reach and the chat bot.
- **Every external system sits behind a seam** (D-016): the work source (at the hub, D-024), the workspace provider, the harness, delivery (the deliver node's binding), and the human channel are all interfaces with the reference stack (GitHub, winter, Claude Code) as first bindings.

## Division of truth

Three tiers of truth, cleanly separated (D-022/D-024):

- **The PM system owns _what work is defined_** — the backing backlog (GitHub issues in the reference binding). Durable, shared, already where work is tracked.
- **The hub owns _the fleet's view of that work_** — chunks wrapping the PM items, workflow position, judgements, artifacts, questions, routes. Runners never talk to the PM system; the hub's PM binding ingests specific items by id and serves their contents pass-through (D-047) — nothing is written back to the PM item in the MVP.
- **The runner store owns _this machine's execution right now_** — leases, heartbeats, pids, env bindings, and the minting of leases. Fast, local, second-granular. Lease facts (chunk, epoch, runner) are the one thing that travels (D-044): each mint is reported to the hub, whose epoch fence and reassignment floor consume them; everything else never leaves the box.

The runner's PULL step (see [runner/loop.md](./runner/loop.md)) exchanges state between the lower two tiers: chunks and answers come down, transitions, artifacts, and questions go up.

## Store facts, derive status

The schema principle governing **both** daemons' stores — adopted deliberately from Agent Orchestrator's architecture:

> Store only durable **facts**. **Status is always derived** by query, never written as a column.

A fact is a thing that definitely happened at a definite time — a lease created, a heartbeat received, a transition recorded, a verdict parsed. A chunk's *status* — running, stalled, waiting-on-human, done, failed — is computed from facts, never stored. **Written status lies after a crash:** a process that wrote "running" and then died leaves a database that reports it is running forever. A status *derived* from "last heartbeat was 20 minutes ago and the pid is dead" tells the truth no matter how the process ended. This single decision is what makes crash recovery correct rather than aspirational (D-004).
