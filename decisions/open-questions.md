# Open questions

These are unresolved. They are recorded here so that a cold reviewer can see exactly where the design is still soft, and so that convergence on them is deliberate rather than accidental. When a question is resolved, it graduates into the [decision log](./log.md) as a dated entry and is removed from this file.

## Seam interface contracts

Decided (D-016): from `milestone:mvp`, all five seams (workspace, work source, harness, delivery, human channel) are interfaces, and no component works against a concrete binding. What remains unresolved is the shape of each contract — the operations, data shapes, and error semantics per seam — to be worked out as part of this discovery. Two seams have a head start: harness (`spawn` / `resume` / `resume_cmd` / `verdict` in [design/harness-adapters.md](../design/harness-adapters.md) — the resume-with-message gap is closed, D-050) and workspace (largely settled: acquire-by-chunk → opaque env id + workdir / release / clean-by-contract, D-021; allocation-stateless provider with runner-store-held allocation truth, D-062; env ids presented to the worker via the spawn preamble, D-063; capability-slot packaging with an exec protocol for BYO providers, D-064 — still unsketched: capacity signaling, env affinity, and the exec wire schema, all in scope for this discovery; per-counterpart maturity tracked in [design/runner/contracts.md](../design/runner/contracts.md)). The work-source seam plugs into the hub (D-024) and its model is settled (D-047: id-addressed ingestion, a pointer per item, pass-through reads, no MVP write-back) — the remaining surface details are tracked in the PM-ingestion question below; the delivery seam executes at the hub (D-030) with its mechanics now settled (FIFO queue D-057, conflict routing D-058, capability-gated PR merging D-059, no branch fencing D-060, external-merge detection via polling D-065, env hold-until-outcome D-066) — what remains for it here is only the per-operation contract shape, like the other seams.

## Workflow YAML schema (MVP-blocking)

Decided (D-025): work travels a hub-defined, cyclical graph of nodes configured in YAML, shipping in the MVP with a default graph. The schema itself is unshaped: the node fields (prompt, judgement spec, deterministic checks, edge conditions, human-gate marker, runner-vs-hub execution, per-node session freshness — D-054), graph-level validation (every judgement outcome has an edge, cycles are intentional, one entry node), and the prompt-composition shape sketched in [design/hub/sample-graph.yaml](../design/hub/sample-graph.yaml) — node base prompt + per-edge `prompt_addendum`, both as file references resolved and inlined at graph mint so immutability (D-033) holds. Gate surfacing is decided (D-052: a parked-at-gate chunk's open Decision reaches the human via both the CLI and the board, one hub route). Versioning is decided — graphs are standalone immutable entities, chunks pin their graph, migration is explicit, lineage is emergent from `migration_target` pointers (D-033/D-040) — and so are the migration modes (D-034: deferred name-matched migration at the next transition, `migrate --force` revoking the lease via a new epoch) and the unmapped-node case (D-051: fall back to the target graph's entry node). Intent storage is settled (D-037/D-040: request rows plus each graph's `auto_migrate` + `migration_target`, pending derived); still unshaped are the apply mechanics — how a pending migration reaches the holding runner (the runner's outbound state pull is the natural candidate — D-043) and whether the hub applies deferred migrations at transition ingestion. Related (D-032/D-041): the shape of the runner-side gate configuration — how an operator declares "gate every node named X" and how that composes with the graph's own gate nodes.

## Runner↔hub protocol details (MVP-blocking)

D-022/D-023 fixed the shape (hub authoritative for chunks, workflow state, artifacts, questions; the runner's embedded store authoritative for leases, heartbeats, pids, env bindings, with lease/epoch facts minted there and reported up as the fence's input — D-044; CLI a pure client of both APIs). D-024/D-027 fixed the flow (work acquired from the hub; transitions recorded to it). One property is settled within it: **per-runner flush is FIFO**, so the hub's fence input arrives ordered — a lease fact always precedes the transitions minted under it (D-044). Remaining wire-level details: the local API transport (unix domain socket vs localhost HTTP — constrained by the milliseconds-cheap hook path); whether a degraded read path exists while the runner daemon is down; event batching, dedup, and replay semantics for flush-after-outage (idempotent ingestion keys); and a runner-level liveness heartbeat to the hub so the board can show a runner offline (distinct from, and much slower than, the worker tool-call heartbeat).

## Local ask/answer fast path

How could a local developer quickly address ask/answer scenarios without having to go to the hub — is there a mechanism to explore here that would enable this? The protocol ([design/ask-answer.md](../design/ask-answer.md)) makes every question a hub row and every answer a hub write: even with a colocated hub (D-022) and the developer sitting at the runner machine, answering means CLI → hub → stream/poll → runner → resume. Candidate mechanisms: a runner-local answer verb that writes the answer as a runner-store fact and delivers immediately, syncing to the hub afterward; or `blizzard hub answer` short-circuiting delivery when the target session is on the same machine. The tension is authority — questions and answers are hub-owned facts (D-022/D-023) and the first-write-wins CAS lives at the hub, so any local path either forfeits that arbitration or needs a reconciliation story (what happens when a local answer and a remote board answer race?).

## Local vs hub pause reconciliation

Runner pause exists on two surfaces: the hub registry's `paused` flag (`PATCH /runners/{id}`, D-043) and the runner singleton's local flag (`PATCH /runner`, [design/runner/api.md](../design/runner/api.md)) — the local path must keep working while the hub is unreachable. Unresolved: how the two flags compose into the effective state. Simplest candidate: effective paused = local ∨ hub, each flag clearable only on the surface that set it; alternatives are last-write-wins with the runner syncing its local fact upward, or the local flag as a pure overlay the hub never learns of. Whatever wins must stay legible on the board (the hub can only render what it holds) and idempotent under replays and missed deliveries (D-043's rationale).

## Artifact storage details

Decided (D-026): pointers (branch/commit, pushed to the forge first) and assets (text or blobs) stored at the hub. Selection is also decided (D-036): artifacts live in an append-only KV keyed `{node}.{artifact-name}.{epoch}` with latest-by-epoch reads. Unresolved: asset size limits and storage location (sqlite rows vs files beside the DB), retention/GC policy, whether the hub verifies a pointer resolves before accepting it, and any constraints on the `{artifact-name}` vocabulary itself.

## Chunk identity and vocabulary

The hub wraps PM items in its own chunk structure (D-024), which points where D-006's issue-keyed task identity was already headed: a hub-native chunk id carrying an array of PM pointers (D-047), with the GitHub issue as the reference binding's referent. Unresolved: the exact identity scheme, and reconciling the vocabulary — "chunk" (what travels the graph, what a runner holds) vs. the planner's "execution unit" (a batch of tasks acquired all-or-nothing) vs. "task" (a PM item). The planner docs predate the graph and still speak in units.

## Batching × the workflow graph

The batching ladder (D-011) and the planner were designed against a flat queue; the graph changes the acquirable object from "a bundle of tasks" to "a chunk at a node." Parked until the engine shape settles: whether a batch is one chunk with many PM references, how per-member verdicts work at a node exit, and the per-member heartbeat payload schema. The PLAN phase re-enters the runner loop when this resolves.

## PM ingestion surface (MVP-blocking)

Decided (D-047): ingestion is explicit and id-addressed — a pointer per PM item, pass-through reads at the hub with hub-held per-vendor credentials, ephemeral chunks, and no MVP write-back. Unresolved: the intake surface itself (a `blizzard hub ingest` verb, the raw hub route, or both — and whether the MVP ships a canned agent-native ingest flow for the GitHub binding or only documents the pattern: an LLM with the PM system's own tooling supplies the specific ids); the pass-through read surface (route shape, and what beyond the body is exposed — comments, attachments, linked items); the pointer's exact scheme (interacts with the chunk-identity question above); per-vendor credential configuration (app registration, token storage at the hub); and discard mechanics (who may discard, and what happens to a discarded chunk's artifacts and history at the hub).

## Queue ordering and prioritization

Priority must actually steer what the fleet pulls next (`persona:product-manager`'s core need). With id-addressed ingestion (D-047) the hub never reads priority off the PM system — ordering is a hub-side property by construction. Unresolved: how the hub orders the ready queue (ingestion order as the default? an explicit rank set from the web app — D-048? aging, starvation protection), and whether runners can express preferences that interact with it.

## Web app controls and operator scope

The MVP web app is authless and single-operator (D-018/D-048), with prioritize and group settled as its controls, and gate-decision resolution settled too (D-052: resolvable from the board and the CLI, one hub route) — still open there: whether it also answers *questions* (`blizzard hub answer` is the MVP acceptance path) or renders them read-only, and the rank/group route shapes ([design/hub/api.md](../design/hub/api.md)). The remote slice adds the viewer/operator role split — unresolved: exactly which controls the operator role gets — answering questions is settled (D-015), but pause/resume of runners, requeueing a chunk, killing a stuck worker, and editing graph YAML are all candidates — and how each maps onto durable hub state that runners pull; pause/resume already has its shape (the runner's `paused` flag, D-043), the rest remain unmapped.

## Cost controls

An unattended overnight fleet has token-spend runaway as a first-order operational hazard, and nothing in the design bounds it. Unresolved: per-chunk and per-night budget caps, token/cost telemetry recorded as facts (per node-step, per chunk — `persona:harness-engineer`'s comparable-runs need), model routing by cost, and a kill-switch when spend exceeds a configured ceiling.

## Worker sandboxing and permissions

Workers run unattended and their output merges to main in the baseline — what a worker is *allowed to do* is undesigned. Unresolved: the permission mode workers run under (permission-skipped? allowlisted?), the blast radius of a compromised or confused agent (network access, credentials in the environment, force-push protection), and whether permission profiles are per-node config (a review node needs less than a build node).

## Transcript retention and forensics

The hub never holds transcripts (D-012), so they live only on runner machines — but `persona:harness-engineer`'s incident reconstruction needs them: the fact record explains the lifecycle, the transcript explains the reasoning. Unresolved: retention window and rotation on the runner, the access path during forensics (a `blizzard` verb? raw files?), and whether a transcript *pointer* (machine + session id) should be a hub-side fact so the board can link to it.

## Status-derivation event vocabulary (MVP-blocking)

D-004 says status is derived, never stored, and the design's prose now speaks in recorded facts — but the fact vocabulary itself is not yet canonical. Needed: the full list of **facts/events** each derived status computes from (ask recorded → `waiting_on_human`; answer delivered → running again; escalation recorded → `needs-human`; delivery landed → done; …), which store owns each event (runner vs hub), and the derivation queries — so no implementer improvises a status column.

## Identity & permissioning spike (post-MVP)

Decided (D-018): the first hub ships authless behind Tailscale, and auth is configurable because the hub may run locally or in the cloud. The spike itself is the open work: identity provider choice (Google OAuth2 or similar), how identities map onto the viewer/operator roles, per-user permissions, and the configuration shape that lets a local hub run with no auth while a cloud hub requires it.

## Lease / heartbeat constants

The concrete values for lease TTL, heartbeat interval, the default retry cap when a node's `retries` doesn't specify one (currently 2 — the node's own `retries.max` is authoritative, [design/domain/graph.md](../design/domain/graph.md)), and the tick interval (currently ~30s).
