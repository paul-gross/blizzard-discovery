# Open questions

These are unresolved. They are recorded here so that a cold reviewer can see exactly where the design is still soft, and so that convergence on them is deliberate rather than accidental. When a question is resolved, it graduates into the [decision log](./log.md) as a dated entry and is removed from this file.

## Seam interface contracts

Decided (D-016): from `milestone:mvp`, all five seams (workspace, work source, harness, delivery, human channel) are interfaces, and no component works against a concrete binding. What remains unresolved is the shape of each contract — the operations, data shapes, and error semantics per seam — to be worked out as part of this discovery. Two seams have a head start: harness (`spawn` / `resume` / `resume_cmd` / `verdict` in [design/harness-adapters.md](../design/harness-adapters.md) — the resume-with-message gap is closed, D-050; session identity and the conformance-suite requirement are settled, D-092) and workspace (largely settled: acquire-by-chunk → opaque env id + workdir / release / clean-by-contract, D-021; allocation-stateless provider with runner-store-held allocation truth, D-062; env ids presented to the worker via the spawn preamble, D-063; capability-slot packaging with an exec protocol for BYO providers, D-064 — still unsketched: capacity signaling, env affinity, and the exec wire schema, all in scope for this discovery; per-counterpart maturity tracked in [design/runner/contracts.md](../design/runner/contracts.md)). The work-source seam plugs into the hub (D-024) and its model and surface are settled (D-047: id-addressed ingestion, pass-through reads, no MVP write-back; D-074: intake verb + route, body-and-comments reads, token credentials, discard/stop split; D-104/D-105/D-106/D-107: `{source, ref}` pointers resolved through config-defined named sources, ingest-time source resolution, and binding-owned rendering); the delivery seam executes at the hub (D-030) with its mechanics now settled (FIFO queue D-057, conflict routing D-058, capability-gated PR merging D-059, no branch fencing D-060, external-merge detection via polling D-065, env hold-until-outcome D-066) — what remains for it here is only the per-operation contract shape, like the other seams.

## Local ask/answer fast path

How could a local developer quickly address ask/answer scenarios without having to go to the hub — is there a mechanism to explore here that would enable this? The protocol ([design/ask-answer.md](../design/ask-answer.md)) makes every question a hub row and every answer a hub write: even with a colocated hub (D-022) and the developer sitting at the runner machine, answering means CLI → hub → stream/poll → runner → resume. Candidate mechanisms: a runner-local answer verb that writes the answer as a runner-store fact and delivers immediately, syncing to the hub afterward; or `blizzard hub answer` short-circuiting delivery when the target session is on the same machine. The tension is authority — questions and answers are hub-owned facts (D-022/D-023) and the first-write-wins CAS lives at the hub, so any local path either forfeits that arbitration or needs a reconciliation story (what happens when a local answer and a remote board answer race?).

## Local vs hub pause reconciliation

Runner pause exists on two surfaces: the hub registry's `paused` flag (`PATCH /runners/{id}`, D-043) and the runner singleton's local flag (`PATCH /runner`, [design/runner/api.md](../design/runner/api.md)) — the local path must keep working while the hub is unreachable. Unresolved: how the two flags compose into the effective state. Simplest candidate: effective paused = local ∨ hub, each flag clearable only on the surface that set it; alternatives are last-write-wins with the runner syncing its local fact upward, or the local flag as a pure overlay the hub never learns of. Whatever wins must stay legible on the board (the hub can only render what it holds) and idempotent under replays and missed deliveries (D-043's rationale).

## Artifact storage details

Decided (D-026): pointers (branch/commit, pushed to the forge first) and assets (text or blobs) stored at the hub. Selection is also decided (D-036): artifacts live in an append-only KV keyed `{node}.{artifact-name}.{epoch}` with latest-by-epoch reads. Unresolved: asset size limits and storage location (sqlite rows vs files beside the DB), retention/GC policy, whether the hub verifies a pointer resolves before accepting it, and any constraints on the `{artifact-name}` vocabulary itself.

## Batching × the workflow graph

The batching ladder (D-011) and the planner were designed against a flat queue; the graph changes the acquirable object from "a bundle of tasks" to "a chunk at a node," and the vocabulary is now settled (D-076: a batch is one chunk carrying many PM pointers; "execution unit" is retired). Parked until batching is taken up: how per-member verdicts work at a node exit, the per-member heartbeat payload schema, and rewriting the planner docs in chunk terms. The PLAN phase re-enters the runner loop when this resolves.

## Queue ordering and prioritization

Priority must actually steer what the fleet pulls next (`persona:product-manager`'s core need). With id-addressed ingestion (D-047) the hub never reads priority off the PM system — ordering is a hub-side property by construction. Unresolved: how the hub orders the ready queue (ingestion order as the default? an explicit rank set from the web app — D-048? aging, starvation protection), and whether runners can express preferences that interact with it.

## Web app controls and operator scope

The MVP web app is authless and single-operator (D-018/D-048), with prioritize and group settled as its controls, and gate-decision resolution settled too (D-052: resolvable from the board and the CLI, one hub route) — still open there: whether it also answers *questions* (`blizzard hub answer` is the MVP acceptance path) or renders them read-only, and the rank/group route shapes ([design/hub/api.md](../design/hub/api.md)). The remote slice adds the viewer/operator role split — unresolved: exactly which controls the operator role gets — answering questions is settled (D-015), but pause/resume of runners, requeueing a chunk, killing a stuck worker, and editing graph YAML are all candidates — and how each maps onto durable hub state that runners pull; pause/resume already has its shape (the runner's `paused` flag, D-043), the rest remain unmapped.

## Cost controls

An unattended overnight fleet has token-spend runaway as a first-order operational hazard, and nothing in the MVP bounds it — `epic:cost` (D-087) owns the design. Unresolved: per-chunk and per-night budget caps, token/cost telemetry recorded as facts (per node-step, per chunk — `persona:harness-engineer`'s comparable-runs need), model routing by cost, and a kill-switch when spend exceeds a configured ceiling.

## Worker sandboxing and permissions

The MVP stance is settled (D-087): workers run permission-skipped, configured per runner — `epic:security` owns the real design. Unresolved there: permission profiles beyond skip-everything, the blast radius of a compromised or confused agent (network access, credentials in the environment, force-push protection), and whether permission profiles are per-node config (a review node needs less than a build node).

## Transcript retention and forensics

The hub never holds transcripts (D-012), so they live only on runner machines — but `persona:harness-engineer`'s incident reconstruction needs them: the fact record explains the lifecycle, the transcript explains the reasoning. Unresolved: retention window and rotation on the runner, the access path during forensics (a `blizzard` verb? raw files?), and whether a transcript *pointer* (machine + session id) should be a hub-side fact so the board can link to it.

## Identity & permissioning spike (post-MVP)

Decided (D-018): the first hub ships authless behind Tailscale, and auth is configurable because the hub may run locally or in the cloud. The spike itself is the open work: identity provider choice (Google OAuth2 or similar), how identities map onto the viewer/operator roles, per-user permissions, and the configuration shape that lets a local hub run with no auth while a cloud hub requires it.

## Lease / heartbeat constants

The concrete values for lease TTL and the reap staleness threshold (conservative by design — ~1–2 h as the starting default, because heartbeats ride tool calls and the threshold is bounded below by the longest tool call a healthy worker makes), heartbeat interval, the default retry cap when a node's `retries` doesn't specify one (currently 2 — attempt failures only, D-078; the node's own `retries.max` is authoritative, [design/domain/graph.md](../design/domain/graph.md)), the tick interval (currently ~30s — the judgement resume and advancement should land within ~1 tick, well inside the reap threshold), and the runner-liveness heartbeat cadence and offline staleness threshold (D-070 — much slower than the worker heartbeat; ~60s and ~3 missed intervals as starting points).
