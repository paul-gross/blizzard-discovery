# Harness adapters

Blizzard is harness-agnostic. It drives Claude Code today and is designed to drive OpenCode and Codex as they mature. All three CLIs converged on the same shape — **headless run + persisted session + resume** — so one small adapter interface covers all of them.

## The adapter interface

Three functions per harness:

```
spawn(envs, prompt, session_id)  -> pid
resume_cmd(env, session_id)      -> string
verdict(output)                  -> result
```

- `spawn` starts a headless worker against a pre-assigned session id, pointed at the chunk's environment ids (a chunk may hold several — D-021), and returns its pid.
- `resume_cmd` returns the literal shell command a human runs to resume that session interactively.
- `verdict` parses the **judgement-resume reply** (D-038) into a structured result — the `<Choice>{name}</Choice>` selection plus the assessment payload (D-042). Node prompting is two-phase: the worker's own exit carries no verdict; the runner resumes the session with the judgement prompt and this function parses what comes back.

Adapters must stay **dumb**. They translate; they do not decide. All arbitration lives in the deterministic core.

**Known gap:** the interface has no automated resume-with-message operation, though the judgement prompt (D-038), answer delivery, and the CI feedback loop all need one — tracked in [runner/contracts.md](./runner/contracts.md).

## Per-harness comparison

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

- **`PreToolUse`** — can deny a tool call with feedback to the agent (this is where per-call gating of singleton resources can live).
- **`PostToolUse`** — emits the heartbeat (progress detection with no agent cooperation; see [runner/loop.md](./runner/loop.md)).
- **`SessionEnd`** — releases the lease.
- **`SessionStart`** — injects context at the top of a session.

Session handling: `--session-id` pre-assigns the id at spawn; `claude --resume <id>` is the **interactive** human takeover; `claude -p --resume` is the **automated** follow-up. Never run two processes against one session — always kill first.

### OpenCode

A client/server architecture: `opencode serve` runs a backend exposing an HTTP API, and `opencode run --attach` drives it. This gives OpenCode the **strongest live-takeover story** — a human can attach a TUI to a running backend and watch or take over without killing anything. Plugin events provide the gating hooks.

### Codex

`codex exec --json` emits JSONL events; `codex exec resume <id>` (or `--last`) does automated follow-ups; `codex resume <id>` is the interactive takeover. `--output-schema` enforces a JSON verdict and, importantly, **works on resumed runs**. Codex has the **weakest deterministic per-call gating**, so it leans hardest on the wrapper-CLI enforcement described below.

## Enforcement lives at the resource boundary, not in hooks

Because per-harness gating strength varies — Claude Code strong, Codex weak — enforcement of singleton resources cannot depend on hooks. It lives in a **harness-agnostic wrapper CLI at the resource boundary**:

- The wrapper does `acquire → heartbeat → release` against the runner's local API.
- Direct access to the guarded resource is made *impossible* — the only path to the resource is through the wrapper, so an agent cannot bypass the lease no matter how weak its harness's hooks are.

Per-harness hooks are **polish** on top of this, not the enforcement mechanism. An optional MCP lease server can provide agent-facing ergonomics (a nicer interface for the agent to request a lease), but the wrapper at the boundary is what actually guarantees single-writer access.

## Heterogeneous fleets fall out for free

Once the adapter interface is uniform, mixing harnesses costs nothing:

- **Route chunks per harness** — send some work to Claude Code, some to Codex, based on fit.
- **Retry with a different harness** as an escalation rung — a chunk that fails twice under one harness can be retried under another before it goes to a human.

## Human takeover flows

Every harness exposes an interactive resume, and escalation hands the human the exact command to run.

A chunk that fails twice or gets stuck escalates as `needs-human` — a hub-recorded state; nothing is written to the PM item (D-047) — and the escalation record carries the **literal takeover command**, for example:

```
cd <env> && claude --resume <session-id>
```

The human lands in the agent's full context, with the agent's own worktrees, finishes or unblocks the work, and requeues it. The escalation fact is recorded in the runner store **before** the command is printed — the derived status is already `needs-human` — so no supervisor process can race the human for the same chunk.

## Keeping adapters honest

CLIs change monthly. To keep adapter drift from silently breaking the fleet:

- **Adapters stay dumb** — the smaller they are, the less there is to break.
- **Pin CLI versions** — a fleet run pins the harness versions it was validated against.
- **`blizzard selftest`** runs a trivial task through each harness before any unattended period, catching a broken adapter before it wrecks a night of work.
