# Test strategy

> **Graduated.** The test strategy now lives in **blizzard-harness**, its canonical owner:
> `blizzard-harness:/verification/blizzard.md` (the verifiability matrix).
> This file is a pointer; edit the tiers and tier rules there.

The four test tiers (unit, component, service, e2e), the tier rules (seams-mocked service/e2e, mock-counterpart one-sided tests, mock-data setup, sqlite-only test runs), and their stable method ids are owned by the harness verifiability matrix at `blizzard-harness:/verification/blizzard.md`.

Crash correctness remains an orthogonal dimension, not a fifth tier — see [crash-testing.md](./crash-testing.md).
The mocks the upper tiers bind are owned by [mocking.md](./mocking.md).
