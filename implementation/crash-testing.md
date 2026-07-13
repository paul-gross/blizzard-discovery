# Crash-correctness harness (kill-9 testing)

> **Graduated.** The crash-correctness rules now live in **blizzard-harness**, their canonical owner.
> This file is a pointer; edit the rules there.

MVP acceptance criterion 4 promises that `kill -9` at every step boundary is **a tested operation, not a hope** ([product/mvp.md](../product/mvp.md)).

- **The four architectural requirements** the daemons are built to honor — a steppable loop, an injected clock, a crash-point registry, and a facts-level invariant checker (D-007/D-036/D-057/D-069, D-004/D-023) — are owned by `blizzard-harness:/architecture/crash-correctness.md`.
- **The kill-9 sweep** that iterates the registry against real subprocesses, and its division of labor with the unit and component tiers, is the `blizzard:crash-sweep` method in the harness verifiability matrix at `blizzard-harness:/verification/blizzard.md`.

Crash correctness is an orthogonal dimension, not a fifth tier ([testing.md](./testing.md)); it is exercised with the same mock fleet ([mocking.md](./mocking.md)) and scenario fixtures ([verification.md](./verification.md)) the tiers already use.
