# Glossary

One line per term, for cold readers and cross-checking harnesses. The owning document has the full treatment.

| Term | Meaning |
|------|---------|
| **seam** | A named point where an external system plugs into blizzard: workspace, work source, harness, delivery, human channel. |
| **reference stack** | The first bindings of the seams: winter (workspace), GitHub (work source + gated delivery), Claude Code (harness). Bindings, not identity. |
| **gate** | An opt-in human sign-off: a gate node in the workflow graph, or a runner-config gate imposed on a node by node name (D-025/D-032/D-041). Adding/removing gates is the HITL→HOTL dial; the default graph has none — agents verify and merge to main. |
| **winter** | The polyrepo workspace framework; the reference binding for the workspace seam — owns worktrees, ports, and service orchestration. |
| **`persona:<slug>`** | Stable identifier for a persona defined in [product/personas.md](./product/personas.md); requirements cite personas by id. |
| **`epic:<slug>`** | Stable identifier for a feature epic defined in [product/epics.md](./product/epics.md); scope statements cite epics by id, optionally naming a slice. |
| **`milestone:<slug>`** | Stable identifier for a roadmap milestone (`milestone:mvp`, `milestone:centralized-hub`); a milestone is shippable and always spans more than one epic (D-029). |
| **slice** | A partial landing of an epic (e.g. `epic:hub`'s separation slice in `milestone:mvp`, its remote slice in `milestone:centralized-hub`). |
| **HITL / HOTL** | Human-in-the-loop (review every step) vs. human-on-the-loop (supervise outcomes); blizzard is `persona:senior-se`'s on-ramp from the former to the latter. |
| **feature environment** | Winter's isolation unit: one worktree per project repo on a shared branch, with its own ports and services. |
| **environment id** | The opaque string naming an execution environment (`alpha`, `feature-x7`), allocated by the workspace provider; the runner↔workspace contract and the routing fact key on it. |
| **workspace provider** | The per-runner binding of the workspace seam (formerly the "workspace integration link"): allocates/releases environments by id and guarantees an acquired environment is clean (D-021). |
| **harness** | A coding-agent CLI blizzard drives: Claude Code, OpenCode, or Codex. |
| **adapter** | The four-function shim (`spawn` / `resume` / `resume_cmd` / `verdict`, D-050) that translates one harness's CLI; deliberately dumb. |
| **runner store** | The runner's embedded database: a sqlite (WAL) database inside `blizzard-runner`, reached only through the runner's local API via the `blizzard` CLI (D-023/D-028). |
| **task / PM item** | A unit of backlog work in the backing project-management system (a GitHub issue in the reference binding); ingested by id into a chunk (D-024/D-047). |
| **PM pointer** | What a chunk holds per wrapped PM item (D-047): `{source, ref}` — the configured `[[pm_source]]`'s name plus the item's source-relative reference (D-104/D-105) — pass-through reads through that source's own credentialed binding; contents are never stored. |
| **chunk** | The hub's unit of orchestrated work: wraps PM item(s), travels the workflow graph, accumulates artifacts; what a runner acquires, holds, and advances. |
| **workflow graph** | The hub-defined YAML graph of nodes a chunk travels — cyclical by design; the default is build → review → deliver (D-025). |
| **node** | A station in a workflow graph ("build", "review", "deliver"): a prompt, config, and judgement spec. Runner-executed by default; gates and delivery are nodes too. |
| **hub node** | A node executed by the hub itself rather than a runner; the deliver node is the first (D-030). |
| **node envelope** | What a runner receives to execute a node-step: the node's prompt and config plus the chunk's relevant artifacts. |
| **judgement** | The evaluation at a node's exit that selects the outgoing edge — in the MVP, the worker's verdict, informed by the checks the worker ran in-session (D-025/D-077); engine-executed checks are post-MVP. |
| **artifact** | A node-step's durable output stored at the hub (D-026): a **pointer** (branch/commit, pushed to the forge first) or an **asset** (text or blob). |
| **execution unit** | Retired (D-076) — the planner docs' old term for an acquirable bundle. A batch is one **chunk** carrying many PM pointers; the parked planner docs get rewritten in chunk terms when batching lands. |
| **acquisition** | The hub granting a ready chunk to exactly one runner (D-024) — claim-by-route since D-080: the runner peeks, acquires environments, and posts the complete route; the hub accepts exactly one claim. The one point of genuine cross-runner contention, and where fleet exactly-once is upheld (D-049). Distinct from a **lease**: acquisition is who gets the chunk, the lease is the runner's local node-step record. |
| **lease** | One node-step execution attempt (D-035/D-082): a TTL lease minted by the runner — one lease = one **session**, whose spawn, judgement-resume, and answer-resume pids all run within it — renewed by heartbeat, carrying a fresh fencing epoch, kept dormant while the chunk parks on a human; reported to the hub as a lease fact (D-044). Single-writer bookkeeping, not a contended CAS (D-049) — the runner has no rival on its own machine; cross-runner arbitration is **acquisition** at the hub. |
| **node-step** | One execution of one node by one worker under one lease (D-035) — the unit ADVANCE judges; consecutive node-steps run under successive leases on the same runner (no re-queue, D-027). |
| **heartbeat** | Liveness+progress signal emitted as a side effect of the worker's tool use (`PostToolUse` hook) — no agent cooperation required. |
| **epoch (fencing token)** | Monotonic counter minted with each lease (D-035) and reported to the hub (D-044); the hub rejects stale-epoch transition submissions (D-007/D-036) so a zombie cannot clobber its successor. |
| **reap** | Expiring a lease whose holder is gone (stale heartbeat, or dead pid checked together with recorded process start time). Reap ends the attempt — retry or escalate — never by itself the chunk's tenure or its environments (D-082/D-083). |
| **verdict** | The structured judgement reply — a `<Choice>{name}</Choice>` selection plus assessment — elicited by the node's judgement prompt after the worker declares done (D-038/D-042); missing or unparseable = failure (unless an open ask parked the chunk). |
| **choice** | One reified outcome of one node's judgement (D-042): scoped to its judgement spec, its id keys exactly one edge; what the worker emits in `<Choice>{name}</Choice>` and what gates render as buttons. |
| **migration** | The explicit re-pin of a chunk from one immutable graph to another (D-034/D-037): a request (intent) and a record (applied fact) — deferred name-matched by default, forced via epoch revocation for hotfixes. Never a transition. |
| **decision** | A gate's parking row (D-045): a durable multiple-choice ask carrying the step's artifacts, written where a worker-judged node would write its transition; the human's choice resolves it into the transition. Pending derives — a decision no transition references. |
| **reassignment** | Moving a held chunk to another runner — a supported exception to runner-stickiness (D-027/D-035): the new runner rebuilds the environment from the chunk's branch pointers, mints leases above the hub-supplied epoch floor (D-044), and may adopt unsubmitted in-progress work found ahead of the last submitted artifact commit. |
| **supervisor** | The stateless deterministic reconciliation loop (systemd): REAP → PULL → FILL → ADVANCE each tick. |
| **merge queue** | The hub's single-writer delivery lane: the deliver node lands one chunk's branch artifacts at a time, epoch-fenced (D-030). |
| **escalation / `needs_human`** | A chunk parked for a human after bounded retries, carrying the literal session-resume command; closes by supersession — the next lease mint after requeue (D-067). |
| **takeover** | A human entering a parked chunk's session interactively — `blizzard runner takeover <chunk id>` records the fact and execs the adapter-composed resume command; hand-back is an explicit `blizzard runner requeue` ([design/cli.md](./design/cli.md)). |
| **hub** | The work-orchestrator daemon (`blizzard-hub`); never holds code or transcripts. The canonical responsibility list lives in [design/hub/index.md](./design/hub/index.md). |
| **coordinator** | The hub's singleton, runner-like internal executor of hub nodes (D-079): acquires the hub node-step, executes it (deliver first), and records the exit transition under a lease and epoch it mints itself. |
| **detach** | The superadmin's forcible release of a chunk from its runner (`POST /chunks/{id}/detach`, D-088): appends `route.released`, the chunk re-derives ready, and the old runner is fenced by the next claim's epoch floor. |
| **runner** | The machine-level daemon (`blizzard-runner`): the supervisor loop behind a local API, bound to one prepared workspace and registered with the hub; connects outbound-only. Its host is the "runner machine". |
| **web app / board** | The hub's web front (`epic:board`): hub-served from the MVP — fleet observability, ready-queue prioritization, chunk grouping (D-048); PWA reach and the viewer/operator roles arrive remote (`milestone:centralized-hub`). |
| **ask/answer** | The `blizzard runner ask` → park (`waiting_on_human`) → human answers → resume-with-answer protocol ([design/ask-answer.md](./design/ask-answer.md)). |
| **`waiting_on_human`** | A derived chunk condition — computed from an open ask or an unresolved gate decision (D-045), never stored (D-004): parked on a person, reap clock stopped. |
| **planner** | The optional LLM PLAN phase that shapes PM items into multi-pointer chunks (D-076); gated by a deterministic validator. Parked pending the batching × workflow-graph open question. |
| **conflict packing** | Putting predicted-conflicting tasks into one sequenced unit, converting lock contention into intra-agent ordering. |
| **`blizzard runner selftest`** | A trivial task run through each harness before unattended periods, to catch adapter drift early. |
