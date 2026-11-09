# ADR-0046 — Strata as the Inference Server: Same Weights, 1.67× the Agent Speed

## Status

**Accepted (November 2026).** The local path of the NetBox agent now runs against
[Strata](https://github.com/Niko1221), a closed, precompiled ggml/llama.cpp-derived server that a
collaborator installed on the same machine, serving the **same** Qwen3.8-Flash-Next UD-Q4_K_XL
files that ADR-0038 measured under llama-server. The agent changed only in how it authenticates
and in what it keeps from the server's response; the model, prompts, tools, skills and
evaluators are untouched. On the full 90-question v5 benchmark the agent's time fell from
14,396 s to 8,603 s at the same model-call count, with correctness 0.922 → 0.939.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`8d090fa`, merged `15d224d`; decision note `docs/development/2026-11-08_strata-inference-server.md`; harness
record `docs/traces/2026-11-08_netbox-benchmark-v5_strata-full.md`)
**Bears on**: [ADR-0038](0038-local-frontier-model-viable-cost-not-capability.md) (its "cost,
not capability" verdict is the thing this changes), [ADR-0042](0042-expert-offloading-not-the-lever-pinned-weights-rejected-gpu-expert-cache.md)
(it rejected llama.cpp's GPU expert cache on this hardware; Strata's profile-placed cache is the
other implementation of that idea, and it wins — the lever was the placement, not the cache), [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md)
(server-reported token counts: one of them is wrong on Strata and is now read from a different
field), [ADR-0030](0030-model-matrix-evaluation-harness.md) (the harness is what made the swap
measurable rather than anecdotal)

## Context

ADR-0038 found the 176B MoE model *viable* on two RTX 2080 Tis but paid for in prefill seconds:
every agent call re-reads a 15k–70k-token prompt, the output is a short tool call, and
llama-server read new prompt tokens at 219–242 tok/s even after the NUMA and `--load-mode none`
tuning that bought 1.34× in October. The work in `rtf` on expert caching (`--moe-cache-mib`) lost
at every size on this hardware, and the research note's verdict was to cut round trips rather
than chase the server.

On 2026-11-05 a collaborator installed Strata 0.2.10 (engine 0.1.57, CUDA 13) on the same box
to serve the same weights to his own agent. Strata is a different implementation of the same
idea the October A/B rejected: an expert cache in VRAM, but **placed from a usage profile** and
pre-filled at start (its log: 1,207 slots on one card, 1,547 on the other, 72.8 % hit rate during
decode), a layer split across the two cards with the prompt path borrowing cache slots so
prefill runs in 6,400-token GPU chunks, int8 KV with the active 32k resident in VRAM and the
rest streamed from RAM, a conversation cache that parks and reuses KV by prefix, and
speculative decoding with a shipped draft model. It exposes an OpenAI-compatible API on
`0.0.0.0:8080` behind a Bearer key that is required even from localhost, serves one request at
a time, and holds ~10.3 GB of each 11 GB card plus ~110 GB RSS — so the project's llama-server
cannot run beside it. The collaborator's headline, 33.6 tok/s, was short-prompt prose.

The question was therefore not "is Strata fast" but "is it fast on *this* traffic, and does the
agent still answer the same questions correctly". Those are exactly the two things the Phase 5
harness exists to settle.

## Decision

1. **Treat the inference server as configuration, and keep it that way.** The agent targets
   Strata through the existing `LLAMACPP_BASE_URL` / `LLAMACPP_API_KEY` settings. No code path
   knows which server it is talking to. Switching back to llama-server is the reverse edit.
2. **Three additions to the llama.cpp backend, none of them server-specific in effect:**
   - the start-up `/v1/models` check and the web layer's `/slots` probe send the API key as a
     Bearer header whenever one is configured (Strata answers 401 without it; llama-server
     ignores it);
   - the cached-token count is taken from llama.cpp's `timings.cache_n` when the OpenAI field
     `prompt_tokens_details.cached_tokens` is 0 — Strata reports 0 there even on a 16,197-of-
     16,202-token prefix hit, which would have made the ADR-0039 ledger's *cached* column lie;
   - the `timings` block is kept on **non-streaming** calls as well as streamed ones, so every
     model call the harness records in LangSmith carries `prompt_ms`, `predicted_ms`,
     `prompt_n`, `predicted_n` and `cache_n`.
3. **Gate the swap on the benchmark, in two steps.** The 8-question subset that gated the
   October server change first (same example ids, same judge), then the full 90 questions,
   compared with October's `…-v5-newbuild-full-2865e34b` run.
4. **Record the server-side compute per call so a shared slot is auditable.** Strata's single
   slot is shared with the collaborator's agent. Wall minus (`prompt_ms` + `predicted_ms`) is
   queue wait; it is now in every run's metadata instead of being inferred from a log without
   timestamps.

## Evidence

**Idle-box probes (2026-11-07), agent-shaped requests, both servers on the same files.**
Decode on tool-call/reasoning output 16.1 tok/s (Strata, drafts ~50 % accepted) against
13.5–14.7 (llama-server); at 16k context 14.5 vs ~14.5. Prefill at 16k: **937 vs 219–315
tok/s**. OpenAI tool calls, `include_usage`, `enable_thinking=false` and the project's model
alias all accepted. The 33.6 tok/s figure did not reproduce on agent output: the draft model is
accepted far more often on prose.

**8-question subset (2026-11-07)**, `…-v5-strata-subset-2a33b51a` vs October's
`…-v5-newbuild-subset-a155acd7`: 8/8 correct both; agent time **1,180 → 683 s (1.73×)**; model
calls 40 → 36; 29.5 → 19.0 s per call; 1.98 → 1.50 ms per prompt token. On the three questions
with identical call counts: 1.13×, 1.45×, 3.44× (the last had paid the cold prefix in October).
The one slower question (0.84×) was the model choosing seven calls instead of four.

**Full v5 (2026-11-08)**, `…-v5-strata-full-00b39a92`, same dataset, evaluators and
`gpt-oss:20b` judge as October:

| | October, llama-server | Strata |
|---|---|---|
| agent time, sum of 90 root runs | 14,396 s | **8,603 s (1.67×)** |
| mean / median / max per question | 160.0 / 119.9 / 761 s | **95.6 / 63.1 / 688 s** |
| simple / medium / advanced tier mean | 96.7 / 167.6 / 215.6 s | 51.9 / 91.4 / 143.5 s (1.86× / 1.83× / 1.50×) |
| questions faster than October | — | 74 of 90 |
| correctness | 0.922 | **0.939** (8 up, 6 down) |
| completeness · entity coverage (n=71) | 0.944 · 0.859 | 0.972 · 0.908 |
| tool calls per question · tool errors | 4.878 · 0 | 4.444 · 0 |
| model calls · seconds inside the model | 411 · 13,938 s | 412 · 7,910 s |
| seconds per model call | 33.9 | **19.2** |
| ms per prompt token (incl. cached) | 2.12 | **1.41** |
| wall, first start to last end | 5.30 h | 5.91 h |
| root-run errors | 0 | 0 |

**From the new per-call `timings` (412 of 412 calls):** prompt 1,929 s for 555,474 new tokens
(288 tok/s); generation 5,959 s for 121,635 tokens (20.4 tok/s); compute 7,889 s of the 7,910 s
spent inside model calls, so **queue wait was 21 s across the run** — the shared slot was not
touched. Strata's engine log gained 413 request lines against the 412 recorded calls.

**Why the wall clock got worse while the agent got faster.** Strata leaves under 1 GB free on
each GPU, so Ollama placed the 14 GB `gpt-oss:20b` judge entirely on CPU, where it competed
with Strata's own CPU expert path (load average 36 on 40 threads; judging ~130 s per question
against ~52 s in October). Agent time excludes judging. The 288 tok/s prefill under that load,
against 937 idle, is the size of the interference.

**The thing the speed-up does not touch.** Model-call counts are identical (412 vs 411) and
completion tokens are identical (121.6k vs 122.6k): the model planned the same work and each
call cost 43 % less. The slowest questions are still the ones where the model chose ten tool
calls instead of four, and a 14k-token GraphQL result still has to be read once — ~60 s on
llama-server, ~49 s on Strata under judge load, ~15 s on Strata idle — before the prefix cache
makes it free on later calls. Field selection and page limits are worth exactly what they were.

## Consequences

### Positive

- **ADR-0038's verdict moves.** "Viable, cost not capability" was priced at 160 s mean per
  question and 33.9 s per model call; it is now 96 s and 19.2 s on the same weights, same
  GPUs, same answers. The web chat's two-to-three-minute turns become about one minute.
- **The gate did its job twice.** The subset predicted 1.73×; the full run delivered 1.67×, and
  the correctness check caught no regression. Nothing about the swap rested on the headline
  tok/s figure, which turned out not to describe this workload.
- **Every benchmark call now carries server-side compute time.** Queue wait, prefill and decode
  rates per call come from the run metadata, which also retires the log scraping that produced
  October's per-call figures.
- **The agent did not change.** The three additions are inert on llama-server, so the two
  servers remain interchangeable by configuration.

### Negative / limitations

- **Closed binary.** None of the October tuning (`--numa isolate`, worth +56 % on llama-server)
  can be applied; Strata's own log lists NUMA placement per node as a known limitation. Its
  internals are known only from its log lines. If it breaks, the fallback is llama-server, and
  the two cannot run at once.
- **One slot, shared.** The collaborator's agent is pinned to the same instance. The run was
  clean (21 s wait) but that was circumstance, not design; the per-call timings make contention
  visible after the fact, not avoidable.
- **Not an equal-load comparison.** October's judge had GPU room; November's ran on CPU against
  the server under test. Strata's advantage under equal conditions is therefore *at least*
  1.67×, not exactly it.
- **Scores are within judge noise.** 8 up / 6 down on an LLM judge is "no regression", not an
  improvement, and is recorded as such.
- **The cloud-matrix re-run owed since ADR-0041 is still owed.** Nothing here discharges it.

## Relationship to ADR-0038, ADR-0039 and ADR-0042

ADR-0038 established that the frontier-class local model answers the benchmark and that the
price is paid in prefill. This ADR keeps the model and the price's *unit* and changes its
*rate*, without reopening the capability question — that is why the gate was the same benchmark
rather than a new one. ADR-0039's principle that token numbers come from the server or not at
all survives, with one correction: on Strata the OpenAI `cached_tokens` field is the number that
is wrong, and the llama.cpp `timings.cache_n` is the one that is right, so the ledger reads the
latter when the former is 0. ADR-0042 measured llama.cpp's `--moe-cache-mib` losing at every size that
fit and concluded expert offloading was not the lever; Strata does not contradict the measurement, it
changes the implementation — a cache pre-filled from a usage profile, with the prompt path borrowing
its slots — and the result says the idea was right and the October form of it was not. ADR-0042's
verdict on llama.cpp's own cache stands.

## References

- Decision note with the probe measurements, both tables and the configuration:
  `ollamaDeepAgents/docs/development/2026-11-08_strata-inference-server.md`
- Harness record: `ollamaDeepAgents/docs/traces/2026-11-08_netbox-benchmark-v5_strata-full.md`
- LangSmith experiments: `llamacpp-qwen3.8-flash-next-UD-Q4_K_XL-v5-strata-full-00b39a92`,
  `…-v5-strata-subset-2a33b51a`; October baselines `…-v5-newbuild-full-2865e34b`,
  `…-v5-newbuild-subset-a155acd7`
- October server change and the expert-cache A/B ([ADR-0042](0042-expert-offloading-not-the-lever-pinned-weights-rejected-gpu-expert-cache.md)) this supersedes in outcome:
  `ollamaDeepAgents/docs/development/2026-10-17_moe-cache-ab-and-server-switch.md`
- Phase 5 narrative: [Local Frontier Inference](../phases/phase-5-production-deepagents/local-frontier-inference.md)
