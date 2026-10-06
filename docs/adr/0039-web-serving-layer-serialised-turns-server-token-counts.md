# ADR-0039 — A Web Serving Layer Over the Unchanged Agent: Serialise Turns, Trust the Server's Token Counts

## Status

**Accepted (September 2026).** The NetBox DeepAgents agent had been reachable only through a
Rich CLI and the evaluation harnesses. It now has a browser front end and a WebSocket back end,
built so that **nothing the harnesses measure changed**: `query()`, the system prompt, the
middleware stack and the skills are byte-identical to the commit the v5 and session-harness
results were recorded against. The interface is recorded as an ADR because building it forced
three decisions with measurable consequences — how to share one llama-server slot, where token
and context numbers come from, and how streaming interacts with a model that is silent for
minutes at a time — and because the last of those surfaced a failure the non-streaming harness
could never have shown.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`b85cb1c`, merged `64ad9c2`; design in `PRPs/netbox-web-chat.md`, gate results in
`docs/development/2026-09-28_web-chat.md`, run book in `docs/setup/web-chat.md`)
**Bears on**: [ADR-0038](0038-local-frontier-model-viable-cost-not-capability.md) (local
inference is cost-bound, and this layer is where that cost is felt),
[ADR-0029](0029-langsmith-observability-platform.md) (per-turn trace links),
[ADR-0026](0026-claude-sdk-project-requirements-package.md) (the Phase 4 Claude SDK chatbox whose
Nuxt front end and chunk protocol were ported),
[ADR-0022](0022-interactive-cli-architecture-design.md) (which listed "evolution from CLI to
web-based interface" as a future consideration — now done, against a different agent)

## Context

Three facts shaped the design, all already recorded in this programme:

- **Turns take minutes.** ADR-0038 measured 35–486 s per turn on the local 176B model, decode
  at 12–15 t/s, and a cold first turn that pays an ~8.7k-token prefill in one call. The CLI
  prints nothing until the whole answer exists. For a person, streaming and a visible tool-call
  feed are not polish; they are the difference between a usable tool and a hung terminal.
- **One slot.** `scripts/serve_qwen4exp.sh` runs llama-server without `--parallel`, and the
  NetBox MCP client is a stdio subprocess. Two concurrent requests would be queued by the
  server anyway and would thrash the prefix cache that serves ~85% of prompt tokens.
- **Conversation hygiene is a correctness control.** The session harness (RESEARCH_LOG
  2026-09-25) showed the same 11 questions scoring 0.750 or 0.950 depending only on order,
  because a later turn reused a wrong population inherited from an earlier one. "New
  conversation" is therefore not a convenience; it is how an operator breaks contamination.
  A gauge showing how much context a thread carries makes that decision informed.

The Phase 4 Claude SDK chatbox (ADR-0026) already had a working FastAPI + WebSocket + Nuxt 3
implementation of the transport. Its `{type, content, completed, metadata}` chunk protocol and
front-end composables were ported rather than redesigned.

## Decision

1. **One agent per process; the browser conversation id is the LangGraph `thread_id`.** The
   agent already had `InMemorySaver` and a per-call `thread_id` override; the web layer needed no
   new memory machinery. Memory is process-local by design (no durable checkpointer); the UI
   shows a "server memory lost" banner after a restart.
   *Superseded 2026-10-05 by [ADR-0040](0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md):
   the web process now injects an `AsyncSqliteSaver`; the CLI and harnesses keep `InMemorySaver`.*
2. **Turns are serialised with one `asyncio.Lock`, not with `--parallel`.** Waiting clients are
   told how many turns are ahead of them. Cancel is `Task.cancel()` on the consumer of
   `agent.astream()`, which closes the HTTP stream and frees the slot. Adding server slots would
   split the KV budget on a model where that is unmeasured.
3. **A second streaming path, `stream_events()`, using `stream_mode=["messages","updates"]`.**
   `query()` is untouched so the harnesses keep measuring exactly what they measured before.
   "messages" yields token deltas and the trailing usage chunk; "updates" yields tool calls and
   tool results by node. Tool results are classified by the existing
   `TOOL_VALIDATION_ERROR:` / `TOOL_API_ERROR:` prefixes and previewed at 2,000 characters.
4. **Token and context numbers come from the server, or not at all.** Every model call reports
   llama-server's `usage` (`prompt_tokens` is the resident context; `cached_tokens` the prefix
   served from cache; `completion_tokens` includes reasoning) and llama.cpp's `timings` block
   (prefill and decode t/s). Two things had to change to get them: `stream_usage=True`, which
   langchain-openai auto-enables **only** for the default OpenAI URL, and a 12-line `ChatOpenAI`
   subclass that keeps `timings`, which the base class drops. No client-side estimate is used
   anywhere; `count_tokens_approximately` undercounts this agent's prompts ~19× because the
   system prompt, tool schemas and skills are injected at call time.
5. **The streaming silence watchdog is disabled for the llama.cpp backend.** See Evidence.
6. **A LangGraph root `run_id` is minted per turn**, so the LangSmith trace of a turn is
   addressable before it finishes; the footer links to it. Older turns are backfilled from
   LangSmith by `thread_id` metadata, matched to turns by nearest `end_time`.
7. **Single-operator scope.** Bound to `127.0.0.1`; no auth, no model switching (one model is
   loaded), no reasoning display (ChatOpenAI drops `reasoning_content`). Ports are defined once
   each: 8010 back end, 3010 front end.

## Evidence

**The transport works against the real stack** (llama-server `b11053`, Qwen3.8-Flash-Next
UD-Q4_K_XL, NetBox demo data; one run each):

| Check | Result |
|---|---|
| "List the sites for tenant Dunder-Mifflin with their status" | 167.2 s, 4 model calls, 3 tool calls, correct (14 sites, all Active) |
| Streaming tool-call parsing under `--jinja` | all 3 `tool_use` chunks carried well-formed dict args; all results `ok` |
| Context cross-check against the server | mid-turn `/slots`: 8,949 prompt / 8,739 cached; the call's usage chunk: 8,953 / 8,739 |
| Second client sending during a turn | `queued`, three heartbeats, ran only after the first turn's `done`; 2.3 s with 8,719 cached (the shared prefix) |
| Follow-up on the same thread | answered from memory, 0 tool calls, 28.7 s; cached 8,719 of 10,889 because another thread had run in between |
| Cancel at 25 s into a cross-site query | `cancelled` chunk 49 ms after the request; `/slots` `is_processing:false` |

**The first user turn through the browser died, and the harness could never have shown why.**
*"How many power outlets does each Halvorsen Logistics PDU provide, and how many PDUs are
there?"* ran 4 m 06 s, 6 model calls, 7 tool calls, then failed with
`StreamChunkTimeoutError: No streaming chunk received for 120.0s ... chunks_received=0`. The
LangSmith timeline: calls 1–6 took 11–25 s on 8.7k–10.9k of context; the 7th tool result (the
tenant's device list) was **64,199 characters**; the 7th model call received **0 chunks in
120.2 s**. langchain-openai ≥1.2 arms a 120 s gap-between-parsed-chunks watchdog on *async
streaming* calls only, and llama-server emits nothing while it prefills. Every LLM run in every
earlier trace in this programme is `stream=False` — the CLI and both harnesses call the model
without streaming, so the identical prompt on 2026-09-24 simply waited 217 s and succeeded.

| | first run (watchdog on) | replay (watchdog off) |
|---|---|---|
| model calls / tool calls | 6 / 7, then killed | 7 / 6 |
| 7th call | 0 chunks in 120.2 s | **19,976 new tokens prefilled at 119.6 t/s ≈ 167 s of silence**, then streamed |
| context end | 10,867 | 30,673 (23.4% of 131,072) |
| tokens read / cached / written | 58,566 / 56,398 / 824 | 89,080 / 67,130 / 1,258 |
| wall | 4 m 06 s → error | 324 s, correct (12 APC AP7901 PDUs, 8 outlets each) |

**Middleware costs 0.27% of wall time.** Over the user's 7-turn session (thread `cc16da50`,
1,361.5 s, 36 model calls, 31 tool calls, 105 runs per trace):

| Component | Seconds | Share |
|---|---|---|
| model generation (`LlamaCppChatOpenAI`) | 1,332.4 | 97.9% |
| NetBox tool execution | 27.9 | 2.0% |
| five `awrap_model_call` wrappers (Skills, Filesystem, SubAgent, Summarization, AnthropicPromptCaching), 36 calls | 2.60 | 0.19% (72 ms/call) |
| `awrap_tool_call` wrappers | 0.37 | 0.03% (12 ms/call) |
| hook nodes (`before_agent`, `before_model`, `after_model`), 194 runs | 0.74 | 0.05% |

The project's own middlewares (`FilterErrorRecovery`, `Metrics`, `QueryMetrics`) total 0.5 s.
The real cost of the framework middleware is the ~8.7k-token fixed prefix it injects into every
prompt — visible as cached tokens, not as seconds.

**A 7-turn session as the UI reports it**: turns of 222 / 278 / 173 / 144 / 345 / 155 / 45 s;
context 15,411 → 41,611 tokens with no compaction; roughly 847k prompt tokens read in total
(each call re-sends the whole context, which is why the cached share matters more than the raw
total); turns 6 and 7 answered with one tool call each by reusing `site_id 25` from earlier
turns — the memory working as intended, and the same mechanism that contaminated the
site-first session run.

**Trace-link backfill**: the thread-lookup query returned all 7 root runs; seeded in a browser
without links, all 7 footers matched the correct run ids in order.

## Consequences

### Positive
- The operator sees text as it is written, every tool call as it happens, and the exact
  context size the next turn will pay for — the three numbers ADR-0038 says govern cost.
- "New conversation" is now a first-class control with a gauge beside it; the contamination
  finding has a user-facing remedy.
- Token, cache and speed figures are the server's own, cross-checked against `/slots` within
  1%, replacing log-scraping regexes and a 19×-wrong in-process estimate.
- Every turn links to its LangSmith trace, including turns recorded before the feature existed.
- The evaluation harnesses are untouched and their results remain comparable.

### Negative / limitations
- **One slot means one user at a time.** A second tab queues. This is the correct behaviour for
  the hardware, not a scaling story.
- <del>**Memory does not survive a backend restart** (`InMemorySaver`). The transcript survives in
  the browser; the model's memory of it does not.</del> *Removed by ADR-0040 (2026-10-05): checkpoints
  are in SQLite and a cancelled turn is rolled back out of the thread.*
- **With the watchdog off, a genuinely hung stream relies on the Stop button.** The server is
  local and the cancel path is measured, but nothing times out automatically.
- **A cancelled call reports no tokens**: llama-server emits usage only when a call completes.
- **Backfill is time-matched** (nearest `end_time` within 5 min). Serialised turns make a swap
  impossible here; it would not be safe for a multi-slot server.
- **Reasoning text is invisible.** `completion_tokens` includes it, so "1 call, 8,192 written,
  0 characters" is what the output-budget failure looks like; the UI now shows that as an error
  rather than an empty bubble, but cannot show the reasoning itself.
- **No benchmark has been run through the web path.** The claim that scores are unaffected
  rests on `query()` being byte-identical, not on a re-run. Every timing above is n=1.

## Relationship to ADR-0038 and ADR-0032

ADR-0038 concluded that local capability is no longer the constraint, cost is. This layer is
where that cost becomes a user's waiting time, and it adds one constraint ADR-0038 could not
see: **prefill silence is a client-side hazard once you stream**. ADR-0032 recorded the only
prior middleware-latency number (+13.7% for QuickJS/PTC, deferred); the 0.27% measured here is
the baseline it should be read against — the framework's wrapper chain is not where time goes.

## References

- [Phase 5 → Web Chat — Serving the Local Agent](../phases/phase-5-production-deepagents/web-chat.md)
- [Phase 5 → Local Frontier Inference](../phases/phase-5-production-deepagents/local-frontier-inference.md)
- [ADR-0040 — Durable Web-Chat Memory](0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md)
  (supersedes decision 1); [ADR-0041 — Reject `__in`](0041-in-lookup-silently-ignored-reject-in-validator.md)
  (found through this layer)
- [ADR-0038 — Local Frontier Model: Cost, Not Capability](0038-local-frontier-model-viable-cost-not-capability.md)
- [ADR-0026 — Claude SDK Project Requirements Package](0026-claude-sdk-project-requirements-package.md)
- Repository: `src/web/` (`api.py`, `session.py`, `events.py`, `tracing.py`),
  `src/agents/llamacpp_config.py` (`LlamaCppChatOpenAI`, `stream_usage`,
  `LLAMACPP_STREAM_CHUNK_TIMEOUT_S`), `src/agents/netbox_agent.py` (`stream_events`),
  `frontend/`, `scripts/ws_smoke.py`
- Upstream: langchain-openai `BaseChatOpenAI.stream_chunk_timeout` (default 120 s, async
  streaming only); `ggml-org/llama.cpp` `tools/server/server-task.cpp` (`usage` with
  `prompt_tokens_details.cached_tokens` and `timings` on the final streamed chunk when
  `stream_options.include_usage` is set)
