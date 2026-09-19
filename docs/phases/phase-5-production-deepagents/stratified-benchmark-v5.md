# Stratified Benchmark v5 — Discriminating Power and Its Limits

The Phase 5 evaluation harness had been running on a **six-question** dataset (`netbox-benchmark-v4`).
Six questions is enough to catch a hallucination class or validate an A/B on a per-question basis, but
not enough to answer the question Phase 5 was building toward: **does query difficulty predict which
model should handle a query** — the empirical basis for model-handoff routing.

This page covers building a 90-question stratified replacement, the negative result it appeared to
produce across three similar models, and the fourth model that overturned that reading.

## Why six questions was not enough

At n=6 the 95% confidence interval on a pass rate is roughly **±0.32–0.40** — wide enough to swallow
any plausible routing effect. Worse, the set had no difficulty structure: there was no way to ask
"does this model fail specifically on the hard queries?" because *hard* was not a defined axis.

A sizing analysis put the target at **90–150 questions, 30–50 per difficulty tier**. At 30 per tier
the standard error on a tier's pass rate is ≈9pp, so only effects of roughly 15–20pp are detectable —
a real limitation, stated up front rather than discovered later.

## Difficulty defined by mechanism, not topic

The central design rule: **difficulty is a property of the retrieval mechanism, never of the subject
matter.**

| Tier | Mechanism | Tool-call budget |
|---|---|---|
| simple | single object, one filter, no aggregation | ≤3 |
| medium | one join/hop or one aggregation, two filters | ≤7 |
| advanced | ≥2 hops, aggregation + filtering, or absence reasoning | ≤12 |

If "advanced" had meant "questions about circuits" rather than "questions needing two hops", the
benchmark would have measured topic familiarity and called it difficulty.

The set is stratified on **four axes simultaneously** — tier × answer type × domain × data island —
and the binding constraint is that **every tenant must appear in every tier**. Otherwise tenant name
becomes a proxy for difficulty and the routing signal is an artefact. The final set achieves exactly
19/19/19 items for the enriched tenant and 11/11/11 for the demo data across the three tiers.

Final composition, hitting the planned targets exactly: count 21, list 21, value 18, boolean 12,
explanation 9, absence 9.

## Authoring rules the data forced

Most of these were discovered by being wrong first.

**Rule A1 — every expected entity must be a literal substring of its own reference answer.**
Auditing the existing v4 set found **four of its six examples violated this**: a *perfect* answer
could not score 1.0, and one (`site-comparison`) self-scored **0.500**. Some previously-quoted v4
entity-coverage numbers were therefore depressed by an authoring bug, not by model behaviour. A
validator now makes this unrepresentable.

**Prefer identifiers over count phrases.** A `"<number> <noun>"` entity breaks whenever a correct
answer inserts a qualifier — `"13 sites"` missed *"13 **Dunder-Mifflin** sites"*, `"6 prefixes"`
missed *"6 **IP** prefixes"*. Rule A1 proves a perfect *reference* scores 1.0; it never proves a
correct *answer* does. Device names, serials and prefixes survive; synthesised counts often don't.

**Absence items cannot carry entities at all.** The same model phrased the same "none" answer two
ways across two runs — *"0 of 39 devices"* then *"None of the devices… Every one of the 39"* — and no
substring matches both. An entity on an absence item measures phrasing luck. Those nine items are
scored by the reference-grounded correctness judge alone.

**Pin the scope or the question is unanswerable.** "NC State University has *how many* devices?" has
two defensible answers — 19 by tenant, 20 by site — because one unnamed device carries no tenant.
Ambiguity of this kind becomes a deliberate **contrast pair** (two items differing only in the
filter), which is the sharpest available test of whether the agent applied it, or it becomes a bug.

**No bare superlatives without a verified unique maximum.** "Which provider supplies the most
circuits?" turned out to be a **tie** (13/13) and was rewritten as a list.

## Harness changes required first

Two were prerequisites, not niceties:

- **The dataset loader was create-only.** `ensure_dataset_v3`/`v4` return early if the dataset name
  exists, so editing the Python file had *no effect*. Iterating on 90 items needs a true
  **add/update/delete reconcile**, keyed on the question string.
- **A feedback-fetch race.** The results reader returned as soon as three evaluator scores landed —
  written before the fourth (reference-grounded correctness) existed, so it could silently drop the
  **primary metric**. The expected count is now derived from the evaluator list.

Per-tier tool-call budgets were added too: a single global threshold penalises the advanced tier by
construction, since it has a higher floor by design.

## Results — three model families, 270 runs, zero errors

| tier | deepseek-v4-flash | deepseek-v4-pro | kimi-k2.6 |
|---|---|---|---|
| simple | 0.967 | 0.983 | 0.933 |
| medium | 0.950 | 0.967 | 0.900 |
| advanced | 0.800 | 0.850 | 0.900 |
| **overall** | **0.906** | **0.933** | **0.911** |

Paired on identical questions (roughly a third less variance than two independent means):

| pair | difference | 95% CI | |
|---|---|---|---|
| flash − pro | −0.028 | [−0.088, +0.033] | not significant |
| flash − kimi | −0.006 | [−0.079, +0.068] | not significant |
| pro − kimi | +0.022 | [−0.033, +0.078] | not significant |

**Every pairwise difference includes zero.** Three unrelated model families land inside a 2.7pp band.

At that point the conclusion drawn was: the set is ~77% saturated (69 of 90 items solved by all
three) and therefore cannot discriminate capability. **That conclusion was wrong, and a fourth model
disproved it.**

## The fourth model — and the correction

**qwen3.5:397b-cloud**, a family distant from both deepseek and kimi, produced the **first
significant results the benchmark has generated**:

| tier | flash | pro | kimi | **qwen3.5:397b** |
|---|---|---|---|---|
| simple | 0.967 | 0.983 | 0.933 | **0.900** |
| medium | 0.950 | 0.967 | 0.900 | **0.883** |
| advanced | 0.800 | 0.850 | 0.900 | **0.667** |
| **overall** | 0.906 | 0.933 | 0.911 | **0.817** |

| pair | difference | 95% CI | |
|---|---|---|---|
| **pro − qwen** | **+0.117** | [+0.038, +0.196] | **SIGNIFICANT** |
| **kimi − qwen** | **+0.094** | [+0.016, +0.173] | **SIGNIFICANT** |
| flash − qwen | +0.089 | [−0.007, +0.185] | just short |

**Saturation turned out to be a property of the models tested, not of the questions.** One additional
family moved it from **69/90 (77%) to 59/90 (66%)**, and the items defeating everything from 3 to 2.
Ten questions that looked inert across three similar models became discriminating immediately:
`vcpu-aggregation`, `ip-mask-mismatch`, `orphaned-cable`, `prefixes-without-vlan`,
`tenant-group-size`, `vms-without-primary-ip` and four more.

**The corrected claim:** the set resolves capability gaps of roughly **9pp and above** on 90 paired
items, and cannot resolve the ~3pp separating flash, pro and kimi. That is a benchmark behaving
correctly at its sample size — not a ceiling.

The reference corrections make this *stronger*: they lifted the other three models (flash 0.911,
kimi 0.928, pro 0.944) while qwen was judged against the corrected wording throughout.

## What the set does measure

**Difficulty**, convincingly — the tier ordering is monotonic for both deepseek models, and the hard
core is genuinely hard.

**Efficiency**, which is the one axis that separates models cleanly:

| model | tool calls (mean) | within budget | answer length (median) |
|---|---|---|---|
| deepseek-v4-pro | 4.93 | 76/90 | 542 chars |
| kimi-k2.6 | 5.76 | 69/90 | 478 chars |
| deepseek-v4-flash | 6.27 | 66/90 | 758 chars |

For model-handoff routing this is the usable signal: **pro reaches the same answers with ~21% fewer
round-trips**. Correctness will not tell you which model to route to; cost per query will.

**Specific capability gaps.** Three items defeat every model tested, led by an instance-wide
tenant-null aggregation where all three report the *unfiltered* total (141 devices, or 52) instead of
the 15 that genuinely carry no tenant. That is a reproducible, family-independent failure — the kind
of finding a six-question set cannot surface.

## A metric caveat worth carrying forward

**`entity_coverage` is confounded with verbosity.** The most correct model scored *lowest* on it
(0.831 vs 0.885) while writing answers **71% the length** of its rival's. Substring coverage rewards
saying more, not being right. It remains useful within a model, and misleading across models.

## Honest limitations

- **The advanced tier ordering is a property of the model, not only of the set** — it held for both
  deepseek models and did not for kimi, whose advanced tier equals its medium. On 3 wins / 2 losses /
  25 ties that difference is noise, not a capability claim.
- **At 30 items per tier, only ~15–20pp effects are detectable.** Questions sharing a tenant are a
  cluster, which can inflate standard errors further.
- **The closed-book baseline is self-judged** — the tool-less model and the judge were the same
  family. It still showed a large discrimination gap (0.033 entity coverage vs 1.000 with tools), but
  the bias is real and unmeasured.

## Where this goes next

**An earlier version of this page said "harder items, not more models" and that further runs were
"uninformative by construction." The fourth model disproved both.** Testing a wider spread of models
is what revealed the discriminating power the set already had — ten items' worth.

Both levers are real, and they do different jobs:

- **A wider model spread** exposes discrimination that already exists. The marginal value of a fifth
  model is highest if it is *unlike* the four already run.
- **Harder items** raise the ceiling for models that are genuinely close. The two survivors that
  defeat every family — `tenantless-instance-wide` and `deletion-cascade` — show the shape that
  works: multi-hop aggregation and cascade reasoning where the API offers no direct path.

The measurable limit is sample size, not the questions: at 90 paired items the set resolves about
**9pp**, so separating models 3pp apart would need roughly an order of magnitude more questions —
which is a poor trade against simply testing more varied models.

**See also**: [Model-Matrix Benchmarking](../../methods/benchmarking.md) ·
[Evaluating for Correctness](evaluation-correctness.md) ·
[GraphQL Read Path](graphql-read-path.md) ·
[ADR-0037 — Stratified v5 Benchmark](../../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md)
