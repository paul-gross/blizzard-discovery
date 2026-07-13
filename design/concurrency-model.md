# Concurrency model

Running many agents at once is a concurrency problem, but it is not a single concurrency problem. Conflating the three distinct problems below is how naive fleets end up either over-locked (agents serialized behind one global mutex) or corrupt (two agents clobbering shared state). Blizzard separates them and applies the cheapest correct mechanism to each.

## Three distinct concurrency problems

### 1. Work assignment — the only real race is at the hub

Many workers must draw from one pool of work without two of them ever holding the same chunk — but that race is resolved in exactly one place, and it is not the runner store. The hub owns the queue of ready chunks and grants each to **exactly one runner** (D-024); that **acquisition** is the sole genuine contention in the system, because separate runner machines are the only rivals that can ever compete for the same chunk. Fleet exactly-once is upheld there.

A runner then **leases** each acquired chunk's node-step for a worker slot it spawns — workers never lease for themselves. This local lease is *not* a contended CAS: a runner is a single-writer daemon (D-023) with no rival on its own machine, so moving a node-step from *unleased* to *leased* is crash-safe, idempotent bookkeeping, not a race anyone can lose. A lease's scope is **one node-step execution attempt** (D-035): one lease, one worker tenure, one fresh epoch — consecutive node-steps of a runner-sticky chunk (D-027) run under successive local leases, and a parked chunk holds no live lease at all.

A lease is a **TTL lease with heartbeats**, not a permanent assignment. A worker that dies holding one does not hold it forever — the lease expires when its heartbeat goes stale, and the node-step is retried under a fresh lease and epoch. What stops a *reaped-but-still-alive* worker from clobbering its successor is not runner-local mutual exclusion — there is no rival on the machine to exclude — but the **fencing epoch checked at the hub** (below): the successor leased with a newer epoch, and the zombie's stale submission is rejected.

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

The lease mechanics are uniform across chunk leases and singleton resources:

- **TTL lease** — a lease or an acquisition is valid only for a bounded time.
- **Heartbeat** — the holder renews the lease periodically by proving it is still alive and making progress. A stale heartbeat is the signal that the holder is gone.
- **Fencing tokens (epochs)** — each lease carries a monotonically incrementing **epoch**, minted per node-step (D-035). Transition submission — which carries the step's artifacts in the same atomic write (D-036) — and the hub's deliver node check the epoch before they act. A holder that was reaped as dead but is in fact still running (a zombie) carries an old epoch, and its submission is rejected — so a reaped-but-still-running worker can never clobber the work of the successor that took over its chunk. This is the classic fencing-token guard against the split-brain that heartbeats alone cannot prevent.

## The delivery lane: the hub's merge queue (D-030)

Delivery — merging a chunk's branch artifacts to main, or opening its pull request — executes at the hub, which processes deliveries through an internal **merge queue**, one chunk at a time. The lane is single-writer *by construction*, across every runner in the fleet — which is exactly why it cannot be a runner-local lease: at `milestone:centralized-hub`, several machines deliver into the same repos, and only the hub sees them all. Combined with epoch-fenced submission, this guarantees that finished work lands linearly and that no zombie can overwrite a live successor's result. The queue's mechanics are settled: strict FIFO, one chunk at a time across all its repos with no train batching (D-057); conflicts route to the conventional `merge` graph (D-058); PR-mode merging is capability-gated, with externally-merged PRs detected by hub-side polling plus an on-demand check route (D-059/D-065). The full treatment is in [workflow-engine.md](./workflow-engine.md).
