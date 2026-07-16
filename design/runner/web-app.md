# The runner web app

A browser surface served by the runner's own `host` personality (`blizzard runner host`, D-061) — the local control plane in a tab. It exists because of a connectivity asymmetry that no hub surface can ever cross: the hub never dials into a runner (D-012/D-043), so runner-machine material — transcripts, leases, resume commands, local health — is structurally invisible to the board; but a browser *on* the runner machine (or its Tailnet) reaches both the runner's local API and the hub just fine. That browser is a client *of that machine* — the connection originates there rather than dialing in, which is why [architecture.md](../architecture.md)'s web-client rule admits it (D-106): the rule's content is that the hub's serving never reaches into a runner, not that no browser may speak to one. The runner web app exploits that asymmetry to be a **dual-API client**: local pages against the [runner local API](./api.md), fleet pages against the [hub's HTTP API + SSE](../hub/api.md) — the full board plus everything the board can never show, without leaving the machine. **Maturity: exploratory** — post-MVP by scope (the MVP's machine-local surface is `blizzard runner status` and its sibling verbs, [cli.md](../cli.md)); D-106 settles the surface's right to exist, and the open points below are real.

## One codebase, two thin apps — never a forked board

D-048 committed the hub board to "never a second app," and this surface keeps that promise at the level where it matters: **fleet views are one shared library, never forked**. The frontend is a single Angular workspace with two thin application entrypoints over shared library projects: the **hub app** composes the fleet-view libraries alone — that build *is* the board ([hub/web-app.md](../hub/web-app.md)) — and the **runner app** composes the same fleet-view libraries plus the local-panel library, with its API wiring pointed at both the local API and the hub's URL. Both builds ship as assets inside the one binary (D-061); `blizzard hub host` serves the hub app, `blizzard runner host` serves the runner app. The thin-entrypoint split keeps the board honest structurally — a fleet page exists once, in the shared library, so the runner app cannot fork it, and the hub build carries no runner-panel code. In the runner app, a fleet page may reach the hub **directly from the browser** or **through its own runner** — both are permitted, and the choice is a transport one, not an architectural one: a same-origin read through the runner needs no cross-origin grant, while a direct read needs no relay. Neither is the private side channel D-023 forbids, because the runner relays the hub's public API and never invents state — no private route, no second store, no fact the hub does not already serve. The runner app adds the local pages and controls below; it never forks the fleet ones.

The degraded mode falls out of the same split, and it is the web analog of the CLI rule ([cli.md](../cli.md)): with the hub unreachable, fleet pages banner and stale-out while every local page and control keeps working — pause, requeue, escalations, transcripts. The machine panel is precisely the part of the app that must not depend on the hub.

## The local pages — what the hub can never render

Every view reads existing local-API routes ([api.md](./api.md)), derived-only per D-004:

| View | Backed by |
|------|-----------|
| Machine status: runner id, workspace binding, effective pause, the three capacities with used/free, hub reachability + outbound-buffer depth, last tick | `GET /runner` |
| Leases with derived state — running, stale, parked-on-ask — and heartbeat freshness at second granularity | `GET /leases`, `GET /leases/{id}` |
| Held environments: env id ← chunk bindings, held-since, env-slot utilization | `GET /environments` |
| Escalations, each with its **literal resume command rendered copyable** — the panel's version of the escalation contract | `GET /escalations` |
| Machine-local open asks | `GET /asks?open=true` |
| Machine-local chunks: current node, lease, env bindings | `GET /chunks`, `GET /chunks/{id}` |
| **Transcript viewer** — see below | a new local route, not yet in [api.md](./api.md) |

### The transcript viewer is the core pitch

Transcripts live only on runner machines — the hub holds pointers and assets, never transcripts (D-012) — and `persona:harness-engineer`'s incident reconstruction needs them: the fact record explains the lifecycle, the transcript explains the reasoning. The transcript-retention [open question](../../decisions/open-questions.md) asks what the access path is; this surface is the proposed answer: the panel renders a chunk's session transcripts in place, next to the lease facts and judgements they explain. That needs a new local route (transcripts are harness files on disk, not store facts — the route shape, and whether the adapter seam mediates the file format, are unshaped), and it pairs naturally with the same question's other half: if the transcript *pointer* (machine + session id) becomes a hub-side fact, the board can deep-link a chunk's history straight into the holding runner's panel — reach permitting, which on a Tailnet (D-018) it is.

## The local controls

The controls are exactly the local-API write surface — nothing new is invented for the browser:

- **Pause / start** — `PATCH /runner`, the D-043 declarative pattern; works with the hub unreachable, the operator contract's standing requirement.
- **Requeue** — `POST /chunks/{id}/requeues`, the takeover hand-back, with the same `409`-while-session-open guard.
- **Selftest** — `POST /selftests`, the adapter-drift canary, with `GET /selftests/{id}` polling for the verdict.
- **Takeover — initiated, not executed.** Takeover is inherently a TTY operation: the CLI execs the adapter-composed interactive command in the operator's terminal, and the daemon never touches a TTY ([cli.md](../cli.md)). The panel renders a parked chunk's takeover affordance as the literal `blizzard runner takeover <chunk-id>` command, copyable — the browser hands the human to the terminal, it does not become one. A browser-embedded terminal is deliberately parked.

Fleet controls — answer, decide, prioritize, group — are **not** re-implemented here: the fleet pages are the board's pages, so those controls are the board's controls, hitting the same hub routes (D-048/D-052). One control model per fact owner: local facts get local controls, hub facts get hub controls, and the panel is merely the one tab that shows both.

## What is deliberately not here

- **No terminal in the browser** — takeover stays a CLI exec; the panel only surfaces the command.
- **No third store or private channel** — the panel reads the two existing APIs and nothing else; it adds at most new *routes* to the local API (transcripts), never new state. A fleet page reaching the hub through its own runner (D-106) is a relay of the hub's public API, not a channel of its own — the line is that the runner may *carry* hub state, never *mint* it.
- **Not an MVP surface** — the MVP local surface is the CLI; this arrives post-MVP, sequencing against `milestone:centralized-hub` unscheduled.

## Open points

- **Local API transport.** A browser client weighs on the socket-vs-localhost-HTTP question (the runner↔hub protocol [open question](../../decisions/open-questions.md)): the milliseconds-cheap hook path may keep the unix socket while the panel gets a localhost HTTP listener, or HTTP wins outright.
- **Cross-origin reach.** A fleet page reading the hub directly needs the hub to allow the cross-origin call (trivial while authless behind Tailscale, D-018; interacts with the identity spike once auth exists). D-106 gives the way out rather than the answer — relaying through the page's own runner is same-origin and needs no grant — so what wants deciding is which pages take which route, not whether the grant is possible.
- **Exposure.** Localhost-only vs Tailnet-wide serving — the D-018 perimeter pattern applies, but the panel exposes *controls* on the machine, so the default wants deciding.
- **Transcript route and retention** — the access-path half of the transcript-forensics [open question](../../decisions/open-questions.md) lands here; the retention/rotation half stays open.
