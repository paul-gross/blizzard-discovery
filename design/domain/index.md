# Domain model

The app-wide models — what the [hub API](../hub/api.md) accepts and returns, and what both daemons' stores record facts about. These are the fleet-visible models; the machine-local execution facts (leases, heartbeats, pids, env bindings) are the [runner store](../runner/store.md)'s territory and never leave the box.

Like the API surface, this model is **draft**: the models and their relationships are backed by decisions, but only the question row's shape appears verbatim in the corpus; the rest of the properties are assembled from the decisions that imply them. Two principles govern every model:

- **Facts in, status derived** (D-004) — no model carries a stored status column; anything called a status is computed by query.
- **Never code, never transcripts** (D-012) — a git-commit reference is the closest any model gets to code. (D-026 calls git-commit artifacts "pointers"; this model uses the more concrete name.)

| File | Models | When to read |
|------|--------|--------------|
| [work.md](./work.md) | Status enum · Chunk · Transition · Decision · Migration request · Migration record | …you need the work lifecycle: what a chunk is, its derived statuses, how it moves through its graph, how a gate parks it (D-045), and how it migrates between graphs. |
| [graph.md](./graph.md) | Graph · Node · Edge · Judgement spec · Choice | …you need the workflow definition chunks travel: standalone immutable graphs with enabled flags and migration-target pointers, nodes, choice-keyed edges, judgement specs, and reified choices (D-025/D-033/D-039/D-040/D-042). |
| [artifacts.md](./artifacts.md) | Artifact · Merge queue entry | …you need chunk outputs — the `git_commit`/`asset` data format — and how they land through the merge queue. |
| [questions.md](./questions.md) | Question · Answer | …you need the ask/answer rendezvous rows behind [the protocol](../ask-answer.md). |
| [fleet.md](./fleet.md) | Runner · Route · Event | …you need the fleet registry (including the runner's `paused` state), chunk locators, or the event stream. |
