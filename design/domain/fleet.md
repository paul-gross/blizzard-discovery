# Fleet models

The runner registry, chunk locators, and the event stream. Part of the [domain model](./index.md).

## Route

The locator fact (D-021): what makes every chunk findable, and reassignment thinkable (D-027). Born at acquire — `POST /routes` ([api.md](../hub/api.md)) *is* how a runner takes work: the hub selects ready chunks and creates the route rows (D-024).

| Property | Notes |
|----------|-------|
| `chunk_id` | The chunk being worked. |
| `runner_id` | The runner working it. |
| `workspace_id` | The workspace that runner is bound to (D-019). |
| `environment_id` | Opaque (D-021) — the hub knows *which* env, never *what* an env is. |

## Runner (registry entry)

| Property | Notes |
|----------|-------|
| `runner_id` | The runner's identity in the fleet registry, recorded at registration. |
| `workspace_id` | The per-runner workspace binding (D-019). |
| `last_seen_at` | Most recent contact with the hub — see below. |
| `paused` | **Derived** from appended pause/resume facts (D-004, the D-039 pattern) — set via `PATCH /runners/{id}` (D-043); see below. |
| liveness | **Derived** from `last_seen_at` (D-004) — see below. |

A runner's connection is outbound-only (D-012), and `workspace_id` names which prepared workspace it spawns into.

**`last_seen_at`** is refreshed by any inbound call (acquire, event push, poll) and by the runner-level heartbeat whose cadence is an [open question](../../decisions/open-questions.md). It is the fact liveness derives from.

**liveness** is never a stored column (D-004): `last_seen_at` against a staleness threshold yields online/offline for the board. It is distinct from the worker tool-call heartbeat, which is a runner-store fact and never leaves the machine.

**`paused`** is the operator's brake (D-043): pause/resume facts append and the flag derives, exactly as a graph's enabled-ness does (D-039). The runner reads its own state on its outbound pull and adheres to it — pausing stops new leases, in-flight chunks run to completion ([loop.md](../runner/loop.md)) — never a push into the dev box (D-012). Post-MVP controls (routing pins) take the same declarative shape; there is no directive queue.

## Event

What the SSE stream carries and the batched push ingests — `question.answered` is the one named so far; the canonical fact/event vocabulary that statuses derive from is an [open question](../../decisions/open-questions.md).
