# Artifact models

Chunk outputs and their landing. Part of the [domain model](./index.md).

## The chunk artifact store

A chunk's artifacts live in an **append-only, versioned key-value store** (D-036):

```
{node}.{artifact-name}.{epoch}  →  artifact_id
```

- **Written with the transition, atomically.** A node-step's artifacts are committed in the same epoch-fenced write as its transition ([work.md](./work.md)) — there is no separate artifact submission. A rejected (stale-epoch) transition's artifacts never enter the store; a gate's artifacts arrive on its open **Decision** (D-045), the gate's counterpart of a transition.
- **Append, never overwrite.** Re-running a node adds new entries under its new lease's epoch (one lease per node-step attempt, D-035); earlier entries remain as history.
- **Read = latest wins.** Fetching `{artifact-name}` resolves to the entry with the highest epoch — e.g. `build.review-findings.23` shadows `build.review-findings.17` without deleting it.
- **`{node}` is the node *name*, not the `node_id`.** The name is the cross-version correlator ([graph.md](./graph.md)): after a migration, a re-run of `build` keeps appending to the same `build.*` series. The exact producing node is on the artifact itself (`produced_by.node_id`).

## Artifact

The entity an `artifact_id` points at: a chunk's durable output, stored at the hub, feeding later nodes' envelopes (D-026).

The domain model is a **discriminated union**: an interchangeable `(type, …)` entity whose remaining fields depend on the discriminator. Code works with the typed variants; the compact single-string form is a **storage model**, listed below, that the variants compress to and uncompress from at the serialization boundary.

**Common properties** (every variant):

| Property | Notes |
|----------|-------|
| `artifact_id` | What the store's key entries point at. |
| `type` | The discriminator: `git_commit` or `asset`. |
| `name` | The `{artifact-name}` component of the store key (D-036) — how later nodes select it. |
| `produced_by` | Provenance: `chunk_id` + `node_id` + `epoch` — a reference to the committing transition, making the artifact self-describing when dereferenced by bare id. See below. |

**`git_commit` variant:**

| Property | Notes |
|----------|-------|
| `repo` | The repository the commit lives in. |
| `branch_name` | The branch pushed to the forge **before** submission (D-026). |
| `commit_hash` | Always present, never a branch name alone — see below. |

**`asset` variant:**

| Property | Notes |
|----------|-------|
| `content` | Text (a review's findings, a spike write-up) or an arbitrary blob — size limits and blob storage are an [open question](../../decisions/open-questions.md). |

There is no artifact-level epoch (D-036): the fence is checked once, on the transition that commits the artifacts — the epoch in the store key is that transition's lease epoch, not a fencing field of the artifact.

**`produced_by`** carries the owning `chunk_id`, the exact **`node_id`** ([graph.md](./graph.md) — unique across all graphs, unlike the store key's `{node}` component, which is the node *name*), and the producing lease's `epoch`: one lease per node-step attempt (D-035), so the epoch disambiguates re-runs. The overlap with the store key cannot drift: artifact and KV entry land in the same atomic transition write (D-036).

**`commit_hash`** pins the state that was actually verified: branches move, so the hash accompanies the branch name in every git-commit artifact. The hash is authoritative — the branch name serves only to detect commits ahead of it, and there is deliberately no fencing at the branch ref: a zombie clobbering a branch can lose work, never land wrong work (D-060). The hub stores the reference, never the code (D-012). A chunk touching five repos submits five git-commit artifacts.

### Storage model

How an artifact is persisted — a flat row with the variant fields compressed into one string:

| Column | Notes |
|--------|-------|
| `kind` | The discriminator, verbatim from `type`. |
| `data` | One string, keyed by kind: `git_commit` → `<branch-name>:<commit-hash>` (e.g. `feature/ask-timeout:9f3c2ab`); `asset` → the raw content. |
| `repo` | `git_commit` only — not encoded in the `data` string. |
| `name` | Carried through unchanged. |
| `chunk_id`, `node_id`, `epoch` | The flattened `produced_by` reference. |

Every artifact domain entity must **compress losslessly to this row on serialization and uncompress back to its typed variant on deserialization** — the round trip is exact in both directions. The storage model is a private detail of the owning daemon's store (D-023: nothing but the daemon opens the sqlite; clients see only the API), so surfaces beyond the store speak the domain model.

## Merge queue entry

A chunk's serialized delivery fact (D-030): its git-commit artifacts awaiting landing, processed one chunk at a time after the epoch check. Ordering, batching, and conflict handling are an open question.
