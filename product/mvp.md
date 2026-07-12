# MVP

The MVP is the [roadmap](./roadmap.md)'s `milestone:mvp` — the local fleet — stated here as a testable acceptance journey plus an explicit cut list. The roadmap sequences milestones; this document defines *done* for the first one. It serves exactly one persona: [`persona:application-architect`](./personas.md).

The MVP has a second, self-referential goal: it must be enough of a platform that **blizzard's own future development becomes autonomous**. From the moment `milestone:mvp` passes the journey below, blizzard's backlog is fed to the blizzard fleet and the fleet builds its successor milestones — blizzard is its own first customer. A gap that blocks that dogfooding loop is MVP scope by definition — this document (and its cut list) is amended to admit it, so the cut list stays the single honest record of what is out.

## The acceptance journey

> I pick five small issues from across my workspace's repos and hand their ids to the hub, which mints a chunk for each. On the hub's web app I group two related ones into a single chunk and move the riskiest to the top of the queue. The runner picks them up and I close the laptop lid on the fleet and come back the next morning.
>
> I find: the chunks that succeeded merged to main — each one walked the default graph (build → review → deliver), agent-built, agent-reviewed, agent-tested, no human in the loop. The hub's web app shows every chunk's history: which nodes it visited, the review that failed once and looped back to build carrying the findings, and the artifacts — the branch pointers that were merged, the review notes. One agent hit a genuinely undecidable choice overnight: it asked its question and went dormant. `blizzard status` — the fleet verb, a hub query — shows it waiting; I answer at the hub — `blizzard answer` — and the agent resumes exactly where it left off, no takeover needed. For anything that actually failed, a `needs-human` escalation carrying the literal command that drops me into the stuck agent's session (`cd <env> && claude --resume <sid>`). Nothing was worked twice. No environment is orphaned. `blizzard status` tells the truth about every chunk — including the ones whose worker died — because status is derived from facts, not written by the deceased.
>
> At some point in the night the machine rebooted. It didn't matter: the supervisor and the colocated hub came back under systemd, the supervisor reaped the stale leases, re-read the environment bindings from its store, and continued — every chunk still at exactly the node the hub last recorded.

If any sentence of that journey fails, the MVP is not done.

## Acceptance criteria

Concrete and testable, mostly restating the journey as assertions:

1. **Issues in, landed work out.** An issue ingested by id becomes a chunk holding its pointer (D-024/D-047), acquired by the runner, worked in a clean environment from the workspace provider (winter binding) — the issue's contents fetched through the hub's pass-through API, never directly — walked through the default graph, and merged to main by the hub-executed deliver node through its merge queue (D-030). The environment is released. Nothing is written back to the issue (D-047), and the runner never talks to GitHub's issue layer — PM credentials exist only at the hub.
2. **Exactly-once assignment.** Two concurrent acquisition attempts on one chunk: exactly one wins, always — arbitrated at the hub's queue (D-024), never in a runner store. A loser that later tries to submit (or a runner whose acquisition was expired and reassigned) is fenced at the hub by a stale epoch (D-007/D-044). Within a single runner there is no rival to arbitrate: the single-writer supervisor (D-023) leases each node-step to a worker slot as crash-safe bookkeeping, not a contended CAS.
3. **Zombie fencing.** A transition carrying an epoch older than the chunk's latest lease fact is rejected at the hub (D-044) — a reaped lease's buffered or delayed submission can neither advance the chunk nor enter the merge queue.
4. **`kill -9` at every step boundary** (supervisor and worker, mid-FILL, mid-ADVANCE, mid-tick) recovers to a correct state with no duplicate env, no double delivery, no lying status. This is a tested operation, not a hope.
5. **Fail-safe verdicts.** A node-step whose judgement prompt elicits no parseable verdict — no `<Choice>` back (D-038/D-042) — is recorded as failed; absence of success is never success.
6. **Bounded retries, precise escalation.** Two failures then `needs-human`, and the escalation record contains a resume command that actually works when pasted.
7. **Ask/answer at the hub.** A worker facing an undecidable choice runs `blizzard ask` and exits; the runner forwards the question to the hub where it lives as a durable row, and the chunk parks — its derived status `waiting_on_human`, reap clock stopped. Answering means going to the hub — `blizzard answer` — which resumes the dormant session with the answer, and the chunk's derived status returns to running.
8. **One topology, encapsulated stores.** The hub runs from day one, colocated on the runner machine; the runner reaches it only through the outbound-only runner↔hub protocol (D-022), and the `blizzard` CLI reaches local facts only through the runner's local API (D-023). No code path opens a database it does not own, and no code path special-cases "no hub."
9. **The workflow engine drives the lifecycle (D-025).** The default graph is loaded from YAML at the hub, and a chunk demonstrably follows it — including the cycle: a review judged *fail* routes the chunk back to build carrying the review's findings artifact, and the judgement is the worker verdict plus the node's deterministic checks. The runner advances consecutive nodes without re-queuing (D-027), and the hub's record shows every transition.
10. **Artifacts are durable at the hub (D-026).** A finished node-step submits its branch pointers (pushed to the forge first — one per touched repo) and assets; they are visible at the hub and are fed into subsequent nodes' envelopes. A chunk whose purpose is non-code work (a review, a spike) completes with only asset artifacts.
11. **The web app observes and shapes the queue (D-048).** The hub serves its web app: every chunk's derived status, node history, artifacts, and open questions render live (SSE), and the operator can reorder the ready queue and group unleased chunks into one — the next acquire honors both.

## Scope by epic

Cited against the [epic registry](./epics.md); a slice in parentheses means the epic lands partially. Per D-016, every in-scope epic is built against its seam interface: one binding each (winter, GitHub, Claude Code), but no component may reference a concrete binding directly.

- **In scope:** `epic:store` · `epic:supervisor` · `epic:workflow` (the engine, artifacts, and the default graph — D-025/D-026) · `epic:adapters` (Claude Code only) · `epic:review` (the default graph's review node: agent review + e2e) · `epic:delivery` (the hub-executed deliver node: merge-to-main baseline — D-030) · `epic:ask-answer` (hub slice: answer at the hub) · `epic:hub` (separation slice: the colocated orchestrator daemon, the PM wrapper, and the runner↔hub protocol — D-022/D-024) · `epic:board` (local slice: the hub-served web app — observability, prioritization, grouping — D-048)
- **Out of scope:** `epic:gates` · `epic:migration` · `epic:batching` · `epic:hub` (remote slice) · `epic:board` (remote slice) · `epic:chat` · `epic:ci-feedback` · `epic:team` · `epic:config` (MVP constants may be hard-coded; the tuning surface comes later — the workflow YAML is the one config surface that ships in the MVP)

## The cut list

Everything below is deliberately absent from the MVP. Each cut has a home in the [roadmap](./roadmap.md); none of them may sneak in early without amending this document.

| Cut | Why it can wait | Arrives |
|-----|-----------------|---------|
| Gates — `epic:gates` | Gates are workflow nodes plus a runner-config dial now (D-025/D-032), so no separate mechanism waits — but gate nodes themselves (park for sign-off, surface to the human) and the runner-config shape are absent from the MVP, and the surfacing shape is an [open question](../decisions/open-questions.md). The parking machinery is shared with ask/answer, so the cost when it comes is small. | later |
| Graph migration & lifecycle metadata — `epic:migration` | Immutability and chunk pinning are structural and effectively free (D-033/D-040), so they ship with the engine — but the *movement* machinery waits: migration requests and records (D-034/D-037), auto-drift via `migration_target` (D-040), enable/disable (D-039), and `PATCH /graphs/{id}`. The MVP runs the default graph; editing a graph mid-flight (including D-034's overnight-hotfix scenario) is handled the MVP way — escalate and let the human intervene. | later |
| Batching — `epic:batching` | Execution stays solo: one chunk, one environment, one agent. A chunk wraps one PM item unless the operator grouped chunks by hand in the web app (D-048); *automated* bundling — the heuristic packer and the LLM planner — is an optimization added later, it depends on the solo path existing first, and its interplay with the workflow graph is an [open question](../decisions/open-questions.md). | later |
| Hub-side routing (pin work to a runner) | The seam exists by construction — work is acquired from the hub (D-024) — but no routing flags ship in the MVP; every runner pulls from the common queue. | later |
| PM write-back | Nothing is reflected to the PM item in the MVP (D-047): no labels, comments, status, or close on delivery — after ingest the item is untouched, and the hub's record is the fleet's truth. A future idea, deliberately unexplored. | later |
| Judge-agent nodes | MVP judgement is worker verdict + deterministic checks (D-025); a fresh evaluator agent as a node's judge is a later node kind. | later |
| Remote hub deployment — `epic:hub` (remote slice) | The hub itself ships in the MVP, colocated (D-022) — what waits is running it off-machine and registering multiple runners. Nothing architectural changes when it moves. | milestone:centralized-hub |
| Remote board, phone control — `epic:board` (remote slice) | The web app itself ships in the MVP, hub-served (D-048); what waits is reach — the PWA shell, viewer/operator roles, and auth — which is what `persona:product-manager` and `persona:senior-se` need, and they are not MVP personas. | milestone:centralized-hub |
| Remote ask/answer — `epic:ask-answer` (remote slice) | The protocol — ask-and-exit, `waiting_on_human`, resume-with-answer, the question row at the hub — ships in the MVP (criterion 7). What waits is *reach*: fanning asks out to the board and chat bot for people who aren't at the machine. That is what `persona:product-owner` needs — not an MVP persona. | milestone:centralized-hub |
| Chat bot + operator auth — `epic:chat` | No remote surface exists yet to authenticate to. | milestone:centralized-hub |
| OpenCode / Codex adapters — `epic:adapters` (remaining harnesses) | One harness proves the adapter seam; the interface is designed for three from day one. | later |
| Automated CI feedback — `epic:ci-feedback` | CI red → escalate with resume command; the human is the MVP feedback loop. | later |
| Team mode — `epic:team` | Multi-operator policy and shared visibility ride the remote hub; the hub's queue already arbitrates across runners (D-024), and a chunk's PM referent is already the GitHub issue — a policy layer later, not a schema change now. | later |
