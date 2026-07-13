# Crash-correctness harness (kill-9 testing)

MVP acceptance criterion 4 promises that `kill -9` at every step boundary is **a tested operation, not a hope** ([product/mvp.md](../product/mvp.md)).
This document is the harness that makes that promise executable — and the four architectural requirements it imposes on the daemons, which is why it is settled before the build rather than retrofitted after.
It is not a fifth test tier ([testing.md](./testing.md)): crash correctness is an orthogonal dimension, exercised with the same mock fleet ([mocking.md](./mocking.md)) and scenario fixtures ([verification.md](./verification.md)) the tiers already use.

## The four requirements on daemon code

These are constraints the implementer builds in from the first commit; each is cheap on day one and expensive to retrofit.

1. **A steppable loop.** REAP, PULL, FILL, and ADVANCE are individually callable step functions — each a function of (store, clock, seam clients) — and the daemon's tick timer is merely one driver of them. Tests drive the same functions directly, one step at a time, stopping exactly at a boundary. The hub's coordinator loop follows the same rule.
2. **An injected clock.** All time flows through a clock abstraction wired at the composition root ([architecture.md](./architecture.md)) — no direct `time.time()` / `datetime.now()` in loop, store, or domain code, including SQLAlchemy column defaults. Tests advance time virtually, so lease TTLs, reap staleness thresholds, and "overnight" pass in milliseconds.
3. **A crash-point registry.** The dangerous windows get stable names in a code-owned registry — e.g. `fill.after-env-acquire.before-claim`, `fill.after-claim.before-lease`, `advance.after-buffer.before-flush`, `deliver.after-repo-land`, `reap.after-kill.before-expire` — and under test scaffolding a daemon run as a subprocess SIGKILLs itself on reaching the armed point (selected via environment variable; the same fencing convention as the mock harness — it can never fire outside a test-marked environment). The registry is enumerable, which is what the sweep iterates, and it doubles as the authoritative list of windows the design claims are safe.
4. **A facts-level invariant checker.** A set of assertions evaluated over both stores' facts after any crash → restart → recover cycle: no duplicate env binding, at most one accepted transition per node-step epoch (D-007/D-036), no double delivery and per-repo lands idempotent (D-057), every derived status computable with exactly one match (D-004), outbound-buffer sequence gapless (D-069). Because both stores are facts-only SQL (D-004/D-023), the checker is essentially a library of SQL assertions plus the derivation queries themselves; a failure names the violated invariant.

## The sweep

The top-level proof of criterion 4, run as its own pytest-driven runner (no Playwright — this is daemon-level, not browser-level):

1. Take a scenario from the shared fixture corpus — the acceptance journey and its edge variants, scripted through the mock fleet's prompt-is-the-program harnesses ([mocking.md](./mocking.md)) and seeded as one consistent world ([verification.md](./verification.md)).
2. Enumerate the crash-point registry.
3. For each point: run the scenario with the daemons as **real subprocesses** (real sqlite WAL files, real SIGKILL — durability claims are only proven by killing a real process), arm the point, let the daemon kill itself there, restart it, drive the loop to quiescence with the virtual clock.
4. Assert the invariant checker plus the scenario's own outcome (the chunk still lands, or still parks, exactly as the un-crashed run does).

The mock hub's and mock runner's levers compose with the sweep: killing the runner *while* the mock hub delays or replays is how the store-and-forward and idempotency claims (D-069) are exercised under crash, not just under latency.

## Division of labor with the tiers

- **Unit tier**: each step function's idempotency in isolation — run it twice against the same store state, assert one effect.
- **Component tier**: steps driven in-process against the virtual clock — cheap boundary-order and recovery-logic coverage without subprocesses.
- **The sweep**: real subprocesses, real signals, full scenarios — the only tier-independent piece, and the one that makes criterion 4 literal.
