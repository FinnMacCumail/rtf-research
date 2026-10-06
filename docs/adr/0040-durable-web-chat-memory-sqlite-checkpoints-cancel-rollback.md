# ADR-0040 — Durable Web-Chat Memory: SQLite Checkpoints, and a Cancel That Rolls the Thread Back

## Status

**Accepted (October 2026).** Supersedes decision 1 of [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md)
("memory is process-local by design") and removes its "memory does not survive a backend
restart" limitation. The web layer's LangGraph checkpoints now live in SQLite; a cancelled turn
is removed from the thread instead of being left in it. The agent the harnesses measure is
still unchanged: the CLI and both evaluation harnesses keep per-process memory, and `query()`,
the system prompt, middleware and skills are untouched.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`5271e69`, merged `b2a02ac`; design in `PRPs/netbox-web-chat-persistence.md`, gate results and
the defect record in `docs/development/2026-10-05_web-chat-persistence.md`)
**Bears on**: [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md) (the
layer this changes), [ADR-0018](0018-langgraph-stategraph-architecture.md) (LangGraph state and
checkpointing as the memory substrate), [ADR-0038](0038-local-frontier-model-viable-cost-not-capability.md)
(every token the thread carries is paid for again on the next call)

## Context

ADR-0039 kept `InMemorySaver` because the web layer needed no new memory machinery and the
browser already held the transcript. Using the chat for a week showed why that is the wrong
split: after any backend restart the browser shows a conversation the model has never seen.
The operator reads an answer, asks a follow-up, and the model starts cold — the UI displays
claims the model cannot see, which is exactly the contamination shape the session harness
documented (RESEARCH_LOG 2026-09-25), with the person rather than the model holding the stale
population.

A second defect was measured while designing the fix. Cancelling a turn after its first tool
result left the thread as `[user]` with no answer and `state.next` pointing at the unfinished
node. The next message started a fresh run (LangGraph treats non-`None` input as new input),
but the orphaned question stayed in every later prompt, and so did any tool result that had
already landed.

An external report proposed seven persistence steps. Steps 1–4 (durable checkpointer, injected
rather than hard-coded; cancel rollback; thread-aware resume; server-side delete) are adopted
here. Importing browser transcripts into new threads and a catalogue database were declined:
browser transcripts hold 2,000-character tool previews, so an imported thread would contain
claims without evidence.

## Decision

1. **The checkpointer is injected; the default is still `InMemorySaver`.**
   `NetBoxDeepAgent(checkpointer=...)`. `python -m src.main`, `tests/eval/*` and every existing
   caller are unchanged, so the harness results remain comparable. Only `src/web/` constructs a
   durable saver.
2. **`AsyncSqliteSaver` for the web process, and startup fails if it cannot open.**
   `WEB_CHECKPOINT_DB` (default `data/web_checkpoints.sqlite`). A web server that cannot persist
   memory while the browser shows transcripts would silently reintroduce the failure being fixed,
   so there is no fallback to memory. Added `langgraph-checkpoint-sqlite==3.1.1`; nothing else
   was upgraded (`langgraph-checkpoint` stayed at 4.1.1).
3. **Cancel rolls the thread back; it does not resume.** `TurnRunner` snapshots the thread's
   message ids before each turn and, on user cancel or agent error, removes everything newer
   with `RemoveMessage` via `aupdate_state` while it still holds the turn lock. The browser keeps
   the cancelled bubble for the record; the model never sees the question again.
4. **The rollback is attributed to the node that routes to `END`, discovered from the graph.**
   `aupdate_state(as_node=X)` schedules the successors of `X`. The design assumed `as_node="model"`;
   that leaves `next == ('tools',)`, a pending task. In the real DeepAgents graph the only edge to
   `__end__` leaves `FilterErrorRecoveryMiddleware.after_model`, not `model`. The code reads
   `agent.get_graph().edges` once and falls back to `model`.
5. **Two passes, because finished parallel tool calls are pending writes.** See Evidence.
6. **`resumed` carries the conversation id; `DELETE /conversations/{id}` forgets a thread**
   (checkpoints and ledger; `409` while its turn is running). The UI drops late `resumed` replies
   for other threads and calls the delete fire-and-forget.

## Evidence

**Restart survival, real stack** (llama-server, Qwen3.8-Flash-Next UD-Q4_K_XL, NetBox demo
data; one run each):

| Check | Result |
|---|---|
| Turn 1 on a fresh thread ("one Dunder-Mifflin site, name (id)") | `resumed known=false`; **DM-Akron (2)**, 4 model calls, 3 tool calls, 66.8 s; database created |
| **Backend killed and restarted**, same thread | transcript endpoint returns both messages; `resumed known=true turns=1`; follow-up "repeat the site name and id without tools" answered **DM-Akron (2)** with **0 tool calls, 1 model call, 5.0 s**; 41 checkpoints on disk for the thread |
| Unwritable `WEB_CHECKPOINT_DB` | process exits 3 with the OS error; nothing listening |
| Cancel after the first tool result, then a new message | `cancelled` with `rolled_back_messages=3`; transcript empty; `resumed known=true turns=0`; the next call's input is `[system, human]` only, **context 8,738** — the fresh-thread baseline |

**The defect the first cancel test hid.** The first real cancel looked clean — the transcript
was empty and the next turn ran — but LangSmith showed that turn's model input as
`[system, tool (7,439 chars), human]`, **context 11,060** against the 8,738 baseline. A tool
result from the cancelled turn had survived, and the transcript view hides tool messages, so
nothing in the UI could show it. Reproduced with a `Send`-based toy graph on both savers:

- The cancelled model call had issued **two tool calls**, run as separate tasks. `read_file`
  finished in 10 ms; `netbox_get_objects` was cancelled. The finished result existed only as a
  **pending write** on the step, not in the committed `messages` channel.
- `aget_state(cfg)` merges pending writes into `values`, so the naive orphan list included the
  pending id; `RemoveMessage` acts on the committed channel and that id was not there. The toy
  graph raised *"Attempting to delete a message with an ID that doesn't exist"*; the real agent
  passed silently and carried the write into the next turn.
- First fix: compute orphans from the state pinned to the checkpoint id (committed only). On the
  toy graph `aupdate_state` dropped the pending write; on the real agent it did the opposite —
  `aupdate_state` applies the pending writes of tasks that have already finished *into the new
  checkpoint*, so the rollback itself committed the `read_file` result. A post-rollback check
  logged `Rollback incomplete leftover=1`; that log line is how the defect was caught.
- Final fix: pass 1 removes the committed orphans (and, as a side effect, commits any finished
  tool results and clears the pending step); pass 2 re-reads and removes everything not in the
  pre-turn id set. A regression test with parallel `Send` tool tasks pins it.

Installed versions, checked rather than read from the lock file: deepagents 0.7.5,
langgraph 1.2.5, langgraph-checkpoint 4.1.1, langgraph-checkpoint-sqlite 3.1.1. The repository's
`uv.lock` names newer versions that were never installed.

## Consequences

### Positive
- A conversation the browser shows is a conversation the model remembers, across restarts.
  The "memory lost" banner now appears only for threads created before 2026-10-05.
- Stop leaves no trace in the thread: no orphaned question, no half-landed tool result. The
  contamination finding has one fewer way to happen.
- The harnesses and the CLI measure exactly what they measured before.
- A LangGraph behaviour worth knowing is now recorded: **a cancelled parallel tool step leaves
  finished results as pending writes, and `aupdate_state` commits them.** Any rollback that
  checks only the committed channel will miss them.

### Negative / limitations
- **The token ledger is still in memory.** Its numbers also live in the browser and in
  LangSmith, but a restart resets the server's cumulative view.
- **Conversations from before 2026-10-05 cannot be recovered**; importing transcripts was
  declined on purpose.
- **Rollback discards partial work.** A cancelled 200-second tool loop is thrown away rather than
  resumed (`Command(resume=...)` is the follow-up).
- **The only guard against an incomplete rollback is a warning log line.** It is what found the
  defect, and it is still the only thing watching.
- Every stack timing is n=1.

## Relationship to ADR-0039

ADR-0039 argued that the web layer could be built without touching the agent, and listed
process-local memory as a design choice. This ADR keeps the first claim and withdraws the
second: memory is part of what the operator relies on, so it has to be at least as durable as
the transcript they are reading.

## References

- [Phase 5 → Web Chat — Serving the Local Agent](../phases/phase-5-production-deepagents/web-chat.md)
- [ADR-0039 — Web Serving Layer](0039-web-serving-layer-serialised-turns-server-token-counts.md)
- [ADR-0041 — Reject `__in`](0041-in-lookup-silently-ignored-reject-in-validator.md) (found on
  the first long thread this persistence made visible)
- Repository: `src/web/checkpoints.py`, `src/web/session.py` (`_safe_rollback`),
  `src/agents/netbox_agent.py` (`rollback_turn`, `_rollback_node`),
  `tests/test_web_checkpoints.py::test_rollback_drops_pending_parallel_tool_result`
- Upstream: `langgraph/pregel/main.py` (`aupdate_state`: "apply writes from tasks that already
  ran"); `langgraph/pregel/_loop.py` (`_first`: non-`None` input starts a new run)
