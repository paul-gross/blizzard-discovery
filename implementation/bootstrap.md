# Bootstrap sequencing

The order of operations from "discovery ends" to "the fleet builds blizzard features."
Each phase has an exit criterion; a phase does not start until the previous one's criterion holds, because each phase is load-bearing for the next.

## 1. Stand up `blizzard-workspace`

Fork the winter workspace lineage ([ecosystem.md](./ecosystem.md)): declare the project repos (`blizzard`, `blizzard-harness`, `blizzard-mock`) and standalone extensions in `.winter/config.toml`, run `winter ws init`, and confirm feature environments materialize.

**Exit:** an agent spawned in a feature env sees all three project worktrees and the extension context loads.

## 2. Seed `blizzard-harness`

Instantiate the harness from winter-canon, in winter-harness's style: domain hubs, the verifiability matrix skeleton, architectural guidance.
The rules drafted in this plan graduate here and become canonical — [architecture.md](./architecture.md), the testing tiers, the toolchain — with this plan's files reduced to pointers.

**Exit:** a cold agent in a feature env can answer "what rules is my code held to, and how do I verify a change" from the harness alone.

## 3. Author the service manifests

Write the winter-service-docker/tmux configs for the verification stack ([verification.md](./verification.md)): postgres first, then a slot per service that later phases fill in (mock forge, hub, runner).

**Exit:** `winter service up <env> --wait` brings up postgres, port-band isolated, in two envs at once.

## 4. Build `blizzard-mock` first

The mocks are load-bearing for everything after them, so they precede the app:

1. **Mock GitHub forge** backed by bare git repos, with its first levers ([mocking.md](./mocking.md)).
2. **Fixture-workspace scaffold** — bare `file://` origins plus a generated winter workspace ([verification.md](./verification.md)).
3. **Mock harness exec engine** (the prompt-is-the-program core) with the Claude Code façade; the Codex and OpenCode façades follow when adapters need them.
4. **Mock-data CLI** skeleton — it grows alongside the domain models it operates on.

**Exit:** a scripted prompt, run through the mock harness in a fixture-workspace env, lands a commit that the mock forge merges to bare `main` — no blizzard code involved yet.

## 5. Scaffold `blizzard`

The uv project and package skeleton, the two FastAPI apps and CLI entrypoints, both Alembic trees with `init`/`migrate` verbs, the Angular workspace (two thin apps + shared libraries, design tokens ported from the mockup), the openapi-ts generation script, and the CI workflows ([build.md](./build.md)) — **all quality gates on from the first commit**, when they are cheapest to satisfy.

**Exit:** the PR gate is green on the skeleton; `blizzard hub init . && blizzard hub` serves an empty board from an embedded frontend build.

## 6. Walking skeleton

The [verification.md](./verification.md) acceptance loop as the first feature: one chunk traveling ingest → acquire → mock-scripted commit → deliver → landed in the bare origin, with facts deriving `done`.
Thin every layer rather than deep any one: minimal graph, minimal store, one adapter, one board view.

**Exit:** the acceptance loop passes end-to-end locally; it becomes the standing e2e smoke test.

## 7. Feature build

From here the roadmap drives ([product/roadmap.md](../product/roadmap.md)), the fleet-to-be's own workflow conventions apply, and every feature lands against the walking skeleton's always-working baseline.
