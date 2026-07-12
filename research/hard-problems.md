# Hard problems

Studying Agent Orchestrator surfaced four problems that any serious fleet has to solve. This document names each one and states blizzard's answer.

## 1. Truthful status derivation

**The problem.** A process that writes its own status and then dies leaves a database that lies — it says "running" forever.

**The answer.** Facts-only schema with status **derived** from those facts, from day one. The stores — the runner store and the hub — record only durable facts (lease created, pid + start time, last heartbeat, verdict, ...); status is a query, never a column. A status derived from "last heartbeat 20 minutes ago, pid dead" tells the truth regardless of how the process ended. See [../design/architecture.md](../design/architecture.md).

## 2. CI / review feedback loop

**The problem.** When CI goes red or a reviewer requests changes after a PR is open, the result has to get back to the agent that did the work.

**The answer, staged:**

- **v1.** CI red → escalate with a resume command. A human resumes the session with the failure in hand.
- **v2.** Automated `claude -p --resume` with a CI log excerpt fed back into the owning session. A **hard cap of 2 feedback rounds** prevents fix/break oscillation (an agent that "fixes" one thing and breaks another indefinitely).
- The **delivery ↔ session mapping is a single stored fact** — one row links a chunk's delivery (its PR, where a gate opened one) to the session that must receive its feedback.

## 3. Crash-recovery correctness

**The problem.** The fleet must survive crashes, reboots, and `kill -9` at any point without corrupting state or double-delivering work.

**The answer, layered:**

- **pid + process start time** together, to survive pid reuse (a reused pid does not fool the reaper).
- **Idempotent steps** — every supervisor step re-runs harmlessly.
- **Durable env bindings** — the chunk→env binding is a runner-store fact and environment acquisition is idempotent per chunk (D-021), so recovery re-reads instead of re-creating.
- **Epoch check on the delivery path** — a zombie's stale epoch is fenced out, so it cannot clobber its successor.
- **`kill -9` testing at every step boundary** — crash recovery is a tested operation, not a hope.

## 4. Adapter drift

**The problem.** The harness CLIs change monthly. Observed in this survey: Codex grew `--output-schema`; Vibe Kanban died outright. An adapter validated last month may be broken today.

**The answer:**

- **Dumb adapters** — the less an adapter does, the less there is to break.
- **Pinned versions** — a run pins the harness versions it was validated against.
- **`blizzard selftest`** — run a trivial task through each harness before any unattended period, so a broken adapter is caught before it costs a night of work.
