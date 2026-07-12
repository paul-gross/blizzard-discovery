# Question and answer models

The ask/answer rendezvous rows, born at the hub — the protocol itself is [ask-answer.md](../ask-answer.md). Part of the [domain model](./index.md).

| Property | Notes |
|----------|-------|
| `question_id` | The question's identity — the answer row and the `question.answered` event key off it. |
| `chunk_id` | The chunk parked on this question — the fact its `waiting_on_human` status derives from. |
| `session_id` | The dormant agent session to resume around the answer. |
| `runner_id` | The runner holding the session — where the answer must be delivered. |
| `options` | The choices offered by `blizzard runner ask --options "a\|b"`; what the board and bot render as buttons. |
| `asked_at` | When the ask was recorded; the reap clock stops for the chunk from here. |
| open / answered | **Derived** from the presence of an answer row (D-004). |
| answer: `answered_by` | Who answered — first-write-wins CAS means exactly one answer row ever exists; losers are told who won. |
| answer: `answer` | The chosen option or free text, carried into the resume prompt ("Answer from \<who\>: \<answer\>. Continue."). |
| answer: `answered_at` | When the winning write landed. |
| delivery | The resume-with-answer is recorded as its own fact — it flips the chunk's derived status back to running. |
