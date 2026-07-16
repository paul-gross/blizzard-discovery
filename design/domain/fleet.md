# Fleet models

The runner registry, chunk locators, and the event stream. Part of the [domain model](./index.md).

## Route

The locator fact (D-021): what makes every chunk findable, and reassignment thinkable (D-027). Born complete at the claim — `POST /routes` ([api.md](../hub/api.md)) *is* how a runner takes work (D-080): the runner peeks the hub-ordered queue, acquires the environments, and posts the full route; the hub accepts exactly one claim per chunk (D-024). Released by detach (`route.released`, D-088) or the chunk's terminal outcome.

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
| `hub_paused` | **Derived** from appended pause/resume facts (D-004, the D-039 pattern) — set via `PATCH /runners/{id}` (D-043); see below. |
| `locally_paused` | **Derived** from appended facts the runner *reports* (D-105) — set on the runner, never here; see below. |
| liveness | **Derived** from `last_seen_at` (D-004) — see below. |

A runner's connection is outbound-only (D-012), and `workspace_id` names which prepared workspace it spawns into.

**`last_seen_at`** is refreshed by any inbound call (acquire, event push, poll) and by the dedicated runner-level heartbeat, `POST /runners/{id}/heartbeats` (D-070) — the SSE subscription deliberately never counts, because dead connections linger silently. Cadence and staleness threshold live with the other [constants](../../decisions/open-questions.md). It is the fact liveness derives from.

**liveness** is never a stored column (D-004): `last_seen_at` against a staleness threshold yields online/offline for the board. It is distinct from the worker tool-call heartbeat, which is a runner-store fact and never leaves the machine.

**A runner has two brakes** (D-105), because two parties can stop it for different reasons. Both append facts and derive their flag (D-004, the D-039 pattern); neither is a stored status. Either one stops new leases — in-flight chunks always run to completion ([loop.md](../runner/loop.md)) — so **effective paused is their OR**, and each is cleared only on the surface that set it.

**`hub_paused`** is the *fleet's* brake (D-043): the operator stops a runner from here, via `PATCH /runners/{id}`. The runner reads it on its outbound pull and adheres to it — never a push into the dev box (D-012) — and `blizzard hub resume` clears it. Advisory today: the hub does not yet refuse a paused runner's claim, so the brake leaks for as long as the runner's next pull takes.

**`locally_paused`** is the *runner's own* brake — it declines to claim ("I won't try"). It is set on the runner machine (`PATCH /runner`, [runner/api.md](../runner/api.md)), which is why it works with the hub unreachable, and it reaches the registry only because the runner **reports** it — `runner.locally_paused` / `runner.locally_resumed` ride its outbound buffer ([events.md](./events.md)). The hub only ever reads this one: it cannot set it, and `blizzard hub resume` cannot clear it. The report exists so the board can say *which* brake is on, since the hub can only render what it holds.

Post-MVP controls (routing pins) take the same declarative shape; there is no directive queue.

## Event

What the SSE stream carries and the batched push ingests. Fact names double as event names — the canonical vocabulary is [events.md](./events.md) (D-067); `question.answered` is one of them.
