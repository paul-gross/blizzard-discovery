# Coding-harness adapters

Blizzard is coding-harness-agnostic. It drives Claude Code today and is designed to drive OpenCode and Codex as they mature. All three CLIs converged on the same shape — **headless run + persisted session + resume** — so one small adapter interface covers all of them.

## The adapter interface

Four functions per coding harness:

```
spawn(envs, prompt, session_id)     -> pid
resume(env, session_id, message)    -> pid
resume_cmd(env, session_id)         -> string
verdict(output)                     -> result
```

- `spawn` starts a headless worker against a pre-assigned session id, pointed at the chunk's environment ids (a chunk may hold several — D-021), and returns its pid.
- `resume` delivers a message into an existing session headlessly and returns the new pid (D-050) — the operation behind the judgement prompt (D-038), answer delivery, and the CI feedback loop. Never run against a live process — always kill first.
- `resume_cmd` returns the literal shell command a human runs to resume that session interactively.
- `verdict` parses the **judgement-resume reply** (D-038) into a structured result — the `<Choice>{name}</Choice>` selection plus the assessment payload, carried as coding-harness-native structured output (D-042/D-056). Node prompting is two-phase: the worker's own exit carries no verdict; the runner resumes the session with the judgement prompt and this function parses what comes back.

Adapters must stay **dumb**. They translate; they do not decide. All arbitration lives in the deterministic core.

## Per-coding-harness comparison

| Aspect | Claude Code | OpenCode | Codex |
|--------|-------------|----------|-------|
| Headless run | `claude -p --output-format json` | `opencode run` | `codex exec --json` (JSONL events) |
| Pre-assign session | `--session-id` | server-assigned via API | (session id from run) |
| Automated follow-up | `claude -p --resume <id>` | `opencode run --attach` | `codex exec resume <id>` / `--last` |
| Interactive takeover | `claude --resume <id>` | attach a TUI to the running backend | `codex resume <id>` |
| Enforced structured output (the judgement reply, D-038) | `--output-format json` | plugin/API shape | `--output-schema` (works on resumed runs — exactly what the two-phase judgement resume needs) |
| Deterministic gating | richest — full hook suite | plugin events | weakest — relies on wrapper enforcement |
| Live-takeover story | resume (kill first) | strongest — attach a TUI to a live backend | resume |

### Claude Code

The richest deterministic hook surface:

- **`PreToolUse`** — can deny a tool call with feedback to the agent (this is where per-call gating of singleton resources can live). Also the redirect for the native question tool: deny `AskUserQuestion` with "you are running unattended — use `blizzard runner ask '…' --options 'a|b'` and end your turn," converting a worker's fumble into an immediate correct ask ([ask-answer.md](./ask-answer.md)).
- **`PostToolUse`** — emits the heartbeat (progress detection with no agent cooperation; see [runner/loop.md](./runner/loop.md)).
- **`SessionEnd`** — releases the lease.
- **`SessionStart`** — injects context at the top of a session.

Session handling: `--session-id` pre-assigns the id at spawn; `claude --resume <id>` is the **interactive** human takeover; `claude -p --resume` is the **automated** follow-up. Never run two processes against one session — always kill first.

### OpenCode

A client/server architecture: `opencode serve` runs a backend exposing an HTTP API, and `opencode run --attach` drives it. This gives OpenCode the **strongest live-takeover story** — a human can attach a TUI to a running backend and watch or take over without killing anything. Plugin events provide the gating hooks.

### Codex

`codex exec --json` emits JSONL events; `codex exec resume <id>` (or `--last`) does automated follow-ups; `codex resume <id>` is the interactive takeover. `--output-schema` enforces a JSON verdict and, importantly, **works on resumed runs**. Codex has the **weakest deterministic per-call gating**, so it leans hardest on the wrapper-CLI enforcement described below.

## Hook delivery and heartbeat correlation

Hooks are configuration, not ambient behavior — something must place them where the spawned process reads them, and what that something is differs per coding harness.

**Delivery is spawn-scoped and adapter-owned.** The Claude Code adapter passes a runner-owned settings file on the command line — `claude -p --session-id <sid> --settings <blizzard-worker-settings.json>` — carrying the worker hook set: the `PostToolUse` heartbeat, the `AskUserQuestion` deny, and the rest. The file ships with the adapter and is versioned with it; nothing is materialized into the environment or checked into project repos (repos know nothing about the fleet), and a human's own `claude` session in the same worktree carries no fleet hooks. Codex has no per-tool hook configuration to deliver (its `notify` program is coarse, set via config overrides); OpenCode's plugins are in-process modules discovered from config — a process-per-worker topology points at a blizzard plugin directory at spawn, while a shared `opencode serve` backend configures the plugin once, server-side.

**Correlation keys on runner-minted identity, echoed back — never self-reported.** Lease id, session id, and epoch are recorded together at FILL, before the process exists. The transport differs per coding harness: Claude Code hooks run as child processes of the worker and inherit `BLIZZARD_LEASE_ID`/`BLIZZARD_SESSION_ID` from the spawn environment, so `blizzard runner heartbeat` posts to the right lease with no arguments — and inheritance is per process tree, so concurrent workers on one machine cannot misattribute a heartbeat to a sibling. Codex needs no ambient identity at all: the runner is the parent holding the `codex exec --json` stdout pipe, and the pipe *is* the identity. A shared OpenCode server sees one environment for every session, so its plugin correlates by the session id in each event payload, resolved to the lease through the runner store.

**Takeover sessions carry no hooks.** Interactive resumes — `blizzard runner takeover` ([cli.md](./cli.md)) or the pasted resume command — launch without the worker settings file, and that is correct rather than an omission: a parked chunk holds no live lease (D-035), so there is nothing to heartbeat (a takeover heartbeat would only draw `410 Gone` from the local API).

All three mechanics are per-coding-harness surface that drifts monthly — `blizzard runner selftest` verifies them before an unattended period; this document does not.

## Enforcement lives at the resource boundary, not in hooks

Because per-coding-harness gating strength varies — Claude Code strong, Codex weak — enforcement of singleton resources cannot depend on hooks. It lives in a **coding-harness-agnostic wrapper CLI at the resource boundary**:

- The wrapper does `acquire → heartbeat → release` against the runner's local API.
- Direct access to the guarded resource is made *impossible* — the only path to the resource is through the wrapper, so an agent cannot bypass the lease no matter how weak its coding harness's hooks are.

Per-coding-harness hooks are **polish** on top of this, not the enforcement mechanism. An optional MCP lease server can provide agent-facing ergonomics (a nicer interface for the agent to request a lease), but the wrapper at the boundary is what actually guarantees single-writer access.

## Heterogeneous fleets fall out for free

Once the adapter interface is uniform, mixing coding harnesses costs nothing:

- **Route chunks per coding harness** — send some work to Claude Code, some to Codex, based on fit.
- **Retry with a different coding harness** as an escalation rung — a chunk that fails twice under one coding harness can be retried under another before it goes to a human.

## Human takeover flows

Every coding harness exposes an interactive resume, and escalation hands the human the exact command to run.

A chunk that fails twice or gets stuck escalates as `needs-human` — a hub-recorded state; nothing is written to the PM item (D-047) — and the escalation record carries the **literal takeover command**, for example:

```
cd <env> && claude --resume <session-id>
```

The ergonomic path is `blizzard runner takeover <chunk id>` ([cli.md](./cli.md)): the CLI resolves the chunk's environment, session, and coding harness through the runner's local API, records the takeover fact, and execs the adapter-composed interactive command in the operator's terminal — correct by construction, no hand-typed session ids or paths. The literal command in the escalation record is the fallback when the CLI is not at hand.

The human lands in the agent's full context, with the agent's own worktrees, finishes or unblocks the work, and requeues it. The escalation fact is recorded in the runner store **before** the command is printed — the derived status is already `needs-human` — so no supervisor process can race the human for the same chunk.

Takeover is **kill-then-resume, not attach**, on two of the three coding harnesses: a headless `claude -p` or `codex exec` run is a one-shot process with no server to join — the persisted session, not the process, is the handoff artifact. OpenCode's live TUI attach is real but stays coding-harness-specific polish: a question answered over an attach leaves no hub row and no ask fact, so the reap clock never stopped and REAP can kill the session mid-conversation. The escalation path — park, record, resume command — is the only takeover the fleet can see.

The resumed interactive session runs **outside the runner's supervision**: the parked chunk holds no live lease (D-035), so nothing heartbeats the human's process, nothing reaps it, and its exit declares nothing — exit-is-done semantics apply only to runner-spawned headless workers. Hand-back is the explicit requeue, which mints a fresh lease; the human should end their interactive session before requeuing, honoring the one-process-per-session rule above.

## Keeping adapters honest

CLIs change monthly. To keep adapter drift from silently breaking the fleet:

- **Adapters stay dumb** — the smaller they are, the less there is to break.
- **Pin CLI versions** — a fleet run pins the coding-harness versions it was validated against.
- **`blizzard runner selftest`** runs a trivial task through each coding harness before any unattended period, catching a broken adapter before it wrecks a night of work.
