# Verification during implementation

The agents building blizzard verify it by **operating it**: winter stands up the full stack in an isolated feature environment, the agent generates work that needs completing, and watches it travel the entire lifecycle — ingest → acquire → work → deliver → landed commit.
This is the e2e-with-mocks bar ([testing.md](./testing.md)) turned into a hands-on dev loop: not a test suite artifact, but the thing an agent does after any change to convince itself the system still works.

## The per-env stack — winter services

Everything the loop needs runs as winter-orchestrated services in the feature env, port-band isolated so parallel envs verify in parallel:

| Service | Notes |
|---------|-------|
| postgres | Backing DB when a scenario wants it; sqlite scenarios skip it. |
| mock GitHub forge | The PM + delivery seams ([mocking.md](./mocking.md)). |
| `blizzard hub` | The code under test. |
| `blizzard runner` | The code under test; spawns mock coding harnesses (not a service — the runner spawns them). |

## The target-workspace problem

The stack above is the easy half.
The hard half: the runner must act on a **real winter workspace somewhere in the filesystem** — acquiring feature environments, resetting worktrees, making commits.
Mocking the workspace seam would gut the verification: winter is the reference binding, and acquire/release/reset-on-acquire is precisely the behavior that must be *seen* working.

The answer is a **fixture workspace** — generated, disposable, and real:

- A scaffold tool in `blizzard-mock` mints a directory containing **bare git origin repos** (addressed as `file://` remotes — no network, no real forge in the git path) and a real winter workspace initialized against them, with a small committed history and a `.winter/config.toml` declaring them as project repos.
- It lives under a **per-env scratch path** (keyed off the feature env, e.g. `WINTER_ENV`), so two feature envs verifying at once never share a fixture.
- It is a real workspace root (its own `.winter/config.toml`): the runner under test drives the real winter CLI against *it*, exercising the actual workspace binding — while remaining a different workspace from the one the building agent itself works in.

## Real commits from a mock agent

The runner spawns a mock coding harness into an env of the fixture workspace, and yes — commits still have to happen, so the mock makes them.
"Do work" is just another scripted behavior ([mocking.md](./mocking.md)): apply a prescribed diff, `git commit`, exit with the scripted verdict.
The mock never thinks, but everything downstream of it is fully real: the push to the `file://` origin, the PR minted at the mock forge, the merge queue, the land.

## One git truth: the forge fronts the fixture origins

The mock forge is configured to front the **same bare repos** the fixture workspace pushes to — its backing model is owned by [mocking.md](./mocking.md).
That single-git-truth wiring is what makes verification honest: mergeability is computed against real refs, "merge this PR" performs a real merge into bare `main`, and the external-merge lever is a direct push a running delivery flow must then detect (D-065).

## Scenario consistency

A scenario spans three state holders, and they must agree:

| State holder | Seeded by |
|--------------|-----------|
| Hub/runner store rows (chunks, leases, facts) | mock-data CLI ([mocking.md](./mocking.md)) |
| Forge state (issues, comments, PRs) | mock forge fixtures |
| Git state (bare origins, workspace, envs) | fixture-workspace scaffold |

A chunk citing PM item `#12` needs issue `#12` living in the mock forge; a delivered chunk needs its merged commit present in the bare origin.
Named scenario fixtures therefore mint all three together — the mock-data CLI's fixture concept extends from "rows" to "a consistent world."

## The acceptance loop

The loop an agent runs to verify a change end-to-end, entirely locally, zero tokens:

1. `winter service up <env> --wait` — postgres, mock forge, hub, runner.
2. Seed a scenario — fixture workspace + forge issues + store rows, one named fixture.
3. Generate work: file an issue on the mock forge, `blizzard hub ingest` it.
4. Watch the chunk travel: env acquired in the fixture workspace, mock harness commits, PR minted, merge queue lands it.
5. Assert the outcome at both ends: the commit is reachable from bare `main`, and the hub's facts derive the chunk `done`.
6. Pull a lever mid-flight (conflict, external merge, harness failure) when the change under test is about an edge, and assert the recovery instead.
