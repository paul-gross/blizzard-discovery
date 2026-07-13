# Test strategy

Four tiers, all of them used — each answers a different question, and none substitutes for another.
The mocks the upper tiers depend on are owned by [mocking.md](./mocking.md).

| Tier | Scope | Tooling |
|------|-------|---------|
| **Unit** | One class or function in isolation. | pytest |
| **Component** | A larger area of the app — a whole domain slice or subsystem, not one class — wired with real internal collaborators and test doubles only at the seams. | pytest |
| **Service** | The actual HTTP APIs of a running hub or runner, exercised from outside, with the seams bound to supported mocks (mock harness, mock counterpart daemon, mock-data-seeded store). | Playwright |
| **E2E** | The full system — hub, runner, web app — driven through the browser and CLI, running **fully locally with every seam bound to the mock fleet**: an agent e2e tests with mocks and verifies it works locally. | Playwright (a separate Playwright project from the service tier) |

## Tier rules

- **Service tests never spend real tokens and never touch the network.** The harness seam is bound to a mock coding harness from `blizzard-mock`; the work-source and delivery seams to the mock GitHub forge; the workspace seam to mocks or local fixtures.
- **One-sided service tests use the mock counterpart.** Runner service tests run against the mock hub; hub service tests run against the mock runner — edge cases are constructed by driving the mock's levers, not by contriving the real daemon into rare states.
- **Test data is set up through the mock-data CLI and fixtures**, not ad-hoc SQL — see [mocking.md](./mocking.md).
- **Tests run against sqlite.** Postgres support is a configuration concern held by staying inside SQLAlchemy's portable surface ([tech-stack.md](./tech-stack.md)) — not a second test matrix.
- **Crash correctness is an orthogonal dimension, not a fifth tier.** The kill-9 sweep and the four architectural requirements it imposes on the daemons (steppable loop, injected clock, crash-point registry, invariant checker) are [crash-testing.md](./crash-testing.md).
