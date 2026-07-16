# Build, CI, and release

How `blizzard` code becomes checked, built, and released artifacts.
CI runs on **GitHub Actions**.

## One repo, one wheel

The `blizzard` repo builds a single distributable: the Python wheel containing both daemons, the CLI, and the compiled frontend.
The build pipeline compiles both Angular apps (hub app, runner app — [tech-stack.md](./tech-stack.md)) and embeds their outputs as static assets inside the wheel, so the released artifact needs no Node at install or runtime (D-061, D-096).
There is one build entrypoint an agent or workflow invokes; the exact repo directory layout is settled during bootstrap ([bootstrap.md](./bootstrap.md)).

## Branch and release model (D-098, D-104)

**Trunk-based, milestones as tags — no long-lived milestone branches.**

- `master` is the only long-lived branch: always releasable, and not branch-protected. Work arrives three ways (D-104): an agent pushing directly from a local feature environment, or a fleet chunk landing through the hub's `deliver` node in `merge-to-main` mode (the hub lands it) or `open-pr` mode (the hub parks the chunk on a PR a human resolves). Only the last leaves a PR standing for the merge gate to hold, so the other two rely on the local check run and the tiers CI runs after the push.
- A **milestone is a tag** (`milestone:mvp` → `v0.1.0`): pushing the tag *is* cutting the milestone build — CI builds and publishes the release from it. Prerelease candidates are `v0.1.0-rc.N` tags, cut whenever a shareable build is wanted.
- Versioning: `0.<milestone>.<patch>` while pre-1.0.
- **Backport branches are lazy and short-lived.** Only if a released milestone needs a fix after master has moved on: cut `release/v0.1` from the tag, cherry-pick, tag `v0.1.1` from it, delete the branch when support ends. Nothing ever merges back to master — work never left it.

## CI workflows (GitHub Actions)

| Trigger | Runs |
|---------|------|
| PR to master | ruff, pyright, eslint ([tech-stack.md](./tech-stack.md)), client drift check, unit + component tiers. The merge gate. |
| Push to master | The PR gate plus the service tier and the crash sweep ([crash-testing.md](./crash-testing.md)); publishes a dev build (`0.<next>.0.dev<run>`) for fleet dogfooding. |
| Tag `v*` | Full suite including e2e, wheel build with embedded frontend, GitHub Release. |

All tiers run seams-mocked ([testing.md](./testing.md)) — CI never needs tokens, a real forge, or the network beyond dependency installs.

## Store migrations (D-099)

> **Graduated** to **blizzard-harness**.
The migration policy — Alembic, manual, CLI-driven, never automatic; two independent trees; paired `upgrade`/`downgrade`; `blizzard hub init` / `blizzard hub migrate` and the runner equivalents; daemons refuse to run on revision skew — is owned by `blizzard-harness:/standards/persistence.md` (`bzh:manual-migrations`).
The CLI verb tables are owned by [design/cli.md](../design/cli.md).

## Generated API client (D-100)

> **Graduated** to **blizzard-harness**.
The rule — the Angular apps consume an openapi-ts-generated TypeScript client from the FastAPI OpenAPI specs (never hand-written fetch), the client is committed, and the gate regenerates-and-diffs to make drift impossible — is owned by `blizzard-harness:/standards/frontend.md` (`bzh:generated-client`).
