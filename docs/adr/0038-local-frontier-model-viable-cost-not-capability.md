# ADR-0038 — A Frontier-Class Model Runs Locally: the Constraint Is Cost, Not Capability

## Status

**Accepted (September 2026).** Evidence is an **n=1-per-tier existence proof**, not a benchmark
score. It is recorded as an ADR because it bears directly on a decision the whole phase has been
building toward — model-handoff routing — and because it **partly reverses the premise** that local
models are not capable enough for this agent.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Bears on**: [ADR-0037](0037-stratified-benchmark-v5-difficulty-not-capability.md) (v5 benchmark),
[ADR-0028](0028-native-local-and-cloud-models-on-deepagents.md) (local & cloud models),
[ADR-0035](0035-deepagents-0.7.5-upgrade.md) (upgrade taken partly to unblock routing)

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

### Negative / limitations
- **n=1 per tier. This is not a score.** One advanced question that happened to need 20 calls may not
  be representative; the same caution that produced the ADR-0037 correction applies with more force
  here, not less.
- **The reported tool-call fragility did not reproduce — and this test could not have detected it.**
  Across 17 logged turns there were no repetition loops, no malformed `<tool_call>`, no premature
  stops, no truncation. But at the reported ~0.7%-per-turn rate, **24 tool calls yields ~0.17
  expected failures**. Observing zero is consistent with the problem being real. The claim that
  fragility *disqualifies* this model was stated too strongly and is withdrawn; it is not refuted.
- **≥32k context is required.** The first advanced attempt died at `18181 tokens exceeds the
  available context size (16384)` — a configuration limit chosen because quantized KV could not be
  confirmed safe on this hybrid GDN architecture. At 32k it ran to 28,106 tokens and succeeded. The
  failure was the harness operator's, not the model's.
- **Throughput makes the full v5 harness impractical.** Extrapolating the observed per-tier times to
  30/30/30 gives **~12 hours per run**, and ~36 hours at the established 3× replication standard.
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
