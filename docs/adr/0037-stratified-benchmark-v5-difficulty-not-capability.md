# ADR-0037 — Stratified 90-Question Benchmark (v5): Discriminating Power and Its Limits

## Status

**Accepted, then REVISED (September 2026).** The original conclusion — that the set measures
difficulty and efficiency but *not* capability — was drawn from three models and is **wrong as
stated**. A fourth, more distant model family overturned it. The original reasoning is preserved
below so the record shows what was believed and on what evidence; the correction follows
immediately.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Extends**: [ADR-0030](0030-model-matrix-evaluation-harness.md) (model-matrix harness),
[ADR-0033](0033-reference-grounded-correctness-evaluator.md) (correctness evaluator)

## Correction (supersedes the conclusion below)

A fourth run — **qwen3.5:397b-cloud**, a family distant from both deepseek and kimi — produced the
**first statistically significant results the benchmark has generated**:

| tier | flash | pro | kimi | **qwen3.5:397b** |
|---|---|---|---|---|
| simple | 0.967 | 0.983 | 0.933 | **0.900** |
| medium | 0.950 | 0.967 | 0.900 | **0.883** |
| advanced | 0.800 | 0.850 | 0.900 | **0.667** |
| **overall** | 0.906 | 0.933 | 0.911 | **0.817** |

Paired on identical questions: **pro − qwen +0.117 [+0.038, +0.196] SIGNIFICANT**;
**kimi − qwen +0.094 [+0.016, +0.173] SIGNIFICANT**; flash − qwen +0.089 [−0.007, +0.185], just
short. The three mutual comparisons between flash, pro and kimi remain non-significant.

**Saturation is a property of the models tested, not only of the questions.** Adding one unrelated
family moved it from **69/90 (77%) to 59/90 (66%)**, and the items defeating every model from 3 to 2.
Ten items that carried no information across three similar models became discriminating the moment a
genuinely different one was tested: `vcpu-aggregation`, `ip-mask-mismatch`, `orphaned-cable`,
`prefixes-without-vlan`, `tenant-group-size`, `vms-without-primary-ip`, `device-update-share`,
`empty-cloud-clusters`, `no-primary-ip-active`, `tenants-without-sites`.

**The corrected claim:** the set resolves capability differences of roughly **9pp and above** on 90
paired items. It cannot resolve the ~3pp separating flash, pro and kimi. The original "ceiling" was
an artefact of testing three closely-matched models — not a limit of the questions.

**Two process errors worth recording**, since both are reusable lessons:
1. Generalising "cannot discriminate" from **n=3 models**, all of comparable strength.
2. Asserting that further runs were *"uninformative by construction."* That prediction was stated
   with unwarranted confidence and was falsified by the very next run.

The reference fixes ([three wording defects](../methods/benchmarking.md)) **strengthen** this result
rather than explaining it away: correcting them raised the other three models (flash 0.911, kimi
0.928, pro 0.944) while qwen was judged against the corrected wording throughout, widening the gap.

## Context

The Phase 5 harness ran on **six** questions (`netbox-benchmark-v4`). At n=6 the 95% CI on a pass
rate is roughly **±0.32–0.40**, and the set had no difficulty structure at all.

The next planned step — **model-handoff routing** (route simple queries to a cheap model, hard ones
to a strong one) — rests on an empirical premise that had never been tested: *that query difficulty
predicts which model should handle a query.* Testing it needs a dataset that (a) is large enough to
resolve a routing-sized effect and (b) has difficulty as an explicit, principled axis.

The demo NetBox instance was also too thin to support 90 distinct questions, so an **additive tenant**
was seeded alongside it (69 devices, 42 prefixes, 31 VLANs, 14 circuits, 28 VMs, plus deliberate
data defects), leaving the original demo data untouched as a second "island".

## Decision

**Build a 90-question benchmark stratified on four axes simultaneously — tier × answer type × domain
× data island — with difficulty defined by retrieval mechanism, and accept the measurement it
produces even if negative.**

- **Difficulty is mechanism, never topic**: simple = one object/one filter (≤3 tool calls); medium =
  one join or one aggregation (≤7); advanced = ≥2 hops, aggregation + filtering, or absence reasoning
  (≤12). Had "advanced" meant "circuits questions", the set would measure topic familiarity.
- **Every tenant appears in every tier** (final: 19/19/19 and 11/11/11). Otherwise tenant name proxies
  for difficulty and any routing signal is an artefact.
- **Ground truth computed live** against a pinned snapshot using the agent's own **read-only** token —
  parity, so a question is never unanswerable for the wrong reason.
- **Two harness defects fixed first**: the dataset loader was *create-only* (editing the source had no
  effect once the dataset existed), and the results reader could return before the primary
  correctness score landed.

## Evidence *(as originally recorded — see the Correction above)*

Three model families, **270 runs across three complete 90-item experiments, zero errors**:

| tier | deepseek-v4-flash | deepseek-v4-pro | kimi-k2.6 |
|---|---|---|---|
| simple | 0.967 | 0.983 | 0.933 |
| medium | 0.950 | 0.967 | 0.900 |
| advanced | 0.800 | 0.850 | 0.900 |
| **overall** | **0.906** | **0.933** | **0.911** |

Paired on identical questions — every pairwise difference **includes zero**:
flash − pro **−0.028** [−0.088, +0.033]; flash − kimi **−0.006** [−0.079, +0.068];
pro − kimi **+0.022** [−0.033, +0.078].

**The decisive statistic is saturation, not the means: 69 of 90 items are solved by all three models,
and only 3 defeat all three.** A ~77%-saturated set cannot discriminate capability — the solved items
carry no information.

Efficiency, by contrast, separates cleanly: pro 4.93 tool calls (76/90 within budget), kimi 5.76
(69/90), flash 6.27 (66/90).

## Consequences

### Positive
- **Difficulty measurement works** — monotonic tier ordering for both deepseek models, and a
  genuinely hard core that survives every model tested.
- **Efficiency is the usable routing signal.** Correctness will not tell you which model to route to;
  **cost per query will** — pro reaches the same answers with ~21% fewer round-trips.
- **Three reproducible, family-independent capability gaps**, led by an instance-wide tenant-null
  aggregation where all three models report the *unfiltered* total instead of the filtered one.
- **Reusable authoring rules** (see [Benchmarking](../methods/benchmarking.md)), the most important
  being that an expected entity must be a literal substring of its own reference answer — an audit
  found **four of the six v4 examples violated this**, so a perfect answer could not score 1.0.
- Harness hardening: a true add/update/delete dataset sync, per-tier tool-call budgets, and a
  feedback-count fix that had been able to drop the primary metric.

### Negative / limitations
- <del>**The set ceilings at ~0.91–0.93.** Further model runs against it are known in advance to be
  uninformative; raising the ceiling requires *harder items*, not more models.</del>
  **WITHDRAWN.** The fourth model run falsified this directly (see the Correction above). The set
  resolves ~9pp gaps; it could not resolve the ~3pp between three closely-matched models. What
  raises discriminating power is **a wider spread of models**, and harder items *in addition* —
  not instead.
- **`entity_coverage` is confounded with verbosity** — the most correct model scored *lowest* on it
  while writing answers 71% the length of its rival's. Useful within a model, misleading across
  models.
- **At 30 items per tier only ~15–20pp effects are detectable**, and questions sharing a tenant form
  clusters that can inflate standard errors further.
- **The closed-book baseline is self-judged** (tool-less model and judge from the same family). It
  still showed a large gap, but the bias is real and unquantified.
- Tier ordering proved to be **partly a property of the model**: it held for both deepseek models and
  not for kimi. On 3 wins / 2 losses / 25 ties that difference is noise, not a capability claim.

## Relationship to ADR-0030 and ADR-0033

ADR-0030 established the model-matrix harness and ADR-0033 added the reference-grounded correctness
axis. This ADR scales the *dataset* behind both and, in doing so, finds the harness's current
practical limit: with competent models the discriminating power now lies in **efficiency and specific
failure modes**, not in aggregate correctness.

## References

- [Phase 5 → Stratified Benchmark v5](../phases/phase-5-production-deepagents/stratified-benchmark-v5.md)
- [ADR-0038 — Local Frontier Model: Cost, Not Capability](0038-local-frontier-model-viable-cost-not-capability.md) — the first local evidence bearing on the model-handoff routing premise named in *Context* above
- [Research Methods → Benchmarking](../methods/benchmarking.md)
- [ADR-0030 — Model-Matrix Evaluation Harness](0030-model-matrix-evaluation-harness.md)
- [ADR-0033 — Reference-Grounded Correctness Evaluator](0033-reference-grounded-correctness-evaluator.md)
- Repository: `tests/eval/dataset_v5.py`, `tests/eval/run_matrix_v5.py`,
  `docs/development/2026-09-16_v5-question-authoring-plan.md`,
  `docs/traces/2026-09-17_netbox-benchmark-v5_v5-2model.md`,
  `docs/traces/2026-09-17_netbox-benchmark-v5_v5-distant.md`
