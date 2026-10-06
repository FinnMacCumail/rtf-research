# Web Chat — Serving the Local Agent

Until September 2026 the NetBox DeepAgents agent was reachable in two ways: a Rich CLI that
prints nothing until the whole answer exists, and the evaluation harnesses. Both are fine for
measuring. Neither is how anyone would use an assistant whose turns take
[35 to 486 seconds](local-frontier-inference.md). This page records the serving layer that was
added on top of the agent, the two things building it revealed that the harnesses could not,
and the two things a week of using it revealed (durable memory, and a filter NetBox ignores).
The decisions are in [ADR-0039](../../adr/0039-web-serving-layer-serialised-turns-server-token-counts.md),
[ADR-0040](../../adr/0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md) and
[ADR-0041](../../adr/0041-in-lookup-silently-ignored-reject-in-validator.md).

## The rule: the agent does not change

`query()`, the system prompt, the middleware stack and the skills are byte-identical to the
commit the v5 and session-harness results were recorded against. The web layer adds a second
streaming method, `stream_events()`, and everything else lives in a new `src/web/` package and a
`frontend/` app. That rule is what keeps every number elsewhere in Phase 5 comparable.

## What the operator sees

- **Text as it is written**, with the first token typically 10 s after send on a warm cache and
  ~75 s after a server restart (the cold prefill ADR-0038 measured).
- **Every NetBox tool call as it happens**, with its arguments, and its result marked ok,
  validation error or API error — the same structured errors the filter-recovery middleware
  produces.
- **A turn footer** with elapsed time, model calls, tool calls, tokens read (and how many were
  served from the prompt cache), tokens written, the resident context and its share of the
  131,072 window, prefill and decode tokens per second, and a link to the turn's LangSmith trace.
- **A context gauge** per conversation with a marker at the compaction trigger (0.85 of the
  window, ~111k), and a divider in the transcript if compaction fires.
- **A token ledger** per conversation: cumulative read, cached, written, model calls, wall time,
  with per-turn rows.
- **Stop**, which frees the single model slot within ~50 ms, and **New conversation**.

The last two are not conveniences. The
[session harness](local-frontier-inference.md#the-session-harness-one-wrong-scope-four-wrong-answers)
showed that the same eleven questions score 0.750 or 0.950 depending only on their order,
because a later turn reused a wrong population inherited from an earlier one. A person watching
the tool feed can see a turn answer from context without querying; the gauge shows how much
context the thread is carrying; "New conversation" is the way to break it.

## Where the numbers come from

Every figure in the footer is llama-server's own. Each model call ends with a `usage` block —
`prompt_tokens` is the resident context, `cached_tokens` the prefix served from cache,
`completion_tokens` what the model wrote including its reasoning — and a llama.cpp `timings`
block with real prefill and decode rates. Getting them took two changes that are worth
recording: langchain-openai only turns on `stream_usage` for the default OpenAI URL, so with
a custom `base_url` every usage field is silently `None`; and it drops the `timings` block
unless `ChatOpenAI` is subclassed. Nothing in the UI estimates tokens client-side; the
in-process `count_tokens_approximately` undercounts this agent's prompts about 19× because the
system prompt, tool schemas and skills are injected at call time.

Cross-checked once against the server mid-turn: `/slots` reported 8,949 prompt tokens with
8,739 cached; the call's usage chunk reported 8,953 and 8,739.

## Finding one: the first real turn died, and only a streaming client could have seen it

The first question asked through the browser — *how many power outlets does each Halvorsen
Logistics PDU provide, and how many PDUs are there* — ran for 4 m 06 s through six model calls
and seven tool calls, then failed with *"No streaming chunk received for 120.0s"*. The seventh
tool result was the tenant's device list, 64,199 characters. The seventh model call had to
prefill it, llama-server sends nothing while it prefills, and langchain-openai's default
120-second watchdog on async streaming calls killed the request with zero chunks received.

The CLI and both harnesses call the model **without** streaming — every LLM run in every earlier
trace in this programme is `stream=False` — so the identical prompt on 2026-09-24 simply waited
217 s and succeeded. The watchdog is now disabled for the llama.cpp backend. Replayed:

| | first run | replay |
|---|---|---|
| outcome | killed at call 7 | 12 APC AP7901 PDUs, 8 outlets each — correct |
| call 7 | 0 chunks in 120.2 s | 19,976 new tokens prefilled at 119.6 t/s ≈ 167 s of silence |
| context end | 10,867 | 30,673 (23.4%) |
| tokens read / cached / written | 58,566 / 56,398 / 824 | 89,080 / 67,130 / 1,258 |
| wall | 4 m 06 s → error | 324 s |

The general lesson: a measurement path that differs from the product path in a property the
product depends on — here, streaming — will not find the product's failures, however many
questions it runs.

## Finding two: the middleware chain costs 0.27% of wall time

The LangSmith trace tree of a single turn holds 105 runs for 7 model calls, because five
framework middlewares (Skills, Filesystem, SubAgent, Summarization, AnthropicPromptCaching) each
wrap every call. Summed over a real 7-turn session (1,361.5 s, 36 model calls, 31 tool calls):

| Component | Seconds | Share |
|---|---|---|
| model generation | 1,332.4 | 97.9% |
| NetBox tool execution | 27.9 | 2.0% |
| model-call wrappers, 36 calls | 2.60 | 0.19% (72 ms per call) |
| tool-call wrappers | 0.37 | 0.03% |
| hook nodes, 194 runs | 0.74 | 0.05% |

The only prior middleware-latency number in this programme is the +13.7% that deferred
QuickJS/PTC (ADR-0032). The framework's wrapper chain is not where time goes; its real cost is
the ~8.7k-token fixed prefix it injects into every prompt, which the prompt cache absorbs.

## A session, as the UI reports it

The seven-turn conversation that surfaced finding one, re-run to completion:

| turn | time | model calls | tools | tokens read | context end |
|---|---|---|---|---|---|
| 1 PDU outlets and count | 222 s | 7 | 6 | 77,156 | 15,411 |
| 2 outlets in use | 278 s | 10 | 9 | 182,155 | 23,524 |
| 3 upstream feeds | 173 s | 6 | 5 | 148,433 | 25,522 |
| 4 host with one supply connected | 144 s | 4 | 3 | 107,500 | 27,830 |
| 5 Halvorsen vs NC State feeds | 345 s | 5 | 6 | 166,695 | 40,303 |
| 6 device count at HVL-SEA-DC1 | 155 s | 2 | 1 | 82,124 | 41,192 |
| 7 rack names at HVL-SEA-DC1 | 45 s | 2 | 1 | 82,887 | 41,611 |

Context climbed from 15k to 41.6k without compaction; about 847k prompt tokens were read in
total because every call re-sends the whole context, which is why the cached share, not the raw
total, is the number to watch. Turns 6 and 7 reused `site_id 25` from earlier turns with a
single tool call each — the memory working as intended, and the same mechanism that
contaminated the site-first session run.

## Memory that survives a restart, and a cancel that leaves nothing behind (October 2026)

A week of use showed the one thing the page above got wrong: after any backend restart the
browser displayed a conversation the model had never seen, and the operator's follow-up
started cold. The checkpoints now live in SQLite (`WEB_CHECKPOINT_DB`), injected into the web
process only — the CLI and the harnesses keep per-process memory, so nothing measured elsewhere
in Phase 5 changed. Verified by killing the backend mid-conversation: the thread came back
known, and the follow-up was answered from memory with **0 tool calls in 5.0 s**.

Stop now rolls the thread back as well. Before, a cancelled turn left the question and any
tool result that had already landed in every later prompt. The rollback found a LangGraph
behaviour worth recording: when a model call issues two tool calls and one is cancelled, the
finished one is a *pending write*, invisible to the committed message list and committed by the
very `aupdate_state` call that performs the rollback. The first "clean" cancel therefore carried
a 7,439-character tool result into the next turn (context 11,060 against the 8,738 baseline);
it took a two-pass rollback and a `Send`-graph regression test to make the next call's input
`[system, human]` again. The decisions are in
[ADR-0040](../../adr/0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md).

## The outlets question: a filter that returned 200 instead of 400

The first long persisted thread exposed a defect a year old. Its third turn — *"Across all
Halvorsen Logistics PDUs, how many of the available power outlets are actually in use?"* — spent
**283 s in one model call**, because the tool result before it was **52,721 characters**: a
`device_id__in` filter over the 12 PDU ids had returned the first 200 power outlets of every
tenant. NetBox does not reject `__in`; it drops the filter and answers 200 with everything.
Verified live against every field class — primary key, relational id, string, integer — none
honours it, while the bare key with a list value (`{"device_id": [149, 150]}`) filters correctly.

The agent's validator and skill had been *recommending* `__in` since a trace earlier in the
year, on the strength of the MCP server's own whitelist and tool description, which list it.
Both now reject it; the local copy of the MCP server is corrected and the diff kept for an
upstream report. The 11-turn session harness, re-run against the September baseline with the
same order: **20 tool calls against 39, 1,312 s against 1,552 s, the outlets question down to
one tool call (177 s against 195 s and 9 calls), zero `__in` attempts, same scores.** The
scores did not move because their misses are the site-versus-tenant scoping the harness
documented in September, not filter syntax. [ADR-0041](../../adr/0041-in-lookup-silently-ignored-reject-in-validator.md)
records the decision and the measurement principle it adds: a check that inspects only the
status code cannot see a filter the server ignores.

## What it does not do

- Serve more than one turn at a time: one llama-server slot, one stdio MCP client, one lock.
- <del>Remember conversations across a backend restart</del> — it does since 2026-10-05
  (ADR-0040); conversations created before that date remain browser-only and show the banner.
- Show the model's reasoning: `ChatOpenAI` drops `reasoning_content`, so reasoning is counted
  in tokens written but never displayed.
- Time out a hung stream on its own — the Stop button is the recovery.
- Prove that scores are unaffected by re-running a benchmark through the browser. The claim
  rests on `query()` being unchanged, and every timing on this page is a single run.

**See also**: [Local Frontier Inference](local-frontier-inference.md) ·
[Observability & Monitoring](observability-and-monitoring.md) ·
[ADR-0039](../../adr/0039-web-serving-layer-serialised-turns-server-token-counts.md) ·
[ADR-0040](../../adr/0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md) ·
[ADR-0041](../../adr/0041-in-lookup-silently-ignored-reject-in-validator.md)
