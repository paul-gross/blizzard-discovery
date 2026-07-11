# Concurrency model

Running many agents at once is a concurrency problem, but it is not a single concurrency problem. Conflating the three distinct problems below is how naive fleets end up either over-locked (agents serialized behind one global mutex) or corrupt (two agents clobbering shared state). Blizzard separates them and applies the cheapest correct mechanism to each.

## Three distinct concurrency problems

### 1. Work assignment — atomic claims in the runner store

Many workers must draw from one pool of work without two of them ever holding the same chunk. The hub owns the queue of ready chunks (D-024); a runner acquires chunks from it and **claims** one for each worker it spawns — workers never claim for themselves. A claim is an atomic compare-and-set in the runner store: the chunk transitions from *unclaimed* to *claimed* in a single atomic step, and exactly one claimant wins any race. A claim's scope is **one node-step execution attempt** (D-035): one claim, one worker tenure, one fresh epoch — consecutive node-steps of a runner-sticky chunk (D-027) run under successive local claims, and a parked chunk holds no live claim at all.

Claims are **TTL leases with heartbeats**, not permanent assignments. A worker that dies holding a claim does not hold it forever — the lease expires when its heartbeat goes stale, and the chunk returns to the pool.

### 2. Workspace isolation — already solved below the workspace seam

Two workers on two different chunks must never share a working tree. This problem is fully solved by the workspace provider — winter feature environments in the reference binding: each chunk runs in its own environment(s), with per-repo worktrees, its own ports, and its own services. There is nothing to lock here because there is nothing shared. This is the problem the rest of the ecosystem does *not* solve at the polyrepo granularity, and it is why winter is the reference workspace.

### 3. True singletons — shared resources and the delivery lane

Some things genuinely are single-writer and cannot be sharded away:

- **Shared external resources** — a resource that all workers must funnel through. These get **leases**, enforced at the resource boundary (D-008).
- **The delivery lane** — landing finished work on the shared upstream is single-writer by nature. This one is not leased at the runner: it is **serialized at the hub** (D-030) — see the merge queue below.

## Reduce contention structurally before locking

The lease list should stay short. Before reaching for a lock, remove the contention:

- **Shard work by repo or subsystem** so that chunks that would collide are simply never assigned concurrently.
- **Make services per-environment** so that what looks like a shared service is actually N independent per-env services with no contention between them.

A lease is a last resort for the handful of resources that are irreducibly single-writer. Every resource made per-env is a lease never contended.

## Leases, TTL, heartbeat, fencing

The lease mechanics are uniform across chunk claims and singleton resources:

- **TTL lease** — a claim or an acquisition is valid only for a bounded time.
- **Heartbeat** — the holder renews the lease periodically by proving it is still alive and making progress. A stale heartbeat is the signal that the holder is gone.
- **Fencing tokens (epochs)** — each claim carries a monotonically incrementing **epoch**, minted per node-step (D-035). Transition submission — which carries the step's artifacts in the same atomic write (D-036) — and the hub's deliver node check the epoch before they act. A holder that was reaped as dead but is in fact still running (a zombie) carries an old epoch, and its submission is rejected — so a reaped-but-still-running worker can never clobber the work of the successor that took over its chunk. This is the classic fencing-token guard against the split-brain that heartbeats alone cannot prevent.

## The delivery lane: the hub's merge queue (D-030)

Delivery — merging a chunk's branch artifacts to main, or opening its pull request — executes at the hub, which processes deliveries through an internal **merge queue**, one chunk at a time. The lane is single-writer *by construction*, across every runner in the fleet — which is exactly why it cannot be a runner-local lease: at `milestone:centralized-hub`, several machines deliver into the same repos, and only the hub sees them all. Combined with epoch-fenced submission, this guarantees that finished work lands linearly and that no zombie can overwrite a live successor's result. The queue's mechanics — ordering, conflict handling, PR mode — are an [open question](../decisions/open-questions.md).
