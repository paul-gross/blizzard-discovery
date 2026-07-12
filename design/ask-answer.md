# The ask/answer protocol

This is the one genuinely new primitive blizzard introduces: a worker facing an undecidable choice asks a human a question **without dying for it**, and the chunk parks — a derived `waiting_on_human` condition alongside running and finished — until the answer arrives and the session is resumed around it.

The protocol ships in `milestone:mvp` against the colocated hub (D-015): a question is a hub row from day one, asks surface in `blizzard hub status`, and answers come from the terminal via `blizzard hub answer`. What the remote slice (`milestone:centralized-hub`) adds is *reach* — fanning asks out to people who are not at the machine (the board, the Telegram bot — D-031), and carrying their answers back.

## Worker convention

The node prompt instructs the worker: on an undecidable choice, run

```
blizzard runner ask "question" --options "a|b"
```

and **end the turn**. The worker does not block, spin, or poll. It asks and exits.

Because `blizzard runner ask` hits the runner's local API *before* the worker exits, the ask is a durable runner-store fact by the time the process ends — which is how the supervisor tells "parked on a question" apart from "died without a verdict" (D-009): a worker exit with no verdict but an open ask parks the chunk; the same exit with neither is a failure.

### The harness's native question tool

`blizzard runner ask` deliberately displaces the harness's native ask-the-user tool (Claude Code's `AskUserQuestion` and its equivalents): a headless worker has no attached human, so the native tool is unavailable in print mode and reaching for it comes back as a tool error, not a hanging prompt. Three layers contain a worker that tries anyway. First, the node prompt names `blizzard runner ask` as the channel (the convention above). Second, a harness whose native question genuinely blocks is caught by heartbeat staleness: a blocked agent makes no tool calls, its lease goes stale, and REAP treats it like any stall — only a recorded ask fact stops the reap clock, and a native question records none. Third, a worker that fumbles the tool error and exits holds neither an ask fact nor a verdict, which is an ordinary failure (D-009) — retried, then escalated. Worst case is one wasted attempt; never a hang, never a false success. On Claude Code, a `PreToolUse` deny hook redirects the fumble in the moment it happens ([harness-adapters.md](./harness-adapters.md)) — polish on top of these layers, not the guarantee.

## The flow, end to end

1. **Ask.** `blizzard runner ask` hits the runner's local API (D-023); the runner records the ask fact and forwards the question to the hub, where it becomes a durable row — `{question_id, chunk_id, session_id, runner_id, options, asked_at}` — and the worker exits. The question's open/answered state is derived from the presence of an answer row, never stored (D-004).
2. **Park.** The supervisor sees the worker end with an open ask on its lease; the recorded ask is the fact from which the chunk's `waiting_on_human` status derives, and the reap clock **stops** for that chunk. A parked chunk is not stalled and is never reaped for inactivity.
3. **Fan out** (`milestone:centralized-hub`). The question appears on the board and is pushed to the Telegram bot (D-031) as a notification with inline answer buttons — the best phone UX, a pattern borrowed from Archon's chat-platform adapters. In the MVP, this step is simply `blizzard hub status` at the machine.
4. **Answer.** `blizzard hub answer` (or, remotely, the bot POSTing `/questions/{id}/answer`) writes the answer row at the hub. The hub applies **first-write-wins CAS**: if two people answer simultaneously, exactly one wins and the loser is told "already answered by X."
5. **Propagate.** The hub emits `question.answered` on the stream. The runner picks it up via its outbound connection or its next PULL poll. Because the answer is a **row, not an in-flight message**, every hop survives a crash — the answer is durable and re-deliverable.
6. **Resume.** The supervisor delivers the answer by **resuming the dormant session**:

   ```
   cd <env> && claude -p --resume <sid> "Answer from <who>: <answer>. Continue."
   ```

   The agent is **reconstituted around the answer**, not messaged mid-flight. The delivery is recorded as a fact — the chunk's derived status returns to running. (Delivery goes through the adapter's `resume` operation — D-050, [runner/contracts.md](./runner/contracts.md).)
7. **Confirm.** The answering surface confirms "delivered, agent resumed," closing the trust loop for the human who answered.

## Why ask-and-exit, not blocking

An alternative **blocking mode** was considered: `blizzard runner ask --wait` would have the CLI long-poll, and the human's answer would become the command's stdout back into the still-live session. This was **deferred** (D-010). Ask-and-exit is crash-equivalent to everything else in the system: it survives reboots, it survives crashes, and it costs nothing while a chunk is parked overnight. A blocking call holds a live process open for hours waiting on a human, which is exactly the fragility the rest of the design avoids.

A possible later **compromise**: `blizzard runner ask --wait --timeout` that blocks for a bounded time and falls back to parking if the human does not answer within the window — the ergonomics of blocking for the common fast answer, with the durability of parking for the slow case.
