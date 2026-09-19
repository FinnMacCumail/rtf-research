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

Extrapolated to the full v5 set (30/30/30): **~12 hours per run**, ~36 hours at the 3× replication
standard this project uses for aggregate rankings.

## Two things recorded because they were wrong

**The context failure was the operator's, not the model's.** The advanced question first died with
`request (18181 tokens) exceeds the available context size (16384 tokens)`. The 16k limit was chosen
deliberately — quantized KV could not be confirmed safe on this hybrid Gated-DeltaNet architecture,
so f16 KV and a smaller window were the conservative choice. At `-c 32768` the same question ran to
28,106 tokens and answered correctly. **≥32k is a requirement for advanced items**, not a preference.

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
- **Expensive locally**: multi-hop traces, 20 tool calls, ~17 minutes.
- **Impractical locally**: the full 90-question harness at ~12 hours per run.

That maps cleanly onto the anchor-count rule already used for GraphQL-vs-MCP routing: *count the
anchor objects.* A handoff policy keyed on the same axis — local for one named object, escalate for
set-anchored multi-hop work — now has direct supporting evidence rather than an untested premise.

**See also**: [ADR-0038 — Local Frontier Model Viable](../../adr/0038-local-frontier-model-viable-cost-not-capability.md) ·
[ADR-0037 — Stratified v5 Benchmark](../../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md) ·
[Stratified Benchmark v5](stratified-benchmark-v5.md) ·
[Model-Matrix Benchmarking](../../methods/benchmarking.md)
