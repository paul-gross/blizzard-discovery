# Workflow graph authoring schema

The canonical YAML schema for graph definitions (D-071). A definition is submitted via `POST /graphs` ([api.md](./api.md)), which validates it, inlines every file reference, and mints an immutable graph (D-033) — the stored definition carries text, never paths. The domain entities the schema compiles into are [domain/graph.md](../domain/graph.md); a conforming example is [sample-graph.yaml](./sample-graph.yaml).

## Top level

| Field | Required | Notes |
|-------|----------|-------|
| `name` | yes | Human label, non-unique — a correlator, like node names (D-040). |
| `entry` | yes | Exactly one entry node, named by key. |
| `nodes` | yes | Map of node name → node. Names are the cross-graph correlator (D-034/D-041) and the `{node}` component of the artifact key (D-036); map keys make them unique for free. |

The graph's operational metadata — `enabled`, `auto_migrate`, `migration_target` — is never authored here; it lives beside the definition and is set after mint via `PATCH /graphs/{id}` (D-037/D-039/D-040).

## Node

| Field | Required | Notes |
|-------|----------|-------|
| `executor` | no (default `runner`) | `runner` or `hub` — deliver is the first hub-executed node (D-030). |
| `prompt` | runner nodes | File reference to the base prompt — the node's invariant identity, opening the pre-prompt phase (D-038). Hub nodes carry none. |
| `checks` | no | List of deterministic commands (tests, lint) — the judgement's second half (D-025). |
| `produces` | no | Artifact names the node is expected to submit (D-026/D-036). |
| `session` | no (default `resume`) | `resume` or `fresh` — per-node session freshness (D-054); review-style nodes opt into cold eyes. |
| `judgement` | yes | See below. |
| `retries` | no | `max` (default from the constants) plus `exhausted: escalate` — `escalate` is the only exhaustion target in the MVP, and the escape hatch out of any cycle. |
| executor-specific fields | hub nodes | The deliver node's `mode: merge-to-main \| open-pr` (D-059); future hub node kinds document their own. |

## Judgement and choices (fused entries — D-071)

Each outcome is **one entry** carrying its description, its destination, and optionally the arrival context — so "every choice has an edge" is unrepresentable to violate, not a rule to check:

```yaml
judgement:
  prompt: ./prompts/build.judgement.md   # worker-judged only: authored prose (D-038);
                                         # the elicitation tail is engine-generated (D-042)
  choices:
    pass:
      description: The work meets this node's criteria and the checks are green.
      to: review
    fail:
      description: The work does not meet the criteria; the failure output is attached.
      to: build                          # self-loops and cycles are intentional
      prompt_addendum: ./prompts/build.from-build.md   # arrival context, appended to the
                                                       # target's base prompt (D-038)
```

- `by: human` beside `prompt` marks a gate node (D-032/D-041): no judgement prompt, the choices become the board's buttons (D-042/D-045).
- Hub-executed nodes have **machinery-defined outcomes with default routing**: deliver's are `landed → done` and `conflict →` the conventional `merge` graph, else this graph's entry node (D-058). Omitting `judgement` accepts the defaults; authoring a choice entry (name matching the machinery set) overrides that one outcome — e.g. inline conflict routing.
- `to` names a node of this graph or the reserved terminal **`done`**. Cross-graph movement is a migration, never an edge (D-040).
- At mint each entry is compiled into a reified Choice and Edge with exact ids ([domain/graph.md](../domain/graph.md)); the fusion is authoring surface only.

## Prompt files

Every `prompt`, `prompt_addendum`, and `judgement.prompt` is a file reference resolved relative to the YAML and inlined at mint (D-033) — editing a prompt file and re-submitting mints a new graph. Naming convention (convention, not schema): `prompts/{node}.md` for a base prompt, `prompts/{to}.from-{from}.md` for an addendum, `prompts/{node}.judgement.md` for a judgement prompt.

## Mint-time validation (D-071)

**Errors** (the definition is rejected):

- `entry` names exactly one existing node.
- Every choice entry has a `description` and a `to` that resolves to a node name or `done`.
- Worker-judged nodes have a `judgement.prompt`; human-judged nodes must not; hub-node choice entries name outcomes in the executor's known set.
- Every file reference resolves and inlines.
- `retries.exhausted`, when present, is `escalate`.

**Warnings** (minted, flagged):

- Nodes unreachable from `entry`.
- No path from `entry` to `done` — legal (retries escape every cycle to escalation) but suspicious.

There is no acyclicity check: cycles are intentional (D-025), bounded by each node's retry cap.
