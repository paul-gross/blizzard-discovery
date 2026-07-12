# Epics

The registry of blizzard's feature epics. Each epic carries a stable identifier of the form `epic:<slug>`; scope statements everywhere else — the [MVP](./mvp.md)'s in/out-of-scope lists, the [roadmap](./roadmap.md)'s milestones, open questions, decision-log entries — cite epics by id rather than by prose, so scope claims stay traceable as wording drifts.

An epic is a capability area, not a version: several epics land partially (a *slice*) before they land fully, and the citing document states which slice it means (e.g. `epic:ask-answer` ships its hub slice in the MVP and its remote slice with `milestone:centralized-hub`). The [story map](./storymap.md) lays these epics out against the release order visually. An epic whose owning doc is marked *pending* has no design home yet — that column is the todo list of design work.

## MVP

The epics in `milestone:mvp` scope, ordered by build order per the [story map](./storymap.md). `epic:ask-answer`, `epic:hub`, and `epic:board` land here as their first slice; the remainder arrives post-MVP.

| Epic | Capability | Owning doc |
|------|------------|------------|
| `epic:hub` | The work-orchestrator daemon: the PM wrapper and queue (D-024), HTTP API + SSE, outbound-only runners, eventual reachability, the merge queue (D-030). Slices: **separation** (`milestone:mvp`: colocated hub + runner↔hub protocol, D-022), **remote** (`milestone:centralized-hub`: off-machine hub, multiple runners). | [design/hub/index.md](../design/hub/index.md) |
| `epic:store` | The runner's embedded database behind its local API: atomic node-step leases, epochs, the facts-only schema. | [design/runner/store.md](../design/runner/store.md) |
| `epic:supervisor` | The runner: the reconciliation loop (REAP / PULL / FILL / ADVANCE), crash recovery, environment leasing, the runner contracts. | [design/runner/index.md](../design/runner/index.md) |
| `epic:adapters` | Harness adapters (spawn / resume / verdict), hooks, version pinning, selftest, human-takeover flows. | [design/harness-adapters.md](../design/harness-adapters.md) |
| `epic:workflow` | The dynamic workflow engine: hub-defined YAML graphs, nodes and judgements, node envelopes, sticky advancement, artifacts (pointers + assets). | [design/workflow-engine.md](../design/workflow-engine.md) |
| `epic:review` | The default graph's review node: one prompt driving the stack's review engine plus e2e checks before a chunk advances. | [design/workflow-engine.md](../design/workflow-engine.md) |
| `epic:delivery` | The deliver node, hub-executed (D-030): the merge queue, epoch-fenced submission, merge-to-main baseline, PR mechanics. | [design/workflow-engine.md](../design/workflow-engine.md) |
| `epic:ask-answer` | Ask-and-exit, `waiting_on_human`, resume-with-answer. Slices: **hub** (`milestone:mvp`: question rows at the hub, `blizzard ask`/`blizzard answer`), **remote** (`milestone:centralized-hub`: fan-out + answers from board/chat clients). | [design/ask-answer.md](../design/ask-answer.md) |
| `epic:board` | The hub's web front. Slices: **local** (`milestone:mvp`: hub-served web app — fleet observability, ready-queue prioritization, chunk grouping — D-048), **remote** (`milestone:centralized-hub`: PWA reach, viewer/operator roles, auth). | [design/hub/web-app.md](../design/hub/web-app.md) |

## Post-MVP

The epics that arrive after `milestone:mvp` — the `milestone:centralized-hub` spoke first, then the roadmap's unscheduled Later pool.

| Epic | Capability | Owning doc |
|------|------------|------------|
| `epic:chat` | Telegram bot (D-031): question notifications with one-tap answers, delivery confirmations. | [design/ask-answer.md](../design/ask-answer.md) |
| `epic:batching` | Execution units, the heuristic packer, the LLM planner, conflict packing, batch failure semantics. | [design/planner.md](../design/planner.md) *(parked)* |
| `epic:gates` | Human gates: gate nodes in the graph plus runner-config gates by node name — the HITL→HOTL dial (D-025/D-032/D-041). | [design/workflow-engine.md](../design/workflow-engine.md) |
| `epic:migration` | Graph lifecycle operations: explicit migration (requests + records, D-034/D-037), auto-drift via `migration_target` (D-040), enable/disable (D-039), the `PATCH /graphs/{id}` metadata surface. Immutability and chunk pinning themselves are structural and ship with `epic:workflow` (D-033). | [design/domain/graph.md](../design/domain/graph.md) |
| `epic:config` | The configuration surface: every operational constant as config (the workflow YAML ships in the MVP), comparable-run tuning for `persona:harness-engineer`. | *(design doc pending; needs stated in [product/personas.md](./personas.md))* |
| `epic:ci-feedback` | Routing CI and review results back into the owning session, capped feedback rounds. | *(design doc pending; staged sketch in [research/hard-problems.md](../research/hard-problems.md))* |
| `epic:team` | Team mode: shared visibility and multi-operator policy over one hub-arbitrated fleet — cross-machine arbitration is the hub's queue (D-024), never forge-native acquisition. | *(design doc pending; roadmap sketch in [product/roadmap.md](./roadmap.md))* |
