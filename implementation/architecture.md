# Architecture rules

> **Graduated.** These rules now live in **blizzard-harness**, their canonical owner:
> `blizzard-harness:/architecture/index.md`.
> This file is a pointer; edit the rules there.

The code-level rules blizzard is held to — the CLEAN aspects carried over from winter-cli, the repository read/write split, and the domain-takes-objects rule — are owned by the harness:

- **CLEAN aspects** (screaming layout, dependency-free domain core, dependency inversion, injection at the composition root) — `blizzard-harness:/architecture/clean-architecture.md`.
- **Repository access** (read-only vs write repositories, layer-gating so only the domain writes, domain-takes-objects) — `blizzard-harness:/architecture/repository-access.md`.
- **System shape** (deterministic shell / intelligent core, pluggable seams, store-facts-derive-status) — `blizzard-harness:/architecture/system-shape.md`.

The composition root referenced elsewhere in this plan (e.g. the injected clock in [crash-testing.md](./crash-testing.md)) is the `bzh:dependency-injection` rule in the harness.
The reference repository shape is `blizzard-harness:/exemplars/python/repo_pattern.py`.
