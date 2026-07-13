# Runner contracts

Every API surface the runner speaks, tracked explicitly so the element-to-element conversations are designed, not discovered during implementation. Each contract carries a **maturity** marker: `settled` (a decision backs it), `draft` (shape proposed here, not yet decided), `open` (named but unshaped). Contracts graduate by getting a decision-log entry.

The runner is the hub of the wheel: it is the *only* component that talks to all of these. Workers talk only to the runner's local API (via hooks) and their code; the hub talks only to runners and clients.

## 1. Runner local API — via the `blizzard` CLI · **draft** (routes in [api.md](./api.md))

Transport: `blizzard <verb>` (D-020) as a pure client of the runner's local API (D-023) — the runner store's sqlite is embedded in the daemon, and the runner reaches it in-process, not through this contract. The surface is resource-oriented with exactly two clients — worker hooks (heartbeats, asks) and the operator's local CLI verbs (status reads, declarative controls, selftests); the route table lives in [api.md](./api.md), the CLI verb surface in [cli.md](../cli.md). The earlier verb-table framing mixed in the loop's internal store writes (lease, bind, release, reap); those never cross the process boundary (D-023/D-049) and are not part of this contract. Open details: the wire transport (tracked in the runner↔hub protocol [open question](../../decisions/open-questions.md)) and exact CLI flags.

## 2. Runner ↔ Workspace provider · **settled shape (D-062/D-064)** (detail in [environments.md](./environments.md))

| Operation | In | Out | Notes |
|-----------|-----|-----|-------|
| `acquire` | chunk id, count, held env ids | `(env id, workdir)` pairs, or refusal | Provider is **allocation-stateless** (D-062): pool is its static config, it picks from pool minus the passed held set; every returned environment is **clean by contract** (D-021); the workdir may not exist yet under a lazy binding (the agent materializes it, D-053/D-063). Idempotent re-acquire is answered from the runner's own binding facts — the provider sees only genuinely-new allocations. All-or-nothing at the runner: on partial satisfaction, release and skip the chunk this tick. |
| `release` | env id | ack | No-op if unknown/already released. Cleaning happens on next acquire, not here. |

Packaging (D-064): a capability slot — reference bindings (plain git, winter) compiled into the `blizzard` binary, BYO providers as invoked executables behind a versioned exec protocol (argv verbs, JSON out).

Open details: capacity signaling (queryable pool size), env affinity hints, grow-on-demand, the exec wire schema.

## 3. Runner ↔ Harness adapter · **draft** (interface in [harness-adapters.md](../harness-adapters.md))

| Operation | In | Out | Notes |
|-----------|-----|-----|-------|
| `spawn` | env ids + workdirs, prompt, session id | pid | Pre-assigned session id; pid + process start time recorded as facts. The prompt is the hub's node envelope plus the runner's machine-local preamble — held env ids and workdirs (D-063); `BLIZZARD_ENV_IDS` joins the injected identity variables. |
| `resume` | env id, session id, message | pid | Automated resume-with-message (D-050): the judgement prompt (D-038), answer delivery ([ask-answer.md](../ask-answer.md)), and the CI feedback loop all deliver through it (`claude -p --resume <sid> "…"`). |
| `resume_cmd` | env id, session id | literal shell command | The *interactive* human-takeover command, for escalation records. |
| `verdict` | worker output | structured result | The agent half of the node judgement (D-025); missing verdict = failure (D-009); assessment payload via harness-native structured output (D-056). |

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

## 6. Operator ↔ Runner · **draft** (local half shaped in [api.md](./api.md))

Named capabilities (loop.md): pause (stop new leases, drain in-flight), start, and work selection — filter or pin what FILL may lease. The local path is declarative state on the runner singleton — `PATCH /runner {paused}`, mirroring the hub's `PATCH /runners/{id}` (D-043) so one control model serves both surfaces, and the local path keeps working while the hub is unreachable. Work-selection knobs take the same declarative shape post-MVP. Open: how the local flag and the hub registry flag compose into the effective state (the pause-reconciliation [open question](../../decisions/open-questions.md)).

## 7. Delivery · **open** — not a runner contract since D-030

Delivery is the deliver node of the workflow graph (D-025), executed **at the hub** through its merge queue with the epoch check (D-030; [concurrency-model.md](../concurrency-model.md)); baseline is merge to main, and a human gate is a node inserted ahead of deliver. The runner's part ends at pushed branch pointers and the transition into the deliver node. Tracked here so the wheel stays complete; the delivery-seam operations themselves (land a chunk's branch artifacts, open/track a PR, surface a gated chunk) are unshaped — see the delivery-mechanics [open question](../../decisions/open-questions.md).
