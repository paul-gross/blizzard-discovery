# Workflow graph models

The definition chunks travel — the engine itself is [workflow-engine.md](../workflow-engine.md). Part of the [domain model](./index.md). Graphs are hub-owned (D-025); the YAML authoring shape is sketched in [sample-graph.yaml](../hub/sample-graph.yaml), and its schema remains an [open question](../../decisions/open-questions.md).

There is **no graph family** (D-040): graphs are standalone immutable entities, and any graph may migrate to any graph so long as the node mapping gets the chunks over. What looks like "versions of a workflow" is emergent — a chain of `migration_target` pointers between graphs that happen to share a name.

## Graph

Two parts under one `graph_id`: the immutable **definition** (D-033 — every edit mints a new graph; an existing definition never changes) and the mutable **operational metadata** beside it — `enabled`, `auto_migrate`, `migration_target`. The metadata is the graph's only mutable surface, set via `PATCH /graphs/{id}` ([api.md](../hub/api.md)) as appended facts the flags derive from (D-004/D-039); it is never authored inside the definition YAML.

| Property | Notes |
|----------|-------|
| `graph_id` | Universally unique; minted at creation. |
| `name` | Human label, non-unique — successors typically keep it. Purely a correlator, exactly as node names are (D-034/D-040). |
| enabled | *Metadata.* **Derived** from the newest enable/disable fact (D-039) — see below. |
| `auto_migrate` | *Metadata.* `deferred` or `off` (D-037): the standing drift intent for chunks pinned to *this* graph — flippable live via PATCH. |
| `migration_target` | *Metadata.* A `graph_id`: where this graph's chunks drift when `auto_migrate` is `deferred` — see below. |
| definition | The fully resolved immutable definition — see below. |
| `created_at` | When the graph was minted. |

**enabled** (D-039) gates exactly one thing: being auto-migrated *to*. While a graph is disabled, no chunk auto-migrates into it — and nothing else changes: chunks pinned to it continue, explicit requests may still target it, the definition is untouched. Graphs mint enabled. Enable/disable are appended operational facts beside the immutable definition; the flag derives (D-004).

**`migration_target`** points anywhere: the ordinary case is the graph's own successor (the re-published edit), but pointing into a differently-named graph retires this workflow into another. A chunk's pending auto-migration is derived, never stored (D-037): its pinned graph has `auto_migrate: deferred` and a `migration_target` that is enabled. Chains resolve **one hop per transition** (D-040): if the target itself points onward, the chunk takes the next hop at its next transition — no transitive resolution, and the path self-corrects as pointers change.

**definition** is stored fully resolved: prompt file references in the authoring YAML (`./prompts/build.md`) are inlined when the graph is minted, so the graph carries text, never paths — editing a prompt file and re-submitting mints a new graph.

## Node

One station in one graph. A node belongs to exactly one immutable graph; same-named nodes in different graphs are **distinct nodes correlated only by name** — that name correlation is what deferred migration's matching keys on (D-034); a migration whose destination name has no match in the target graph lands the chunk at the target's entry node (D-051). Exact references (transitions, artifact provenance) carry `node_id`; cross-graph continuity (migration matching, the artifact store key) uses the name.

| Property | Notes |
|----------|-------|
| `node_id` | Universally unique, minted with the graph — pins the graph and the name at once. |
| `graph_id` | The owning graph. |
| `name` | The cross-graph correlator; the `{node}` component of the artifact store key (D-036); and what runner-side gate configuration selects on (D-032/D-041 — "gate every node named `deliver`"). There is no `type` field: what type would encode is structural (D-041). |
| executor | `runner` or `hub` — deliver is the first hub-executed node (D-030). |
| prompt | The base prompt: the node's invariant identity, opening the pre-prompt phase (D-038). |
| checks | Deterministic commands (tests, lint) — the judgement's second half (D-025). |
| judgement | How the exit judgement is rendered — see Judgement spec. |
| retries | Max attempts plus the exhaustion target (escalation) — the escape hatch out of any cycle. |
| produces | Artifact names the node is expected to submit (D-026/D-036). |
| session | Fresh or resumed agent session for this node's steps — defaults to resume (D-054); review-style nodes opt into cold eyes. |

## Edge

A directed, outcome-keyed connection between two nodes of the **same graph**. Validation rules: every outcome a node's judgement can produce has an edge, exactly one entry node, cycles are intentional.

| Property | Notes |
|----------|-------|
| `from_node_id` | The source node. |
| `choice_id` | The Choice that selects this edge — a reference into the source node's judgement spec (D-042); resolution is checked at mint, so an edge can never dangle. |
| `to_node_id` | The target node — always within the same graph; cross-graph movement is a Migration ([work.md](./work.md)), never an edge. |
| prompt_addendum | Arrival-context text appended to the target's base prompt in the pre-prompt phase (D-038); authored as a file reference, inlined at mint. |

## Judgement spec

Per-node configuration for how the exit judgement is rendered (D-025/D-038/D-042).

| Property | Notes |
|----------|-------|
| judged by | `worker` (default) or `human` — the structural marker of a gate node (D-032/D-041), not a type label. |
| prompt | Worker-judged only: the authored judgement prose, delivered into the session when the worker declares done (D-038). Authored prose **only** — the elicitation tail is engine-generated, see below. |
| choices | The Choice entities this judgement can produce — each keys exactly one outgoing edge. |

The **elicitation tail** is generated by the engine from the choice set and appended to the authored prompt: "Here are your available options: *(name — description, per choice)*. Select exactly one and output it as `<Choice>{name}</Choice>`." It is identical in shape across every node of every graph — no author ever hand-writes verdict-emission instructions — and a reply with a missing or unparseable `<Choice>` is a missing verdict → failure (D-009).

Human-judged (gate) nodes carry no judgement prompt — a person renders the judgement by picking from the **same choices**, which are what the board and chat bot render as buttons (the ask/answer `options` machinery reused). Hub-executed nodes' choices are selected by machinery (`landed`, `conflict` at deliver).

## Choice

One selectable outcome of one node's judgement (D-042). Scoped to its judgement spec — never a global registry: `pass` in build and `pass` in review are different choices that happen to share a name, the same ids-exact/names-correlate philosophy as nodes and graphs (D-040/D-041).

| Property | Notes |
|----------|-------|
| `choice_id` | Minted with the graph — the exact reference edges carry. |
| `name` | Short author-facing key (`pass`, `fail`, `approve`) — unique within the node's choice set; what the worker emits in `<Choice>{name}</Choice>`, resolved against the node's choices. |
| description | Human-facing prose — sharpens the worker's judgement in the elicitation tail, and is the button text at gates. |
