# Personas

Each persona carries a stable identifier of the form `persona:<slug>`. Other documents cite personas by that identifier so a requirement can always be traced to who it serves — a piece of the design that serves no persona here is gold-plating and should be challenged.

The [MVP](./mvp.md) serves exactly one persona: `persona:application-architect`. Every other persona is served by a later roadmap layer, and that is deliberate.

## `persona:senior-se` — the senior software engineer

Can absolutely code, but is not at the level of orchestrating an entire fleet. Doesn't reliably know in advance what the finished code should look like, or the best way to architect the solution. May struggle to contribute to planning, may not yet be "agentic" in daily practice, and — critically — may not trust LLM output, because they haven't made the leap from **HITL** (human-in-the-loop: reviewing every step) to **HOTL** (human-on-the-loop: supervising outcomes).

**What they need from blizzard:** the fleet's structure substitutes for orchestration skill they don't have yet. Work arrives shaped (the planner and the build-skill routing doctrine decide *how* work is executed, so they don't have to); output arrives in trustable units (per-task verdicts, small blast radius); and escalations don't assume deep harness fluency — a `needs-human` hand-off is a pasteable command and full session context, not an archaeology project. Blizzard is their **on-ramp from HITL to HOTL**, and the gates are the dial (D-032): they start with gates at every station — gate nodes in the [workflow graph](../design/workflow-engine.md), or their own runner's node-type gates — and remove them as confidence grows. Confidence is built by the system being structurally unable to lie about status or slip work past a configured gate, not by asking for trust up front.

**Served by:** fail-safe verdicts ([supervisor](../design/runner/loop.md)), the configurable human gates ([mission](./mission.md)), the [planner](../design/planner.md)'s routing doctrine, the board ([hub](../design/hub/index.md)).

## `persona:application-architect` — the application architect (primary)

The agentic-native engineer this project is being built by and for. Has the technical chops to describe systems in writing and work with LLMs to produce larger technical plans and feature specs. Is far enough into agentic development to more or less **trust the output** — not blindly, but because they trust that when a run goes astray they can **harness it**: drop into the exact session, see what the agent saw, and fix or redirect it.

**What they need from blizzard:** unattended throughput (file issues, walk away, return to landed work); truthful status at a glance; cheap takeover (one pasted resume command lands them in the stuck agent's full context); zero duplicated or clobbered work; polyrepo end-to-end. In short: the [mission](./mission.md)'s success criteria are this persona's needs.

**Served by:** everything in the [MVP](./mvp.md) — the [runner store](../design/runner/store.md), the [supervisor](../design/runner/loop.md), the [adapters](../design/harness-adapters.md) and their takeover flows.

## `persona:product-manager` — the product manager

Owns *what gets built and in what order*. Interacts with the fleet exclusively through its two public surfaces: the GitHub backlog going in, and the board coming out. Never touches a harness or an environment.

**What they need from blizzard:** filing and labeling an issue is the entire act of scheduling work; priority expressed on the backlog actually steers what the fleet pulls next (GitHub has no per-issue priority, so the hub must implement ordering — an [open question](../decisions/open-questions.md)); the board answers "where is everything?" without asking an engineer.

**Served by:** the PM-as-backlog truth split ([architecture](../design/architecture.md)), the hub's PM wrapper (D-024, [hub](../design/hub/index.md)), and the board.

## `persona:product-owner` — the product owner

Owns *whether what was built is right*. The natural first responder for the fleet's product-intent questions and the acceptance check on its output.

**What they need from blizzard:** `blizzard ask` questions about product behavior reach them on their phone with one-tap answers, and their answer demonstrably reaches the agent ("delivered, agent resumed"); finished work is traceable to its originating issue and inspectable at whatever gates are configured — a PR when the review gate is on, the merged diff otherwise.

**Served by:** the [ask/answer protocol](../design/ask-answer.md) end to end, the operator role, and the chat bot.

## `persona:harness-engineer` — the harness engineer

The fleet's mechanic. Interested less in any single task's output than in the **ecosystem of outputs**: when the automated system goes wrong, they want to reconstruct exactly what happened; then they want to tweak parameters and observe how the distribution of outcomes shifts.

**What they need from blizzard:** a **configuration-based** system — every operational constant (lease TTL, heartbeat interval, retry cap, tick rate, batch size caps, model/harness routing) lives in config, changeable without touching code; cheap forensics — the facts-only schema means an incident's *lifecycle* is reconstructable by querying what actually happened, with no written-status lies to un-tangle (the agent's *reasoning* lives in transcripts on the runner machine — retention and access are an [open question](../decisions/open-questions.md)); and comparable runs, so a parameter change can be evaluated against real outcome data rather than vibes.

**Served by:** the facts-only schema ([architecture](../design/architecture.md)), `blizzard selftest` and pinned harness versions ([harness-adapters](../design/harness-adapters.md)), the planner's shadow-mode comparison discipline ([planner](../design/planner.md)). The concrete constants are still an [open question](../decisions/open-questions.md) — this persona is the reason they must land in config, not in code.

## `persona:fleet-agent` — the fleet agent (non-human)

The worker session itself. Blizzard's design quality is largely determined by how little cooperation this persona is *trusted* to provide.

**What they need from blizzard:** an unambiguous claim it cannot accidentally share; a way to ask a human a question without dying for it (`blizzard ask`, ask-and-exit); a way to report a verdict that will be believed; heartbeating that costs it nothing (emitted as a side effect of tool use — no self-reporting).

**Served by:** atomic claims and epochs ([concurrency-model](../design/concurrency-model.md)), the `PostToolUse` heartbeat and fail-safe verdicts ([supervisor](../design/runner/loop.md)), ask-and-exit ([ask-answer](../design/ask-answer.md)).
