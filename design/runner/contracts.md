# Runner contracts

Every API surface the runner speaks, tracked explicitly so the element-to-element conversations are designed, not discovered during implementation. Each contract carries a **maturity** marker: `settled` (a decision backs it), `draft` (shape proposed here, not yet decided), `open` (named but unshaped). Contracts graduate by getting a decision-log entry.

The runner is the hub of the wheel: it is the *only* component that talks to all of these. Workers talk only to the runner's local API (via hooks) and their code; the hub talks only to runners and clients.

## 1. Runner local API — via the `blizzard` CLI · **draft**

Transport: `blizzard <verb>` (D-020) as a pure client of the runner's local API (D-023) — the runner store's sqlite is embedded in the daemon, and the runner reaches it in-process, not through this contract. The verbs below are the API surface; which are hook/human-facing and which are runner-internal operations falls out of the facts-placement [open question](../../decisions/open-questions.md). All verbs are idempotent or CAS-atomic.

| Operation | In | Out | Notes |
|-----------|-----|-----|-------|
| `blizzard lease` | runner id | chunk `{chunk_id, node, envs_wanted}` or nothing | Atomic CAS over the chunks acquired from the hub; epoch minted. One lease per node-step execution attempt (D-035). Runner-internal. |
| `blizzard bind` | chunk id, env ids | ack | Records the chunk→env binding fact (D-021), same transaction discipline as lease. |
| `blizzard heartbeat` | lease id | ack | Also invoked by worker hooks on every tool call. |
| `blizzard release` | lease id, reason | ack | Normal completion, requeue, or escalation. |
| `blizzard reap` | — | expired leases | Pid + process-start-time check; retry-or-escalate policy applied. |
| `blizzard status` | filters | derived state (JSON) | A `SELECT`; never a stored status column (D-004). Machine-local view — fleet-wide status is a hub query. |
| `blizzard ask` / `blizzard answer` | question / answer | ack | Ask-and-exit protocol (D-010, D-015); the runner forwards asks to the hub, where the question row lives. |
| `blizzard selftest` | harness | pass/fail | Adapter-drift canary. |

Open detail: exact verb set and flags (tracked in [open questions](../../decisions/open-questions.md)).

## 2. Runner ↔ Workspace provider · **draft** (shape in [environments.md](./environments.md))

| Operation | In | Out | Notes |
|-----------|-----|-----|-------|
| `acquire` | chunk id, count | env ids, or refusal | Idempotent per chunk id; every returned environment is **clean by contract** (D-021). All-or-nothing at the runner: on partial satisfaction, release and skip the chunk this tick. |
| `release` | env id | ack | No-op if unknown/already released. Cleaning happens on next acquire, not here. |

Open details: capacity signaling (refuse-on-acquire vs queryable free-count), env affinity hints, grow-on-demand.

## 3. Runner ↔ Harness adapter · **draft** (interface in [harness-adapters.md](../harness-adapters.md))

| Operation | In | Out | Notes |
|-----------|-----|-----|-------|
| `spawn` | env ids, prompt, session id | pid | Pre-assigned session id; pid + process start time recorded as facts. |
| `resume_cmd` | env id, session id | literal shell command | The *interactive* human-takeover command, for escalation records. |
| `verdict` | worker output | structured result | The agent half of the node judgement (D-025); missing verdict = failure (D-009). |

**Known gap:** the interface has no automated resume-with-message operation, though answer delivery ([ask-answer.md](../ask-answer.md)), the CI feedback loop, and the judgement prompt (D-038 — delivered into the session when the worker declares done) all need one (`claude -p --resume <sid> "…"`). With three consumers the gap is clearly required, not speculative. The per-harness comparison already carries the data (the "automated follow-up" row); the fourth function is to be added when the adapter contract is next revised.

## 4. Hub ↔ Work source (PM binding) · **open** — not a runner contract

Moved hub-side by D-024: the PM binding (GitHub issues in the reference stack) plugs into the hub, which ingests specific items by id into chunks and serves their contents pass-through — no write-back in the MVP (D-047). Runners never speak to the PM system. Tracked here only so the wheel stays complete; the contract's owner is [hub/index.md](../hub/index.md); the intake surface and pointer scheme are tracked in the PM-ingestion and chunk-identity open questions.

## 5. Runner ↔ Hub · **draft** (from the MVP, colocated in the solo setup — D-022; protocol in [hub/index.md](../hub/index.md))

All connections outbound from the runner (D-012); the hub is eventually-reachable — buffer and flush through outages.

| Operation | Direction | Notes |
|-----------|-----------|-------|
| register | runner → hub | runner id + workspace id; makes the runner visible on the board. |
| acquire chunks | runner → hub | ready chunks with their **node envelopes** (prompt, node config, relevant artifacts) — the PULL step (D-024). |
| transition record | runner → hub | judgement + node transition + the step's artifacts, one atomic write (D-027, D-036): git-commit artifacts pushed to the forge first (D-026); idempotent; carries the lease's epoch — the hub rejects stale ones (D-007/D-030). |
| event push | runner → hub | leases, routes ("chunk C, runner R, workspace W, env E" — D-021), verdicts, questions; batched, durable rows. |
| answer / state pull | hub → runner (over the runner's outbound poll/SSE) | delivered answers, plus the runner's declarative operational state — `paused` now, routing knobs post-MVP (D-043). |

## 6. Operator ↔ Runner · **open**

Named capabilities (loop.md): pause (stop new leases, drain in-flight), start, and work selection — filter or pin what FILL may lease. Unshaped: whether these are `blizzard` verbs against the runner's local API, runner config reloads, or hub-relayed controls (all three may coexist; the local path must keep working while the hub is unreachable).

## 7. Delivery · **open** — not a runner contract since D-030

Delivery is the deliver node of the workflow graph (D-025), executed **at the hub** through its merge queue with the epoch check (D-030; [concurrency-model.md](../concurrency-model.md)); baseline is merge to main, and a human gate is a node inserted ahead of deliver. The runner's part ends at pushed branch pointers and the transition into the deliver node. Tracked here so the wheel stays complete; the delivery-seam operations themselves (land a chunk's branch artifacts, open/track a PR, surface a gated chunk) are unshaped — see the delivery-mechanics [open question](../../decisions/open-questions.md).
