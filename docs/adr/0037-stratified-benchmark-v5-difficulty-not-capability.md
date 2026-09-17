# ADR-0037 — Stratified 90-Question Benchmark (v5): Measures Difficulty and Efficiency, Not Capability

## Status

**Accepted** — built and measured (September 2026). Recorded as a **negative result**: the
capability-discrimination goal was not met, and the reason is now known.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Extends**: [ADR-0030](0030-model-matrix-evaluation-harness.md) (model-matrix harness),
[ADR-0033](0033-reference-grounded-correctness-evaluator.md) (correctness evaluator)

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

## Evidence

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
- **The set ceilings at ~0.91–0.93.** Further model runs against it are known in advance to be
  uninformative; raising the ceiling requires *harder items*, not more models.
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
- [Research Methods → Benchmarking](../methods/benchmarking.md)
- [ADR-0030 — Model-Matrix Evaluation Harness](0030-model-matrix-evaluation-harness.md)
- [ADR-0033 — Reference-Grounded Correctness Evaluator](0033-reference-grounded-correctness-evaluator.md)
- Repository: `tests/eval/dataset_v5.py`, `tests/eval/run_matrix_v5.py`,
  `docs/development/2026-09-16_v5-question-authoring-plan.md`,
  `docs/traces/2026-09-17_netbox-benchmark-v5_v5-2model.md`,
  `docs/traces/2026-09-17_netbox-benchmark-v5_v5-distant.md`
