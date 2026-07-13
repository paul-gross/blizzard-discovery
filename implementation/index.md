# Implementation plan

How blizzard gets **built**: the development ecosystem, the architecture rules the code is held to, the testing and mocking strategy, and the tech stack.
The `design/` layer says what the system *is*; this layer says how the repos, tooling, and code that realize it are organized.
Like everything in this corpus, these are proposals — items that settle graduate into the [decision log](../decisions/log.md).

| Document | What it answers |
|----------|-----------------|
| [ecosystem.md](./ecosystem.md) | The development ecosystem: the blizzard-workspace fork of winter, the standalone and project repo inventory, and the canon-derived blizzard-harness. |
| [architecture.md](./architecture.md) | The architecture rules the code is held to: the CLEAN aspects carried over from winter-cli and the read/write repository split. |
| [testing.md](./testing.md) | The four test tiers — unit, component, service, e2e — and what each is for. |
| [crash-testing.md](./crash-testing.md) | The crash-correctness harness behind MVP criterion 4: the steppable loop, injected clock, crash-point registry, invariant checker, and the kill-9 sweep. |
| [mocking.md](./mocking.md) | The mock fleet: mock coding harnesses, the mock hub and mock runner, the mock GitHub forge, and the mock-data CLI agents use to set up test cases. |
| [tech-stack.md](./tech-stack.md) | The language, framework, and storage choices — plus the researched options and recommendation for hosting Angular and FastAPI as one application. |
| [verification.md](./verification.md) | How building agents verify end-to-end locally: the winter-orchestrated per-env stack, the fixture workspace the runner acts on, and the acceptance loop. |
| [build.md](./build.md) | Build, CI, and release: the GitHub Actions gates, the trunk-and-tags release model, store migrations, and the generated API client. |
| [bootstrap.md](./bootstrap.md) | The ordered phases from end-of-discovery to a feature-building fleet, each with an exit criterion. |
