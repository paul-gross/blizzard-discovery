# The mock fleet

Everything blizzard integrates with is a pluggable seam ([design/architecture.md](../design/architecture.md)), and every seam gets a controllable mock.
The mocks live in the `blizzard-mock` project repo ([ecosystem.md](./ecosystem.md)) and exist for one reason: agents building blizzard must be able to construct any state — including rare edge cases — deterministically, cheaply, and without real tokens.

## Mock coding harnesses

The harness seam accepts any coding harness — Claude Code, Codex, OpenCode.
`blizzard-mock` ships **three mock coding harness applications**, one mimicking each of those three, so the adapter layer ([design/harness-adapters.md](../design/harness-adapters.md)) is exercised against realistic per-harness surfaces (invocation shape, output format, exit behavior, resume semantics) in every test without spending real tokens.
**The prompt is the program.** A real harness turns a prompt string into behavior via an LLM; the mock turns it into behavior via `exec()` — the prompt it receives *is* Python code, run in the acquired worktree with the spawn environment.
That makes agent behavior fully simulable with no injection side channel: a test mints a workflow graph whose node prompts are the behavior scripts, and the code rides the real pipeline — hub envelope → runner → adapter → spawn — including per-attempt variation via judgement `prompt_addendum` (D-071).
A script can do anything an agent can: apply a diff and `git commit` (real commits — everything downstream of the harness seam runs for real, [verification.md](./verification.md)), fire the ask/answer protocol by invoking `blizzard ask` and exiting (parking and resume-with-answer then run end-to-end, with the resume message again arriving as code), emit a verdict or a malformed one, hang, or crash.

- A small helper library keeps scripts terse — `ask(...)`, `apply_diff(...)`, `commit(...)`, `verdict(...)`, `hang(...)`, `crash()` — with raw Python underneath for the weird cases.
- Sessions persist a state file, so a resumed script can read what it asked and act on the answer it was resumed with.
- The three per-harness mocks share the exec engine and differ only in their CLI/wire façade — the façade being exactly what the adapters are tested against.
- Arbitrary code execution is the feature, and it is fenced: the mock refuses to run unless test scaffolding marks the environment, so it can never pass as a real harness binding.

## Mock hub and mock runner

The hub and runner are built **against each other's mocks**:

- Building the **runner** → run it against the **mock hub**.
- Building the **hub** → run it against the **mock runner**.

Each mock exposes **levers** — explicit controls an agent (or test) uses to steer the counterpart very tactically: delay a response, drop a delivery, return a conflicting fact, go unreachable mid-lease, replay a message.
This turns "contrive the real daemon into a rare state" into "pull the lever that names the state," which is what makes edge-case coverage tractable for agent-driven development.

## Mock GitHub forge

A standalone mock of the GitHub API subset blizzard touches, covering **two seams with one vendor surface**: the work-source seam (issues with bodies *and comment threads*, served vendor-native — the hub's pass-through reads, D-047/D-074, ship GitHub-shaped code that only a GitHub-shaped mock exercises) and the delivery seam (PRs, merges, and the states the merge queue must survive — D-057/D-058/D-065).
It is what makes the e2e bar reachable: **agents e2e test with mocks and verify it works locally** — the full system runs with no real forge, no real tokens, no network.

Like the mock hub and runner, it is stateful and levered: externally-merged PR, merge conflict, merge rejected, comment added mid-flight, rate-limited, token rejected, unreachable.

**Its backing model is a directory of bare git repos** — the same `file://` origins the fixture workspace's worktrees push to ([verification.md](./verification.md)).
Real GitHub is an API façade over git repos, so the mock is too: issues and PR metadata are mock state, but mergeability is computed against real refs, merging a PR performs a real merge into the bare repo's `main`, and the external-merge lever is a direct push to `main` that the delivery flow must then detect.

**Build it; nothing off-the-shelf fits** (surveyed July 2026): the octokit-ecosystem mocks (kiegroup/mock-github, gimenete/mocktokit) intercept at the JS client layer and cannot stand in front of a Python daemon; octokit/fixtures is archived record-replay of canned scenarios, not stateful; generic mock servers (Mockoon, MockServer, WireMock) have no GitHub domain state, so every lever would be hand-built on top anyway; and Gitea/Forgejo are real forges whose APIs are GitHub-*like* but not GitHub-native — the wrong vendor surface for testing vendor-native pass-through.

## Mock-data CLI

A CLI tool that **creates, destroys, or resets data** inside the data stores (sqlite or postgres) of a hub or runner under test.
It is a tool *for agents* to set up test cases repeatably:

- It operates on **domain models**, not raw tables — an agent asks for "a chunk parked on a question," not for rows.
- Larger data sets are instantiated from **fixtures** — named, versioned scenarios the suite and agents share.
- Reset returns a store to a known-clean state, so every test run starts from the same ground.
