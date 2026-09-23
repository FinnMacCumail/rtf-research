# ADR-0038 — A Frontier-Class Model Runs Locally: the Constraint Is Cost, Not Capability

## Status

**Accepted, then CORRECTED (September 2026).** The original evidence was an **n=1-per-tier existence
proof**, not a benchmark score. It is recorded as an ADR because it bears directly on a decision the
whole phase has been building toward — model-handoff routing — and because it **partly reverses the
premise** that local models are not capable enough for this agent. A subsequent 12-question run
**overturned the context limitation** recorded below; the original reasoning is preserved and the
withdrawn claim struck in place, with the correction immediately following.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Bears on**: [ADR-0037](0037-stratified-benchmark-v5-difficulty-not-capability.md) (v5 benchmark),
[ADR-0028](0028-native-local-and-cloud-models-on-deepagents.md) (local & cloud models),
[ADR-0035](0035-deepagents-0.7.5-upgrade.md) (upgrade taken partly to unblock routing)

## Correction — the context requirement was understated (September 2026)

A **12-question stratified run** (4 per tier, 6/6 across both data islands, scored with the full
evaluator set) replaced the n=1-per-tier existence proof below. It overturned the `≥32k` limitation
recorded in *Negative / limitations*:

| | baseline, `-c 32768` | **`-c 131072 --no-kv-offload`** |
|---|---|---|
| correct | 8 / 12 | **11 / 12** |
| wrong | 1 | 1 |
| **lost to context** | **3** | **0** |
| overflows / truncations | 2 / 1 | **0 / 0** |
| peak context reached | ceiling 32,768 | 44,295 |
| tool calls | 60 | 88 |
| wall time | 82 min | 155 min (**1.9×**) |

**32k was not enough — it was the binding constraint.** All three questions lost to context at 32k
recovered *and scored correct* at 128k: point-to-point circuits (15 calls), branch-site firewalls
(13 calls), changelog deletions (6 calls). A fourth flipped wrong → correct.

<del>**The fix uses the resource this box has in surplus.** `--no-kv-offload` moves the KV cache into
system RAM — **376 GB, against 21 GB of VRAM** — buying 4× context for ~37% of decode speed
(11.2 → 7.0 tok/s).</del> It loaded in **20 seconds**.

**WITHDRAWN — the trade-off was a false choice (September 2026).** `--no-kv-offload` has been
removed. The 128k window did **not** require it: this is a hybrid model, and only **12 of 48 layers
are full attention** (indices 3, 7, 11 … 47). The other 36 are Gated DeltaNet, whose recurrent state
is sized by *sequences*, not tokens, so it does not grow with context. Those 12 layers use just
**2 KV heads** (GQA) at 256 key/value length:

```
12 layers × 2 heads × (256+256) × 2 B = 24 KiB per token
-c 131072                             = 3.00 GiB attention KV  (+ ~0.46 GiB recurrent)
```

Confirmed by VRAM subtraction — resident went **14.7 → 17.4 GiB**, ~3.1 GiB for both caches, leaving
~3.3 GiB headroom on 21 GiB, no OOM. Geometry read from the GGUF; `kv_unified=true` means the four
slots share one pool rather than each claiming `-c` (`llama-context.cpp:290`).

Removing the flag is a **~2× win on every axis**, measured on three NetBox questions in one
accumulating thread:

| | KV in system RAM | KV in VRAM | |
|---|---|---|---|
| prefill (matched ~8.7k prompt) | 89.86 tok/s | **126.42 tok/s** | 1.41× |
| decode | 5.1–7.5 tok/s | **~10.8 tok/s** | ~2× |
| Q2 wall clock | 171.2 s | **91.6 s** | 1.87× |
| Q3 wall clock | 665.3 s | **329.4 s** | 2.02× |

Correctness unchanged; zero truncations. A 1.24× *regression* on the first question after restart is
a cold-cache artefact — that turn paid a full 8,743-token prefill in one 81.8 s call — and is
excluded; only warm turns compare.

**Replicated independently**, on a second run of the same three questions launched from the
committed `serve_qwen4exp.sh` rather than a hand-typed command line — so what was validated is the
artefact people actually use. Again 3/3 correct, one accumulating thread, peak context 9.5k / 10.9k
/ 19.1k:

| | KV in system RAM | first VRAM run | replication |
|---|---|---|---|
| Q2 (warm, 2 LLM calls both runs) | 171.2 s | 91.6 s | **76.9 s — 2.23×** |
| Q3 | 665.3 s | 329.4 s | 413.9 s — 1.61× |
| Q1 (cold first turn) | 101.6 s | 126.1 s | 155.9 s — *excluded* |

**Q2 is the load-bearing comparison** — warm cache, identical call count — and the replication beat
the first VRAM run.

**The cold-turn artefact is now demonstrated rather than argued.** Within Q1, per-call timings were
**108.6 s** (8,743 prompt tokens), then **20.4 s** (8,958) and **23.7 s** (9,534). *More* tokens at
one-fifth the time once the prefix cache fills; 152.7 s of the 155.9 s wall was server-side. Any
benchmark whose first question follows a restart is measuring cache warm-up, not configuration.

**Per-question call-count variance is large, and worth more caution than the timings.** The same Q3,
same configuration, took **9 LLM calls in one run and 14 in the other**. Tool execution accounted for
just **5.4 s of 413.9 s** — the cost is LLM round-trips, not tools. Both runs made **four
`netbox_graphql_schema` introspections**, re-learning the schema mid-query: independent evidence for
the GraphQL routing-tightening work, and a reason not to read much into any single advanced-tier
timing.

**The flag was correct when set.** The failure it addressed was `-c 32768` losing 3 of 12 questions.
Raising the window fixed that, and nobody re-checked whether the offload was still needed. It was
not, and it cost roughly half the throughput for two months of measurements.

**A process error worth recording.** Before testing the window, a per-tool-result size cap was built
into the MCP wrapper on the theory that oversized payloads were the problem. It cost two ~87-minute
runs and was reverted:

1. **Inert.** `langchain_mcp_adapters` declares `response_format="content_and_artifact"`, so tool
   coroutines return a `(content, artifact)` **tuple**. The guard measured only `str`/`list` and
   skipped every MCP result — 3 oversized results, 0 cap firings. The unit test fed it a bare
   string, so it validated an assumption about the adapter rather than the adapter's contract.
   *A test written from the same misunderstanding as the code cannot catch that misunderstanding.*
2. **Mis-sized.** At 60,000 chars (~15,000 tokens) a single permitted result consumed ~63% of the
   23,768-token working budget: still allowed a truncation, while turning a question the baseline
   answered in **3 calls / 141 s** into a **65-minute retry loop**. A per-result cap must be sized
   as *context budget ÷ expected call count*, and even then it treats the symptom.

The correct move was to test the cheap hypothesis first: raising the window took 20 seconds to
verify and made the cap unnecessary.

## Context

[ADR-0037](0037-stratified-benchmark-v5-difficulty-not-capability.md) names **model-handoff
routing** as the next step: route simple queries to a cheap model, hard ones to a strong one. Every
earlier local-model attempt had failed the premise from the wrong end — the local candidates tried
(a 32B, a 7–8B, and others) failed on the **simple** tier, so there was nothing to hand off *from*.
The working assumption was that local models lacked the capability.

**Qwen3.8-Flash-Next** (`config.json` model type `qwen4_exp`, an open-weight preview of the Qwen 4
architecture) changes the premise. It is a **125B MoE with ~6B active per token** (512 experts, 10
routed + 1 shared) plus a **51B n-gram embedding lookup table** — ~176B total. The lookup table is a
table, not a network: it costs memory, not computation. That shape is what makes a frontier-class
model tractable on commodity hardware.

Published third-party reports warned of **tool-calling fragility** in long multi-turn loops with
thinking enabled — repetition loops, malformed `<tool_call>` emission, and spurious mid-task
termination (one production report: ~15 premature stops across 2,176 tool turns, ≈0.7%). That is
precisely this agent's workload, and it was the stated reason to expect failure.

## Decision

**Run the real agent against the real model on the real hardware — three questions, one per
difficulty tier — and treat the result as an existence proof rather than a measurement.**

Three questions cannot produce a score. ADR-0037 established that even **90 paired items resolve
only ~9pp**; three resolve nothing. The question asked here is narrower and answerable at n=1:
*can this model, on this hardware, complete a multi-turn MCP tool loop and get the right answer?*

Hardware: 2× RTX 2080 Ti (21.1 GiB VRAM, compute 7.5), 2× Xeon Silver 4210 (20 physical cores,
2 NUMA nodes), 376 GB DDR4. Quant: unsloth `UD-Q4_K_XL`, 105 GiB, served by `llama-server` through
the existing `LLM_BACKEND=llamacpp` path — **no application code changed**.

## Evidence

Three v5 questions, one per tier, scored by the harness's own `expected_entities` coverage:

| tier | question | time | tool calls | budget | coverage |
|---|---|---|---|---|---|
| simple | device count at a site (all statuses) | 241.9 s | **2** | 3 | **1.0** |
| medium | which rack holds the most devices | 198.1 s | **2** | 7 | **1.0** |
| advanced | which hypervisor has exactly one PSU cabled, and to which outlet | 998.7 s | **20** | 12 ✗ | **1.0** |

**3/3 correct.** The advanced answer named `sea-dc1-esx05` / `sea-dc1-pdu03` Outlet 1 and correctly
classified every decoy the reference lists (`esx01`/`esx02` as 2-of-2; `esx03`/`esx09` as 0-of-2).
It went beyond the reference: it quoted the cable label *"single supply - D10"* as evidence the
single feed is intentional rather than a data gap, and flagged the six unconnected hosts as a
records-audit item.

Throughput measured on this hardware: **decode 11.2 tok/s**, **prefill 64.4 tok/s** at 1.5k tokens,
**~48 tok/s** on realistic ~9k agent prompts.

**The tuning finding that generalises**: llama.cpp defaults `--numa` to *disabled*, so threads
straddle both sockets. `--numa isolate -t 10` was worth **+47%** CPU-only and **+56%** hybrid decode
on an MoE analogue (13.71 → 21.38 tok/s), with sub-1% run-to-run deviation. See
[Local Frontier Inference](../phases/phase-5-production-deepagents/local-frontier-inference.md).

## Consequences

### Positive
- **The capability premise is reversed.** Every previous local candidate failed the *simple* tier;
  this one went 3/3 including a two-hop power trace. Local handling of a NetBox agent is no longer
  ruled out by capability.
- **Cost asymmetry is the real signal, and it is sharp.** Simple and medium cost **2 tool calls
  each**; advanced cost **20** — 167% over budget, 16.6 minutes. That is a 10× jump precisely at the
  tier where help is most wanted, and it is exactly the shape a handoff policy would exploit: keep
  single-anchor lookups local, escalate multi-hop traces.
- **No application changes were required.** `llamacpp_config.py` and the `LLM_BACKEND` branch already
  supported this path; `run_matrix.py` already takes `backend:model` tuples.
- **A reusable hardware lever** (`--numa isolate -t 10`) that applies to every local model on this
  box, not just this one.
- **A second, larger lever: `-b 2048 -ub 2048` is worth 3.8× on prefill** (34.0 → 129.7 tok/s on a
  9,093-token prompt; 2.8× wall clock on the same request). llama.cpp only copies CPU-resident
  expert weights to the GPU once a microbatch is big enough to amortise the PCIe transfer, and the
  default `-ub 512` never reaches that threshold. This matters disproportionately because prefill is
  where an agent lives — **187,272 prompt tokens against 21,825 generated** across 12 questions.

### Negative / limitations
- <del>**n=1 per tier. This is not a score.** One advanced question that happened to need 20 calls may not
  be representative; the same caution that produced the ADR-0037 correction applies with more force
  here, not less.</del>
  **SUPERSEDED — the full 90-question set has now been run locally.** corr **0.906** (simple 0.917 /
  medium 0.933 / advanced 0.867), 0 errors. Against the **re-scored** cloud figures on the same
  items — flash 0.911, kimi 0.928, pro 0.944, qwen3.5:397b 0.817 — the local model is **0.5pp below
  deepseek-v4-flash**. ADR-0037 records that this set resolves gaps of ~9pp and above and cannot
  resolve the ~3pp between closely-matched models, so the honest claim is **indistinguishable from
  flash on this benchmark**, not "matches" and certainly not "beats". It does sit **8.9pp above
  qwen3.5:397b-cloud**, which is a gap the set *can* resolve.
  Note the comparison must use the re-scored cloud numbers: three reference-wording fixes raised
  flash 0.906 → 0.911, kimi 0.911 → 0.928, pro 0.933 → 0.944. The local run came after those fixes,
  so comparing it against the pre-fix set would credit it with corrections it did not earn.
- **Three of seven failures are one retrieval defect, not seven reasoning failures.** On three
  questions the agent searched for "Halvorsen Logistics" as a circuits *provider*, found nothing,
  and answered **"Halvorsen Logistics was not found in NetBox"** — confidently, after 2–5 real tool
  calls, about the seeded tenant that 57 of the 90 questions depend on. The other 87 questions found
  it. This is a scoping failure that produces an authoritative denial rather than an error, which is
  the same dangerous shape as the silent truncation this ADR records elsewhere. Counting it as three
  independent capability misses would overstate the model's weakness and hide a fixable routing bug.
- **The reported tool-call fragility did not reproduce — and this test could not have detected it.**
  Across 17 logged turns there were no repetition loops, no malformed `<tool_call>`, no premature
  stops, no truncation. But at the reported ~0.7%-per-turn rate, **24 tool calls yields ~0.17
  expected failures**. Observing zero is consistent with the problem being real. The claim that
  fragility *disqualifies* this model was stated too strongly and is withdrawn; it is not refuted.
- <del>**≥32k context is required.** The first advanced attempt died at `18181 tokens exceeds the
  available context size (16384)` — a configuration limit chosen because quantized KV could not be
  confirmed safe on this hybrid GDN architecture. At 32k it ran to 28,106 tokens and succeeded. The
  failure was the harness operator's, not the model's.</del>
  **WITHDRAWN — understated.** A 12-question run lost **3 of 12** questions to context *at 32k*
  (2 hard overflows, 1 silent truncation), with peak context reaching 44,295 tokens once the
  ceiling was lifted. The measured requirement is **≥128k**. <del>obtained via `--no-kv-offload`</del> —
  the window needs no offload at all: 131k of KV costs only **3.00 GiB** on this hybrid
  architecture and fits in VRAM. See the Correction above.
- **Throughput: the full harness has now been RUN, not extrapolated — 6.78 h.** Ninety questions,
  one model, serial: **6 h 47 m at 271.6 s/question**, corr **0.906**, comp 0.944, entity_cov 0.866,
  5.52 tool calls, **0 errors**, zero truncations, zero overflows, peak context 75,571 of 131,072.
  Per tier: simple 0.917 / medium 0.933 / advanced 0.867. Decode held at a 13.47 t/s median across
  507 requests, falling only to 10.25 at peak context. That supersedes every extrapolation below —
  but **attribute it to the configuration, not to one flag**: this run differs from the ~15.8 h
  figure in four ways (KV in VRAM, `reasoning_effort="low"`, `max_tokens` 8192, model `profile`
  set). The clean per-flag number for removing `--no-kv-offload` remains the 1.87×/2.02× measured
  in the three-question A/B, where nothing else changed.
  **The projection was wrong again, and the reason generalises.** A 12-question sample predicted
  3.2–4.5 h; the real figure is 6.78 h, optimistic by **1.5–2.1×**. The sample peaked at 22,532
  tokens of context where the full set reached **75,571** — stratified sampling caught the tiers but
  not the expensive tail. *A subset that is representative of difficulty is not thereby
  representative of cost.*
- <del>**Throughput makes the full v5 harness impractical**, and the working configuration makes it more
  so. At 32k the 12-question run averaged 412 s/question (**~10.3 h** extrapolated to 90); at 128k
  it averaged 775 s/question (**~19.4 h**, or ~2.4 days at the 3× replication standard). Adding
  `-ub 2048` brings this to **~15.8 h — now measured, not modelled.**</del> (Superseded by the measured
  6.78 h above; retained because the 1.23× `-ub 2048` comparison below is still valid.) A second 12-question run with
  the flag took **126.3 min against the baseline's 155.1 (1.23×)** at identical correctness (11/12,
  zero context failures) and 77 tool calls against 88. The earlier projection of ~105 min / ~12.6 h
  was **optimistic by ~20%**: it credited the flag's isolated 3.8× prefill gain against *all* prompt
  tokens, when ~85% of them were already served from the prefix cache. Only the genuinely-new
  remainder and each question's cold first turn are accelerated — and decode, which is untouched,
  now accounts for roughly two thirds of the remaining wall time.
- **Speculative decoding is unavailable in every form.** `--spec-type ngram-simple` needs no draft
  model and looked well matched to an agent that echoes tool-result strings, but measured **163
  drafts and zero accepted**, costing 24% of decode; combined with `-ub 2048` it degraded both axes.
  A 0% acceptance rate is mechanical, not a tuning failure — it needs an exactly-repeating 12-token
  run and a table of distinct device names has none. Draft-model speculation is separately
  impossible: `common/speculative.cpp` throws on vocab mismatch and no qwen4exp-vocab draft model
  exists; the MTP head PRs (#27836, #28243) remain unmerged.
- **Prompt caching is not a remaining lever — it already works.** Median LCP similarity **0.956**
  across 75 slot selections, 75 of 76 requests warm, and only 2,432 of a mean 16,405-token context
  actually prefilled: **~85% served from cache**. The residual prefill is genuinely new tool-result
  text.
- **MoE CPU decode achieves only ~19% of memory bandwidth (19–20 GB/s against a 101.5 GB/s STREAM
  triad) and no remedy is available.** NUMA weight mirroring is the matched fix and
  `GGML_NUMA_STRATEGY_MIRROR` exists in ggml's enum *and nowhere else in the source*. Alternative
  runtimes do not help: KTransformers requires Ampere+ and AMX this host lacks; vLLM and SGLang
  cannot host 112 GB on 21 GiB of VRAM.
- **Ollama cannot serve this architecture** on this host: the installed version (0.13.3, Dec 2025)
  predates `qwen4exp` by nine months, and local GGUF import for the architecture is reported broken.
  `llama-server` is the only working path.
- **No Turing-specific datapoint exists in the wild** for this model; bf16 tensor-core paths are
  unavailable below Ampere, though fp16 paths are not.

## Relationship to ADR-0037

ADR-0037 concluded that **efficiency, not correctness, is the usable routing signal**. This ADR is
the first local evidence consistent with that conclusion and sharpens it: correctness did not
separate the tiers here (all 1.0), while **cost separated them by 10×**. A handoff policy keyed on
anchor count or hop count — not on topic — now has direct supporting evidence.

## References

- [Phase 5 → Local Frontier Inference](../phases/phase-5-production-deepagents/local-frontier-inference.md)
- [ADR-0037 — Stratified v5 Benchmark](0037-stratified-benchmark-v5-difficulty-not-capability.md)
- [ADR-0028 — Native Local & Cloud Models on DeepAgents](0028-native-local-and-cloud-models-on-deepagents.md)
- [Research Methods → Benchmarking](../methods/benchmarking.md)
- Upstream: `ggml-org/llama.cpp` PR #27742 (`qwen4exp` architecture, merged 2026-08-27)
- Repository: `src/agents/llamacpp_config.py`, `tests/eval/dataset_v5.py`, `tests/eval/run_matrix.py`
