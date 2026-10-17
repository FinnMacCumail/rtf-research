# ADR-0042 — Expert Offloading Is Not the Lever: Pinned Weights, a Newer Build, a Rejected GPU Expert Cache

## Status

**Accepted (October 2026).** Answers the question ADR-0038 left open under *no remedy is
available*: whether the expert-offloading literature and tooling — the `colibri` engine, the
GPU expert cache llama.cpp merged on 2026-10-07, KTransformers-style CPU kernels, learned expert
prefetchers — could reduce the local agent's latency. Measured on the production model and
flags: the cache is slower at every size that fits, the engine solves a problem this host does
not have, and the two changes that paid were not offloading at all — copying the CPU-resident
experts into pinned RAM, and a newer build. The decode ceiling in ADR-0038 stands; its prefill
half had a fix the server had been printing at startup.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`43afc28` serve script; research report in `docs/development/2026-10-17_moe-offloading-research.md`;
A/B and rerun in `docs/development/2026-10-17_moe-cache-ab-and-server-switch.md`; raw
`llama-bench` tables in `docs/traces/2026-10-17_llama-bench_moe-cache-ab.md`)
**Bears on**: [ADR-0038](0038-local-frontier-model-viable-cost-not-capability.md) (partly
superseded: the "no available fix" limitation), [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md)
(the turn-time accounting the comparison uses), [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md)
(the kind of change that *does* move turn time)

## Context

ADR-0038 established that Qwen3.8-Flash-Next (176B, ~6B active, 112 GB of experts in system RAM
via `-ncmoe 48`) answers the v5 benchmark at cloud-equivalent correctness on two 2080 Tis, and
that the cost is 35–486 s per turn. Its limitations section said MoE CPU decode ran at ~19% of
memory bandwidth and that no remedy was available: NUMA mirroring unimplemented, MTP speculation
unmerged, KTransformers needing AMX this Cascade Lake host lacks.

Between that record and this one, upstream llama.cpp merged a GPU LRU cache for experts kept in
host memory (PR #29887 on 2026-10-07, multi-GPU in #30112), with 1.6–2.2× decode on RTX
4090/5090 for this exact model. A separate engine, `JustVugg/colibri`, had reached 42k stars in
three months on the claim of running 744B models from a 25 GB laptop by streaming experts from
NVMe. And the literature on expert offloading — reactive caches, next-layer prediction,
mixed-precision substitution, CPU-GPU co-execution — had grown to ~55 papers. The question was
whether any of it applied to a batch-size-1 tool-calling agent on PCIe 3.0 cards with 11 GB
each.

Two facts about the workload frame the answer. First, the agent's turn is **5.6 sequential
model calls** on average, 97.9% of wall time inside the model, ~85% of prompt tokens served from
the prefix cache; the worst turns are prefill — an 82 s cold prefix, a 167 s prefill of a
20k-token tool result — not decode. Second, every offloading technique in the literature acts on
decode; llama.cpp's cache is explicitly bypassed for batches above 32 tokens.

## Decision

1. **`--moe-cache-mib` is not used on this host.** Measured with `llama-bench` on the production
   flags at 2, 4 and 6 GiB split across both cards, and at 2.5 GiB on a single card with pinned
   host memory and the process on the card's own NUMA node: slower than no cache in every
   configuration (table below). The serve script records it as a measured dead end beside the
   n-gram one.
2. **`colibri` is not adopted.** It streams experts from disk for models that exceed RAM; this
   model is RAM-resident. Its own README reports this model at 1.9–4.0 tok/s on CPU against the
   agent's 13.5, and on a RAM-resident MoE (Qwen3.6-35B-A3B, RTX 3070) it is slower than
   llama.cpp under Ollama (30.0 vs 38.1 tok/s) at worse perplexity for equal bits. GGUF reading
   exists only on its `dev` branch, CPU/Vulkan only, and the Turing-without-ReBAR path is
   unmeasured.
3. **The server moves to llama.cpp b11713 (`2744e48d1`, 2026-10-17) from b11053**, 660 commits
   including the fused shared-expert and grouped top-k kernels: +39% decode and +7% prefill in
   `llama-bench` on identical flags.
4. **`--load-mode none` is added.** The CPU-resident experts are copied into pinned RAM instead of
   being mmap-ed from the GGUF, so the per-microbatch expert uploads during prefill are DMA
   rather than page-cache copies. `llama-bench` prefill 189 → 390 tok/s; on real agent traffic
   97 → 219 tok/s aggregate. The old server had been printing *"tensor overrides to CPU are used
   with mmap enabled - consider using --load-mode none for better performance"* at every start.
5. **The process is started on NUMA node 1** (`taskset -c 10-19,30-39` before `--numa isolate`,
   which otherwise pins to whichever node the process happens to start on). Pinned to node 1,
   decode measured 15.7 tok/s; pinned to node 0, 11.7. Prefill was equal. The page cache was
   dropped once so the 74 GB pinned buffer allocates on the same node as the threads.
6. **KTransformers, ik_llama.cpp, MoE-Infinity, PowerInfer, Fiddler and the learned
   prefetchers are not pursued.** AMX is absent (Xeon Silver 4210 is Cascade Lake); the forks
   either do not know the `qwen4exp` architecture or do not build for compute capability 7.5;
   the best-documented predictor, ProMoE, gained 1.09× over a plain LRU when implemented inside
   llama.cpp.

## Evidence

**`llama-bench`, production flags** (`-ngl 999 -ncmoe 48 -ot per_layer_token_embd=CPU --numa
isolate -t 10 -fa 1 -ub 2048`, three repetitions; `llama-bench` has no cache flag, so a 5-line
env-var hook set `moe_cache_size` and was reverted):

| configuration | pp2048 tok/s | tg128 tok/s | cache hit |
|---|---|---|---|
| b11053 (old build), mmap, no cache | 176.9 | 11.48 | — |
| b11713, mmap, no cache | 189.0 | **15.92** | — |
| b11713, cache 2 GiB split across both GPUs | 187.5 | 4.69 | 47–49% |
| b11713, cache 4 GiB split | 189.9 | 7.87 | 70–72% |
| b11713, cache 6 GiB split | 189.3 | 10.19 | 79–80% |
| b11713, `-lm none`, no cache, process on node 0 | 389.9 | 11.67 | — |
| b11713, `-lm none`, no cache, process on node 1 | **391.4** | 15.71 | — |
| b11713, `-lm none`, cache 2.5 GiB, GPU0 only, process on GPU0's node | 412.7 | 13.31 | 60% |
| b11713, `-lm none`, cache 2.5 GiB, GPU0 only, process on the far node | 413.6 | 13.47 | 60% |

The last two rows settle the mechanism: moving the process across the socket from the card
changed the cached decode by 1%, so cross-socket PCIe is not why the cache loses. The miss path
is slow on its own — roughly 250 expert uploads of ~2.7 MB per token, serialised against
compute; at 2 GiB the cache moved ~290 GB over PCIe for 384 generated tokens. Even an 80% hit
rate at 6 GiB, more VRAM than the 131k-context server can spare, is 36% slower than no cache.

**The production server, after the switch** (`-c 131072`, 4 unified slots): an 8,117-token prompt
prefilled at **314–318 tok/s**; no out-of-memory with the 74 GB pinned buffer plus `-ub 2048`.

**Real agent traffic**, from llama-server's own `print_timing` lines — the old server's full
24-day log against the new server's first day including the rerun below:

| metric | legacy | new |
|---|---|---|
| prefill, requests with ≥256 new tokens, aggregate | 96.9 tok/s (n=601) | **218.5 tok/s** (n=22) |
| prefill, requests with ≥4,000 new tokens, aggregate | 113.3 tok/s | **242.0 tok/s** |
| decode, requests with ≥32 generated, aggregate | 13.64 tok/s (n=1,047) | 14.60 tok/s (n=43) |
| decode, median | 14.28 | 14.62 |

**Eight v5 questions rerun through the agent** (2 simple, 3 medium, 3 advanced; LangSmith
experiment `llamacpp-qwen3.8-flash-next-UD-Q4_K_XL-v5-newbuild-subset-a155acd7`, same
evaluators as the 90-question run; legacy times are the root-run latencies of the 2026-09-23
run for the same examples):

| | legacy | new |
|---|---|---|
| correctness, 8 items | 8/8 | **8/8** |
| sum of agent wall time | 1,285 s | 1,180 s (1.09×) |
| model calls | 35 | 40 |
| prompt tokens read | 411,514 | 596,974 |
| seconds per model call | 35.7 | **28.5** |
| milliseconds per prompt token | 3.0 | **1.9** |
| questions with the same call count both times | — | 1.22×, 1.34×, 1.56× faster |

The wall-clock gain is small because the model planned differently: 12 tool calls instead of 4
on one question, 7 instead of 6 with three times the tool-result text on another, and the new
run's first question paid the cold prefix where in the legacy run it ran seventh (23.9 → 56.6 s
of model time with one tool call each). Per call and per token, the inference is faster;
per turn, planning variance is larger than the inference gain.

## Consequences

### Positive
- **Prefill, the axis the agent lives on, is 2.2× faster on real traffic** for no quality
  change, no new dependency and one flag. The cold 8.7k-token prefix drops from ~82 s to ~27 s;
  a 20k-token tool result from ~167 s to ~65 s.
- **The expert-cache question is closed with numbers rather than an argument**, including the
  mechanism (miss path, not socket locality), so it will not be re-opened each time a new
  cache PR lands with 4090 figures.
- **The launch is deterministic.** `--numa isolate` no longer depends on which socket the shell
  happened to be scheduled on.
- A method principle, beside ADR-0033's and ADR-0041's: **read the startup log.** The remedy
  recorded as unavailable was printed as a warning on every launch for a month.

### Negative / limitations
- **Decode is unchanged in practice** — 13.6 → 14.6 tok/s on real traffic, within the spread of
  the old log. `llama-bench`'s 11.5 → 15.9 does not survive the server's overheads at 131k
  context. ADR-0038's decode ceiling and its cause stand.
- **Turn time is now dominated by call count and tool-result size**, which this ADR does not
  touch. The 1.09× end-to-end figure is the honest one; the per-call 1.25× is what the server
  changes bought.
- **`--load-mode none` costs ~74 GB of pinned RAM and a 60–90 s cold start** instead of a lazy
  mmap. On this 376 GB host that is free; on a smaller one it would not be.
- **The node-1 advantage was measured twice and not explained.** Node 0 carries the desktop and
  the agent's own processes; that is the likely cause, not a verified one. The pinned-buffer
  placement needed a one-off `drop_caches` as root because the GGUF page cache filled node 1.
- **The rerun is eight questions, one run each**, against single legacy runs. It shows direction
  and per-call rate, not a new benchmark figure; the 90-question harness was not re-run.
- `kernel.numa_balancing=0`, flagged by llama-server as harmful, was set and measured the same
  evening: decode 14.45 → 14.72 tok/s median over five 200-token runs, inside the spread of two
  runs taken with it on (14.13–14.86). Once the process is pinned and its memory node-local the
  balancer has nothing to migrate, so it is not a lever here. Left off, not persisted.

## Relationship to ADR-0038

ADR-0038 said the constraint was cost, not capability, and that the cost had no available fix.
Half of that was wrong in a specific way: the decode half still has no fix, but the prefill
half — which is where the outlier turns were — had one in the server's own warning text. The
accounting that found this came from ADR-0039's server-side token and timing numbers: with
prefill and decode split per call, a 1.5× decode gain was visibly worth ~18 s of a 167 s turn,
which is why the expert-cache result was expected to be a wash before it was measured, and why
the measurement was run anyway.

## References

- [Phase 5 → Local Frontier Inference — the expert-offload question, measured](../phases/phase-5-production-deepagents/local-frontier-inference.md#the-expert-offload-question-measured-rather-than-argued)
- [ADR-0038 — A Frontier-Class Model Runs Locally](0038-local-frontier-model-viable-cost-not-capability.md)
- Repository: `scripts/serve_qwen4exp.sh` (`43afc28`), `docs/development/2026-10-17_moe-offloading-research.md`
  (the research report: colibri profile, tool landscape, ~55 papers, with sources),
  `docs/development/2026-10-17_moe-cache-ab-and-server-switch.md`,
  `docs/traces/2026-10-17_llama-bench_moe-cache-ab.md`,
  `docs/traces/2026-10-17_v5-legacy-root-run-latencies.tsv`
- Upstream: llama.cpp PR #29887 (`--moe-cache-mib`), PR #30112 (multi-GPU cache), commit
  `2744e48d1`; `JustVugg/colibri` README and `docs/qwen36-cuda-tier.md`; ProMoE
  (arXiv:2410.22134); "Cacheable by Design?" (arXiv:2608.18261)
