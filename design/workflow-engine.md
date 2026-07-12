# The workflow engine

How work moves from ideation to completion. A **chunk** of work travels a user-defined graph of **nodes**; at each node an agent (or a human, at a gate) does something to it; a **judgement** at the node's exit picks the outgoing edge; **artifacts** accumulate on the chunk as it goes. The graph is configured at the hub in YAML and ships in `milestone:mvp` with a default graph (D-025).

This engine is what the earlier drafts' fixed pipeline (plan → build → verify → review), the verify step, and the human-gate matrix collapse into: they are all just graph shapes now. Editing the graph *is* the HITL→HOTL dial.

## The chunk

A chunk is the hub's unit of orchestrated work — what the PM binding's items are wrapped into (D-024), what a runner acquires and holds, what travels the graph, and what artifacts attach to. Its durable state lives at the hub: current node, transition history, judgements, questions, artifacts.

**Identity is hub-native, with a PM back-reference:** a chunk carries a hub-minted id wrapping a reference to its PM item(s) — for the GitHub binding, the issue. D-006's original motivation stands: the solo→team transition stays a **sync-policy change, not a schema migration** — a chunk's PM referent is the same issue no matter whose fleet acquires it. The exact identity scheme, and the chunk/execution-unit vocabulary reconciliation, are an [open question](../decisions/open-questions.md).

## The graph

- **Defined at the hub, in YAML.** Graphs are hub configuration; runners never define workflow, they execute node-steps. The YAML schema itself is an [open question](../decisions/open-questions.md).
- **Immutable once created; chunks pin their graph (D-033/D-040).** A graph never changes — an edit mints a new standalone graph, correlated with its predecessor only by name; there is no graph-family entity. Every chunk records the graph it was minted under and travels it to completion, so no chunk is ever stranded at a node its graph no longer has. Moving in-flight work to another graph is an explicit **migration** operation, never an implicit effect of editing, with two modes (D-034): **deferred** (default) — the in-flight node finishes, and the chunk re-pins at its next transition, landing on the target graph's node of the same name; and **forced** — the hotfix path: the live lease is revoked by minting a new epoch, so the existing fence (D-007) rejects the abandoned work, and the current node is redone under the new graph. Standing drift lives on each graph as `auto_migrate` + `migration_target` (D-037/D-040), honored while the target is enabled (D-039). A migration is its own recorded fact, never a transition. The unmapped-node case (no same-named node in the target graph) is an [open question](../decisions/open-questions.md).
- **Nodes name lifecycle stations** — "build", "review", "deliver" — each carrying at least a prompt and a judgement spec. A node is *what should happen to the chunk now*, not how the agent internally decomposes it.
- **Edges are keyed by judgement outcome.** Build succeeded → review; review failed → back to build. Every outcome a node's judgement can produce must have an edge (a validation rule for the schema).
- **Cycles are the point.** The review-fail → build → review loop is the canonical shape; the graph is cyclical by design, with retry caps as the escape hatch into escalation.
- **Nodes are runner-executed or hub-executed.** Most nodes run on the runner holding the chunk; a **hub node** executes at the hub itself — the deliver node is the first (D-030).
- **Human gates are nodes — and a runner-config dial (D-032).** A gate is a node whose "judgement" is a person; a chunk parked there holds an open **Decision** — a durable multiple-choice row carrying the step's artifacts (D-045) — and derives `waiting_on_human` with its reap clock stopped, exactly like an ask (D-010/D-015 machinery reused); the person's choice resolves it, and the holding runner records the transition. Mechanically the two gate levers are one mechanism (D-045): a *graph* gate is the hub refusing a transition at a human-judged node, and a *runner-config* gate is the runner choosing to submit a Decision instead of a transition for a node it was configured to gate. Besides gate nodes in the graph, a runner can be **configured to impose a human gate on any node by node name** (D-041 — names span graphs by convention), so an individual operator dials their own HITL level without forking the fleet's graph. This retires the separate gate-matrix mechanism — D-014's autonomous baseline survives as *the default graph containing no gate nodes*.

## Execution: who does what (D-027)

The **hub owns the rules and the record**: it holds the graph definitions, and every judgement and transition is recorded there — the hub always knows which node every chunk is at.

The **runner holding a chunk advances it**: when a node-step finishes, the runner evaluates the judgement, applies the hub-defined transition rule, records the transition, and continues into the next node **without re-queuing** — same environment, artifacts already in the worktree, session resumable. Whether the next node gets a fresh agent or a resumed session is per-node territory ([open question](../decisions/open-questions.md)). Hub nodes are the exception: when the transition lands on one (deliver, D-030), the hub executes it — the runner holds the chunk's environments until the hub reports the outcome.

**Reassignment is a supported exception**, never the default — it forfeits session continuity, but nothing else. A chunk's artifacts are durable at the hub, so another runner reconstitutes it by fetching the branch pointers into a fresh environment; the hub supplies the chunk's last-known epoch as the floor above which the new runner mints its leases (D-035/D-044), so the previous runner's late submissions are fenced out. The branch on the forge may sit *ahead* of the last submitted artifact commit — in-progress work the previous worker pushed but never submitted — and the adopting runner may inspect that work and adopt it as a shortcut, or reset to the last submitted artifact and redo the node. Hub-side routing (pinning work to a runner by flag) has its natural home here and is explicitly out of the MVP (D-024).

When a runner picks up a node-step it receives a **node envelope**: the prompt, the node config, and the chunk's relevant artifacts. What exactly is in the envelope and how artifacts materialize into the environment is an [open question](../decisions/open-questions.md).

## Judgement (MVP: verdict + checks)

The judgement at a node's exit is, in the MVP:

1. the worker's structured **verdict** — missing verdict = failure, unchanged (D-009);
2. plus the node's configured **deterministic checks** (tests, lint, CI state).

The verdict is elicited by a dedicated **judgement prompt** (D-038): node prompting is two-phase — the pre-prompt (base prompt + taken-edge addendum) instructs the work, and when the worker declares done, the judgement prompt is delivered into the same session (via the adapter's resume-with-message operation) to draw out the structured verdict. Outcomes are reified **Choices** (D-042): the author writes only the judgement prose; the engine appends a generated elicitation tail from the node's choice set ("select exactly one and output it as `<Choice>{name}</Choice>`"), and edges key off choice ids — resolution checked at graph mint. Judgement prompt delivered, no parseable `<Choice>` back → failure. Judge-agent nodes — a fresh evaluator agent rendering the judgement — are a later node kind, not MVP scope. Humans judge only at gate nodes.

## The review node

The default graph's review station is **one node with one prompt** — blizzard builds no review machinery of its own. The node's prompt invokes the review tooling of the stack below the fleet layer (in the reference stack, winter-workflow's review engine and its axes); the reviewing agent runs the project's own checks and e2e flows inside the chunk's environment — where the environment's services are available to drive — and submits its findings as an **asset artifact**, which a *fail* judgement carries back into the build node's envelope. The review node is *data*, not engine: a prompt plus deterministic checks in the graph YAML, which is why `epic:review` needs no dedicated mechanism beyond the engine itself. The prompt's content is authored during implementation.

## Artifacts (D-026)

A node-step's output is submitted to the chunk as one or more artifacts, committed atomically with the step's transition (D-036) into the chunk's append-only artifact store — `{node}.{artifact-name}.{epoch}`, latest-by-epoch reads — and stored at the hub:

| Kind | What it is | Example |
|------|-----------|---------|
| **Pointer** | A reference to git work: branch name and/or commit hash, per repo. **Must be pushed to the forge before submission** — the hub stores a pointer that anyone with repo access can resolve, never a path into a runner's worktree. | A chunk touching five repos submits five branch pointers. |
| **Asset** | A stored value — usually text, in principle any blob. | A code-review findings document; a spike write-up; a markdown design note. |

Artifacts are the chunk's memory across nodes and the reason non-code work fits the same engine: a review node emits a findings asset; when the judgement routes the chunk back to build, the build node's envelope carries those findings as input. A chunk whose *whole purpose* is a review or a spike simply ends with assets instead of branch pointers.

The hub stores pointers and assets — never code, never transcripts (D-012 intact; a pointer to code is not the code). Size limits, blob storage, retention: [open question](../decisions/open-questions.md).

## Delivery is a hub node (D-025/D-030)

The default graph ends in a **deliver** node — the first **hub-executed node**. The chunk's branch artifacts are already pushed to the forge by the time the chunk reaches it (D-026), so the hub can land them without ever holding code: it merges to main (or opens the pull request, where a gate wants one) through an internal **merge queue** that processes one chunk at a time. The delivery lane's single-writer property therefore lives at the hub, where it can serialize deliveries across every runner in the fleet ([concurrency-model.md](./concurrency-model.md)); a submission carrying a stale epoch is rejected before it enters the queue, so a zombie's work never lands. Delivery being a node is what makes the gate story uniform — a team that wants human sign-off before merge inserts a gate node ahead of deliver, and nothing else changes. Merge-queue mechanics — conflict handling, PR mode, ordering — are deliberately stubbed: an [open question](../decisions/open-questions.md).

## The default graph

```text
            (review judged: fail — findings asset carried back)
          ┌──────────────────────────┐
          ▼                          │
     ┌─────────┐              ┌─────┴─────┐              ┌───────────┐
────▶│  build  │─────────────▶│  review   │─────────────▶│  deliver  │────▶ done
     └────┬────┘     pass     └─────┬─────┘     pass     └───────────┘
          │                         │
          └────────────┬────────────┘
                       │  retry cap exceeded
                       ▼
          needs-human escalation (carries the resume command)
```

Build → review → deliver, with the review-fail cycle back to build and bounded retries escaping to escalation. Build and review execute on the holding runner; deliver executes at the hub (D-030). This graph is `milestone:mvp`'s default YAML; it reproduces exactly the behavior the earlier drafts hard-coded. An illustrative (non-schema) YAML rendering of this shape, extended with a human gate: [hub/sample-graph.yaml](./hub/sample-graph.yaml).

## What this engine is not

Blizzard's graph governs a **chunk's lifecycle** — which station the work is at, who judges it, when it lands. It is *not* the task-layer decomposition of a feature into implementation phases; how the agent inside the build node organizes its own work (subagents, phases, skills) belongs to the workflow tooling below blizzard and to the harness. The mission's non-goal stands, one layer down.
