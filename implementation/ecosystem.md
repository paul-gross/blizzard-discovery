# Development ecosystem

Blizzard is built **with winter** — the same multi-worktree, multi-repository, agent-first development model that builds winter itself.
The ecosystem is assembled before the first line of application code, so that every feature is developed in isolated feature environments by agents operating under an explicit harness.

## The workspace: `blizzard-workspace`

A workspace repo **forked from winter**, carrying the blizzard-specific customization commit on top of `winter/master` (the same shared-lineage model winter workspaces use).
It declares the repo inventory below in `.winter/config.toml` and provides feature environments (`alpha/`, `beta/`, …) containing a worktree of every project repo.

## Repo inventory

### Standalone repos (winter extensions)

Installed as extensions of the workspace; not worktreed per feature environment.
The winter-* extensions are used **directly** — the same repos that serve the winter workspace, not forks or copies.

| Repo | Role |
|------|------|
| `winter-workflow` | The agentic build skills and role-pure subagents that drive feature development. |
| `winter-service-tmux` | tmux service orchestration for feature environments. |
| `winter-service-docker` | docker compose service orchestration for feature environments. |
| `winter-canon` | The universal harness conventions every harness derives from. |
| `winter-github` | GitHub issue tooling. |
| `blizzard-harness` | The blizzard conventions repo, installed as an extension so its rules load into every agent context. |

### Project repos (worktreed into every feature environment)

| Repo | Role |
|------|------|
| `blizzard` | The main application — hub, runner, CLI, web app. |
| `blizzard-harness` | The same harness repo, adjacent as a project repo **for editing** — harness rules evolve through the same feature-environment flow as code. |
| `blizzard-mock` | The mock fleet: mock coding harnesses, mock hub, mock runner — see [mocking.md](./mocking.md). |

## The harness: `blizzard-harness`

Blizzard adopts **winter-canon** as its substrate: the canon defines what any harness must be, and `blizzard-harness` is blizzard's instance of one.
It follows `winter-harness` in **style** — domain-organized convention directories, routing hubs, a verifiability matrix, architectural guidance — but carries **blizzard's own rules**, not winter's.
The architecture and testing rules in this implementation plan ([architecture.md](./architecture.md), [testing.md](./testing.md)) are seeds: once the harness repo exists, those rules move there and become the canonical owners, with this plan reduced to pointers.
