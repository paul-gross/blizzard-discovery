# Roadmap

The rollout is a sequence of **named milestones**, each carrying a stable `milestone:<slug>` id that scope statements cite (like `epic:<slug>` and `persona:<slug>`). A milestone is independently useful and shippable, and always spans more than one epic. The always-correct 1:1 fallback exists from the first milestone, so every later layer is an optimization on top of something that already works.

## milestone:mvp — the local fleet

Epics (see the [registry](./epics.md)): `epic:store`, `epic:supervisor`, `epic:workflow`, `epic:adapters` (Claude Code), `epic:review`, `epic:delivery`, `epic:ask-answer` (hub slice), `epic:hub` (separation slice), `epic:board` (local slice).

This is the MVP; [mvp.md](./mvp.md) states its acceptance journey, testable criteria, and cut list.

- `blizzard-hub` from day one, colocated on the runner machine (D-022): the work orchestrator — the canonical responsibility list is in [design/hub/index.md](../design/hub/index.md). HTTP + SSE; the runner connects outbound-only through the same protocol a remote hub uses.
- The workflow engine with the default graph (build → review → deliver, review-fail cycling back to build); judgement = worker verdict + deterministic checks; deliver executes at the hub through its merge queue (D-030).
- Artifacts: branch/commit pointers (pushed to the forge) and assets, submitted per node-step and stored at the hub.
- Supervisor reconciliation loop: `blizzard-runner`, a daemon exposing a local API, with its store — sqlite (WAL) — embedded inside it (D-023); it advances held chunks node to node without re-queuing (D-027).
- The `blizzard` CLI as a pure client: local verbs → runner local API, fleet verbs → hub HTTP API.
- Solo execution (one env, one agent per chunk); a chunk wraps one PM item unless grouped by hand in the web app (D-048).
- The hub-served **web app** (D-048): fleet observability plus queue shaping — reorder the ready queue, group unacquired chunks.
- Claude Code adapter + hooks.
- Ask/answer at the hub: `blizzard ask` parks the chunk (`waiting_on_human`); `blizzard answer` resumes the dormant session with the answer.
- Escalation via printed resume commands.

## milestone:centralized-hub — the control plane beyond the local environment

Epics: `epic:hub` (remote slice), `epic:board` (remote slice), `epic:ask-answer` (remote slice), `epic:chat`.

Enable the control plane beyond the local environment, and the spokes out of the hub: the hub runs centrally, and the spokes fan out from it — remote runners, the board, the chat bot, remote ask/answer. The execution core built in `milestone:mvp` goes untouched. (Multi-*operator* semantics are not this milestone — that is `epic:team`, unscheduled.)

- **Remote hub** — `epic:hub` (remote slice): run `blizzard-hub` off-machine (VPS / cloud / home server); register multiple runners with one hub. No architectural change — the protocol ships in the MVP (D-022). Eventual-reachability hardening: buffer events locally through hub outages and flush on return.
- **Remote board** — `epic:board` (remote slice): the MVP's web app goes remote — phone-friendly PWA, viewer and operator roles (operator control scope is an [open question](../decisions/open-questions.md)).
- **Remote ask/answer** — `epic:ask-answer` (remote slice): asks fan out to the board and notifications; answers flow back from remote clients. (The protocol itself — ask-and-exit, `waiting_on_human`, resume-with-answer, the question row at the hub — ships in the MVP.)
- **Chat + auth** — `epic:chat`: Telegram bot (D-031) with inline answer buttons; identity & permissioning per the post-MVP spike (D-018) — configurable, authless behind Tailscale locally, OAuth2-class identity for cloud-hosted hubs.

## Later

Unscheduled — in any order, and none may change the mvp core's contracts:

- **Batching** (`epic:batching`) — the rollout ladder (D-011): a deterministic heuristic packer (bundle same-repo, `size:small`-labeled items, capped at about 5 per chunk) first, the LLM planner in shadow mode after it, promoted to driving only when it demonstrably wins. Shape pending the batching × workflow-graph [open question](../decisions/open-questions.md).
- **Gates** (`epic:gates`) — human-gate nodes plus the runner-config dial: opt-in sign-offs as gate nodes in the graph, or imposed by a runner's configuration on any node by node name (D-025 made gates graph structure; D-032 added the runner dial, D-041 keyed it on names; this ships both).
- **Graph lifecycle** (`epic:migration`) — moving in-flight chunks between immutable graphs: explicit migration requests (deferred + forced, D-034/D-037), auto-drift via `migration_target` (D-040), enable/disable (D-039), `PATCH /graphs/{id}`. The MVP mints and runs graphs but never moves a chunk between them.
- **Hub-side routing** — pin or filter which runner a chunk goes to, by flag at the hub (the seam exists from the MVP, D-024).
- **Judge-agent nodes** — a fresh evaluator agent as a node's judgement, beyond verdict + checks.
- **Configuration surface** (`epic:config`) — every operational constant in config, with comparable runs for tuning (the workflow YAML ships in the MVP; this is the rest).
- **OpenCode and Codex adapters** (`epic:adapters`).
- **Automated CI feedback** (`epic:ci-feedback`) — capped at 2 rounds.
- **PM write-back** — reflecting chunk state and completion to the backing PM item (labels, comments, close on delivery); the MVP writes nothing back (D-047), and the idea is deliberately unexplored so far.
- **Team mode** (`epic:team`). Multiple operators sharing one fleet: multi-operator policy and shared conventions on top of `milestone:centralized-hub`'s remote hub. Work acquisition stays hub-mediated (D-024) — the hub's chunk model wraps the PM item and the hub's queue arbitrates across machines; a colleague's issue assignment is at most a *reflection* of hub state, never the acquisition itself. (An earlier draft made forge-native issue assignment the atomic cross-machine acquisition; D-024 superseded that — the hub is the arbiter.)
