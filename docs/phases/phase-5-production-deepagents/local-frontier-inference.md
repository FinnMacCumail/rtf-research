# Local Frontier Inference — a 176B Model on Two 2080 Tis

Every local model tried against this NetBox agent had failed on the **simple** tier. That made
model-handoff routing moot: there was nothing to hand off *from*. This page covers the run that
reversed that premise, what it cost, and the one tuning flag that turned out to matter more than a
five-month llama.cpp upgrade.

## Why this model has a shape that fits

**Qwen3.8-Flash-Next** — internally `qwen4_exp`, an open-weight preview of the Qwen 4 architecture —
splits into two very different halves:

| part | size | what it does |
|---|---|---|
| MoE network | 125B total, **~6B active/token** | 512 experts, 10 routed + 1 shared per token |
| N-gram embedding table | **51B** | a *lookup table*, not a network |

The lookup table is the trick. It stores what common token trigrams usually mean, so the model reads
the answer instead of recomputing it layer by layer. A lookup costs no multiplication, and the rows
needed are known before the first layer runs — so the table can live in system RAM (or on SSD) while
only the *thinking* half needs to be near a GPU. That is why a 176B-parameter model has a memory
profile a commodity box can satisfy.

The idea is not original to Qwen and they say so: DeepSeek published **Engram** (arXiv 2601.07372,
Jan 2026) and Google's Gemma 3n shipped per-layer embeddings — llama.cpp's implementation still
borrows Gemma's tensor naming. What Qwen did was ship it at frontier scale in open weights.

## The hardware, and the constraint that actually binds

2× RTX 2080 Ti (**21.1 GiB** VRAM total, compute 7.5, PCIe 3.0, no NVLink), 2× Xeon Silver 4210
(**20 physical cores**, 2 NUMA nodes, DDR4), **376 GB RAM**, NVMe.

The published community reports for this model are dominated by people fighting RAM limits. That
constraint simply does not apply here: at 105 GiB the `UD-Q4_K_XL` quant sits entirely in memory with
room for the **188 GB** Q8_0 if wanted. The binding constraint is **VRAM**: 21 GiB against a 105 GiB
model means the experts live in RAM permanently, and the cost of that is measurable.

```
llama-server -hf unsloth/Qwen3.8-Flash-Next-GGUF:UD-Q4_K_XL \
  --numa isolate -t 10 \
  -ngl 999 -ncmoe 48 \
  -ot 'per_layer_token_embd=CPU' \
  -c 32768 -fa auto --jinja \
  --host 127.0.0.1 --port 58123
```

`-ngl 999` sends everything to the GPUs, then `-ncmoe 48` claws the expert tensors back to RAM and
`-ot per_layer_token_embd=CPU` keeps the ~25–30 GB n-gram table off the cards. Dense layers,
attention and the output head stay resident in VRAM.

## The tuning flag worth more than the upgrade

The llama.cpp checkout was **2362 commits stale**, so it was rebuilt against current upstream to see
whether the measured memory-bandwidth ceiling was an artefact of an old binary. It was not — decode
measured **flat to slightly worse** on the new build. The rebuild's value was answering the question,
not raising the number.

What *did* raise the number was a default nobody had touched. **llama.cpp sets `--numa` to
`disabled`**, so worker threads straddle both sockets and stream expert weights across the
interconnect. On a dual-socket box that is expensive:

| config (MoE analogue, `-r 5`) | decode tok/s |
|---|---|
| CPU-only, default | 9.26 |
| CPU-only, `--numa isolate` | **13.57  (+47%)** |
| experts-on-CPU, default | 13.71 |
| experts-on-CPU, **`--numa isolate -t 10`** | **21.38  (+56%)** |

Thread count matters just as much and in the opposite direction to intuition: under `isolate`,
`-t 5` → 13.40, **`-t 10` → 21.38**, `-t 20` → 17.88; without it, `-t 40` collapses to **5.89**. Ten
threads is exactly the physical core count of one NUMA node. Hyperthreads hurt.

Two corollaries fell out. Apparent *per-GPU* speed differences were a NUMA artefact — an 18.00 vs
12.28 gap between two identical cards shrank to 17.91 vs 16.44 once threads were pinned. And
`-sm tensor` (tensor parallelism) measured **slower** than the default `-sm layer`, because NCCL is
absent and its only packaged build targets a different CUDA major version.

Effective memory bandwidth rose from ~19 GB/s to **~28 GB/s** — still only ~28% of the 101.5 GB/s
STREAM triad this machine achieves, which is where the remaining headroom lives.

## The test: three questions, one per tier

Not a benchmark. [ADR-0037](../../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md)
established that **90 paired items resolve only ~9pp**; three questions resolve nothing. The question
here is narrower: *can it complete a real multi-turn MCP tool loop and get the right answer?*

Run through the agent's existing `LLM_BACKEND=llamacpp` path with **no application code changed**.

| tier | time | tool calls | budget | coverage |
|---|---|---|---|---|
| simple | 241.9 s | **2** | 3 | **1.0** |
| medium | 198.1 s | **2** | 7 | **1.0** |
| advanced | 998.7 s | **20** | 12 ✗ | **1.0** |

**3/3 correct.** The medium answer produced the full per-rack breakdown (R01=11, R02=10, R03=8)
matching the reference exactly, and spotted that one host is unracked — which is why the site total
is 30 while racked devices number 29.

The advanced question is the interesting one. It asks which hypervisor has *exactly one* of two power
supplies cabled, and to which outlet — a trace from power port through cable to PDU outlet. The model
answered `sea-dc1-esx05`, PSU0 → `sea-dc1-pdu03` Outlet 1, and classified every decoy the reference
names correctly (`esx01`/`esx02` cabled 2-of-2; `esx03`/`esx09` 0-of-2). It then went further than
asked: it quoted the cable's own label — *"single supply - D10"* — as evidence the single feed is
deliberate rather than a data gap, and flagged the six unconnected hosts as a records-audit item.

## What it cost, which is the actual finding

Correctness did not separate the tiers — all three scored 1.0. **Cost separated them by 10×.**

Simple and medium each needed **2 tool calls**. Advanced needed **20**, against a budget of 12, over
16.6 minutes. That asymmetry is the routing signal: single-anchor lookups are cheap locally,
multi-hop traces are not. It is direct local evidence for ADR-0037's conclusion that *efficiency, not
correctness, is what discriminates* once models are competent.

Measured throughput: decode **11.2 tok/s**, prefill **64.4 tok/s** at 1.5k tokens, **~48 tok/s** on
realistic ~9k agent prompts. Prefix caching matters enormously in a tool loop — the first turn of a
question costs 3–4 minutes cold, subsequent turns ~23–33 s, because the system prompt and tool
schemas are reused.

Extrapolated to the full v5 set (30/30/30) from these three questions: ~12 hours per run. ***That
estimate is superseded.*** Measured later across 12 questions: 412 s/question at 32k (~10.3 h, but
that configuration loses 1 question in 4 to context) and **775 s/question at `-c 131072
--no-kv-offload` — ~19.4 h per run**, ~2.4 days at the 3× replication standard. See the next section.

## The context window was the binding constraint

The three-question proof above ran at `-c 32768`. Scaling to **12 questions** (4 per tier, 6/6 across
both data islands, full evaluator set) showed that window was not merely a floor to clear — it was
the thing limiting the result:

| | `-c 32768` | **`-c 131072 --no-kv-offload`** |
|---|---|---|
| correct | 8 / 12 | **11 / 12** |
| wrong | 1 | 1 |
| **lost to context** | **3** | **0** |
| overflows / truncations | 2 / 1 | **0 / 0** |
| peak context | ceiling 32,768 | 44,295 |
| tool calls | 60 | 88 |
| wall time | 82 min | 155 min (**1.9×**) |

All three context-lost questions recovered **and scored correct**: point-to-point circuits (15 calls,
1564 s), branch-site firewalls (13 calls, 1862 s), changelog deletions (6 calls, 819 s). A fourth
flipped wrong → correct. The single regression — a changelog count that went correct → wrong using
*fewer* calls (4 → 2), finding 3 records where the baseline found 4 — is the model under-searching at
n=1, not a context effect.

~~**The fix uses the resource this box has in surplus.** `--no-kv-offload` puts the KV cache in system
RAM — **376 GB against 21 GB of VRAM** — buying 4× context for ~37% of decode speed (11.2 → 7.0
tok/s).~~ It loaded in **20 seconds**.

**WITHDRAWN — that trade-off was a false choice, and it cost ~half the throughput for nothing.**
The 128k window never needed the offload. Only **12 of 48 layers are full attention** (indices
3, 7, 11 … 47); the other 36 are Gated DeltaNet, whose recurrent state is sized by *sequences* rather
than tokens and so does not grow with context. Those 12 layers use **2 KV heads** at 256 key/value
length — **24 KiB per token**, so `-c 131072` is **3.00 GiB** of attention KV plus ~0.46 GiB of
recurrent state. Measured by VRAM subtraction: resident 14.7 → 17.4 GiB, ~3.1 GiB for both caches,
~3.3 GiB headroom left, no OOM.

| | KV in system RAM | KV in VRAM | |
|---|---|---|---|
| prefill (matched ~8.7k prompt) | 89.86 tok/s | **126.42 tok/s** | 1.41× |
| decode | 5.1–7.5 tok/s | **~10.8 tok/s** | ~2× |
| Q2 wall clock | 171.2 s | **91.6 s** | 1.87× |
| Q3 wall clock | 665.3 s | **329.4 s** | 2.02× |

Correctness held — the advanced question still named `sea-dc1-esx05` PSU0 → `sea-dc1-pdu03` Outlet 1
and still declined to invent a PDU→feed→panel hop the data does not contain. Zero truncations. The
first question after restart shows a 1.24× regression and is excluded: it paid a full 8,743-token
prefill on a cold cache, 81.8 s in a single call, against a baseline whose server was already warm.

**The flag was correct when it was set** — the failure it fixed was `-c 32768` losing 3 of 12
questions, one of them silently. Raising the window solved that, and nobody went back to ask whether
the offload was still earning its cost. *A workaround outlives the problem it was built for unless
something forces the re-check.*

> **Every runtime figure below this point — 775 s/question, ~19.4 h, ~15.8 h, and the 82 → 155 min
> comparison — was measured with KV offloaded.** They are not withdrawn, but they are lower bounds on
> speed. The 12-question benchmark has **not** been re-run in the new configuration, so no corrected
> figure is quoted here rather than a modelled one.

### The detour: a tool-result cap, built at the wrong layer

Before testing the window, a per-result size cap was added to the MCP wrapper on the theory that
oversized payloads were the problem. It cost two ~87-minute runs and was reverted.

**First it was inert.** `langchain_mcp_adapters` declares `response_format="content_and_artifact"`,
so tool coroutines return a `(content, artifact)` **tuple**; the guard measured only `str`/`list` and
silently skipped every MCP result — three oversized results, zero cap firings. The unit test fed it a
bare string, validating an assumption about the adapter instead of the adapter's contract. *A test
written from the same misunderstanding as the code cannot catch that misunderstanding.*

**Then it was mis-sized.** At 60,000 chars (~15,000 tokens) one permitted result consumed ~63% of the
23,768-token working budget — still allowing a truncation, while turning a question the baseline had
answered in **3 calls / 141 s** into a **65-minute retry loop**. The model behaved correctly
throughout: it never reissued an identical call until the fourth firing, decomposed the changelog by
action type, and discovered that `limit` belongs *inside* `filters`. The guidance was the problem.

The reusable lesson is about ordering, not about caps: **test the cheap hypothesis first.** Raising
the context window took 20 seconds to verify and made the entire cap exercise unnecessary. A
per-result cap, if ever needed, must be sized as *context budget ÷ expected call count* — and even
then it treats the symptom.

## Tuning: one flag worth 3.8×, and one that actively hurts

With the context ceiling gone, the remaining question was whether latency could be improved at all.
Four configurations, measured on a 9,093-token agent-shaped prompt (NetBox records followed by a
request to tabulate them), one server at a time, baseline re-measured in the same session:

| config | prefill tok/s | decode tok/s | wall | drafts accepted |
|---|---|---|---|---|
| baseline | 34.0 | 5.0 | 318.7 s | — |
| **`-b 2048 -ub 2048`** | **129.7** | 5.9 | **113.7 s** | — |
| `--spec-type ngram-simple` | 36.9 | 3.8 | 314.4 s | **0 / 163** |
| both | 101.4 | 3.7 | 159.7 s | 0 / 147 |

**`-ub 2048` is worth 3.8× on prefill and 2.8× on wall clock.** llama.cpp only copies CPU-resident
expert weights to the GPU once a microbatch is large enough to amortise the PCIe transfer. The
default `-ub 512` never reaches that threshold, so every microbatch pays full freight. With 112 GB
of experts in system RAM, that single threshold dominates prefill — and prefill is where this agent
lives: **187,272 prompt tokens against 21,825 generated** in the 12-question run.

**Measured on a second full 12-question run**, not modelled: **155.1 → 126.3 min (1.23×)**, at
identical correctness (11/12, zero context failures) and 77 tool calls against 88. Extrapolated to
the full v5 set: **19.4 h → ~15.8 h**.

The first projection said ~105 min and ~12.6 h. It was **optimistic by about 20%**, and the reason
is instructive: it applied the flag's isolated 3.8× prefill gain to *all* 187,272 prompt tokens,
when ~85% of those were already served from the prefix cache (see below). Only the genuinely-new
remainder and each question's cold first turn get faster. A 3.8× gain on an isolated measurement
became **1.23× end to end** — and decode, which this flag does not touch, now accounts for roughly
two thirds of what is left.

### The n-gram dead end, recorded so nobody repeats it

`--spec-type ngram-simple` looks made for this workload. The agent constantly echoes device names,
IDs and JSON keys straight back out of tool results, and n-gram drafting is exactly the technique
for repetitive output. It needs no draft model and costs no VRAM.

It produced **163 drafts and accepted zero**, costing 24% of decode because the model pays
verification for drafts it always rejects. Combined with `-ub 2048` it degrades *both* axes
(prefill 129.7 → 101.4, decode 5.9 → 3.7).

The reason is mechanical: `ngram-simple` needs an exactly-repeating 12-token run to predict the next
48, and a table of *distinct* device names has almost none. A 0% acceptance rate is not a tuning
problem — no `--spec-ngram-*-size-n` value fixes it. Draft-model speculation is separately
impossible: `common/speculative.cpp` throws on vocab mismatch and no qwen4exp-vocab draft model
exists (the MTP head PRs are still unmerged).

### Two negatives worth as much as the win

**Prompt caching was already doing its job.** The obvious hypothesis — that a ~9k system prompt plus
tool schemas was being re-read every turn — is false. Median LCP similarity **0.956** across 75 slot
selections, 75 of 76 requests served warm, mean context 16,405 tokens of which only **2,432** were
actually prefilled. **~85% of every prompt came from cache.** The remaining prefill is genuinely new
tool-result text, which no caching strategy can remove. That lever was spent before it was pulled.

**MoE CPU decode runs at ~19% of memory bandwidth and there is no available fix.** Measured 19–20
GB/s effective against a 101.5 GB/s STREAM triad. The matched remedy is NUMA weight mirroring —
duplicate weights per node, which 376 GB comfortably holds — and `GGML_NUMA_STRATEGY_MIRROR` exists
in ggml's enum **and nowhere else in the source**: reserved, unimplemented, unreachable. Switching
runtime does not help either: KTransformers requires Ampere+ and its headline numbers need AMX this
Cascade Lake host lacks; vLLM and SGLang cannot host 112 GB on 21 GiB of VRAM at all.

## Two things recorded because they were wrong

**The context failure was the operator's, not the model's.** The advanced question first died with
`request (18181 tokens) exceeds the available context size (16384 tokens)`. The 16k limit was chosen
deliberately — quantized KV could not be confirmed safe on this hybrid Gated-DeltaNet architecture,
so f16 KV and a smaller window were the conservative choice. At `-c 32768` the same question ran to
28,106 tokens and answered correctly. ~~**≥32k is a requirement for advanced items**, not a
preference.~~ **WITHDRAWN — understated:** 32k still lost 3 of 12 questions to context. The measured
requirement is **≥128k** ~~via `--no-kv-offload`~~ — and it needs no offload: 131k of KV is only
**3.00 GiB** on this hybrid architecture and lives in VRAM (see above).

**The predicted tool-calling fragility did not appear — and this test could not have detected it.**
Third-party reports describe repetition loops, malformed `<tool_call>` emission and spurious
mid-task termination in long agent loops. Across 17 logged turns: none of it, and zero truncation.
But the reported rate is ~0.7% per tool turn, so **24 tool calls yields ~0.17 expected failures**.
Seeing zero is exactly what you would observe whether or not the problem is real. The earlier claim
that this fragility *disqualified* the model was too strong and is withdrawn — it is not refuted.

## Where this leaves routing

The premise has moved. It is no longer "local models cannot do this" — this one did, including the
hard item. It is now a cost question with a measurable shape:

- **Cheap locally**: single-anchor lookups, 2 tool calls, ~3–4 minutes.
- **Expensive locally**: multi-hop traces, 13–20 tool calls, 17–31 minutes.
- **Impractical locally**: the full 90-question harness — **~19.4 h** per run at `-c 131072
  --no-kv-offload`, or ~10.3 h at 32k where 1 question in 4 dies of context. The cheaper
  configuration does not finish the work, so only the first figure is honest. Adding
  **`-b 2048 -ub 2048`** brings it to **~15.8 h, measured** — worth having, and still a long way
  from interactive.

Note the direction of that trade. Fixing the context ceiling made the agent **slower and more
expensive** — 60 → 88 tool calls, 82 → 155 minutes — precisely because it stopped hitting a wall and
started doing the work. A cost measured on a configuration that silently drops a quarter of its
questions is not a cost worth quoting.

That maps cleanly onto the anchor-count rule already used for GraphQL-vs-MCP routing: *count the
anchor objects.* A handoff policy keyed on the same axis — local for one named object, escalate for
set-anchored multi-hop work — now has direct supporting evidence rather than an untested premise.

**See also**: [ADR-0038 — Local Frontier Model Viable](../../adr/0038-local-frontier-model-viable-cost-not-capability.md) ·
[ADR-0037 — Stratified v5 Benchmark](../../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md) ·
[Stratified Benchmark v5](stratified-benchmark-v5.md) ·
[Model-Matrix Benchmarking](../../methods/benchmarking.md)
