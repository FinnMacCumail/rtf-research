# Model-Matrix Benchmarking

Methodology for systematically comparing many models on the same agent task. Introduced in
[Phase 5](../phases/phase-5-production-deepagents/multi-model-evaluation.md); this page is the
reusable design.

## Goal

Replace ad-hoc "run a query, eyeball the trace, write a report" comparison with a **reproducible,
automatically-scored matrix**: N models × a fixed dataset × a fixed set of evaluators, producing a
sortable leaderboard. The comparison surface becomes "capability and cost per query class," not
"which framework feels better."

## Components

### 1. A fixed dataset with ground truth

A small, curated set of representative queries (four NetBox query classes in Phase 5), each with a
machine-checkable reference. The reference used here is **`expected_entities`** — the concrete
facts a correct answer must mention — rather than a full reference answer, because:

- Full reference answers churn as the underlying data changes; entity lists are stable.
- Entity coverage is deterministic and cheap to compute.

**Ground-truth tightness matters.** An early dataset version was too permissive — a model could
score 1.0 by naming 3 of 14 required sites. Anchoring `expected_entities` to *full*, trace-derived
ground truth (`v2`) was what made the leaderboard discriminate between strong and weak models.

**Keep a full `reference_answer` too — and actually score against it.** Later dataset versions
(`v3`/`v4`, cross-domain queries) also carry a prose `reference_answer` per example. In `v3` this
field existed but *no evaluator read it*, so a hallucinated figure went uncaught; `v4` adds the
reference-grounded `correctness_judge` (below) that consumes it. `expected_entities` (cheap, stable)
and `reference_answer` (fact-checkable) are complementary, not redundant.

### 2. Evaluators (four complementary signals)

| Evaluator | Type | What it measures |
|---|---|---|
| `entity_coverage` | Code-based, deterministic | Fraction of `expected_entities` present in the answer |
| `completeness_judge` | LLM-as-judge | Whether the answer fully addresses the query (catches partial answers that omit expected aspects) |
| `tool_call_efficiency` | Trajectory metric | Tool calls per query — surfaces retry loops and over-exploration |
| `correctness_judge` | LLM-as-judge, **reference-grounded** | Whether the answer's factual claims *agree with the reference answer* — flags contradictions (counts, percentages, allocations) |

Using all four is deliberate: an efficiency-blind score misses pathologies (e.g. a model that
reaches the right answer via 18 tool calls because its malformed argument shape keeps getting
rejected), and a deterministic-only score misses semantic incompleteness.

**Completeness is not correctness — and the difference is where hallucination hides.** The first
three evaluators never see the *correct* answer: `entity_coverage` matches positive keywords, and
`completeness_judge` is prompted with the question, the answer, and `expected_entities` — not the
`reference_answer`. A confident, specific, on-topic-but-*false* value therefore scores high on
completeness. In Phase 5 one answer scored **0.9 completeness** while hallucinating the exact figure
the question asked for (reporting "~7.7% IP utilization" where the truth was 0%). The
`correctness_judge` closes this gap: it is fed the `reference_answer` and scores **contradiction**,
not coverage. Re-scoring an existing 9-model sweep with it surfaced **9 hallucinations completeness
had rated complete**, and reordered the leaderboard (the more conservative model, penalised by
completeness on negative findings, is the more *correct* one). A reference-grounded correctness axis
is the evaluator that actually serves an anti-hallucination goal. See
[ADR-0033](../adr/0033-reference-grounded-correctness-evaluator.md).

Two caveats travel with it: (1) a correctness judge is only as fair as its reference — an ambiguous
reference produces *false* contradictions, so reference wording must be tightened as errors are
found; (2) LLM-judge correctness on a small dataset is noisy across runs — the per-question signal is
robust, but an aggregate ranking needs ≥3 runs to exceed single-run variance.

### 3. A runner

One experiment per model. Practical concerns the runner handles:

- **Model-agnostic targets** — model identity flows through config, so local, cloud, and reference
  models all run through the same harness without code changes.
- **Skip-completed** — re-runs only score models without an existing complete experiment.
- **Fail-fast on quota** — cloud-inference session limits surface as errors, not bad scores; the
  runner aborts cleanly rather than polluting the leaderboard with zeros, and a force-rerun flag
  exists for deliberate regression checks against a baseline.

## Reading the leaderboard

Rank primarily on quality (`entity_coverage`, `completeness`), then use `tool_call_efficiency` and
latency to separate models of similar quality. Watch for:

- **Quality ties broken by efficiency** — two models at 1.00 completeness can differ 2–4× in tool
  calls and latency.
- **Concise-but-wrong** — a low entity/completeness with *normal* tool counts indicates the model
  is confidently wrong, not merely under-exploring.
- **Protocol/capability failures** — all-errored or no-tool-support outcomes are categorical, not
  quality, results (e.g. a reasoning-only model that can't tool-call, or a model whose tool-call
  serialization is incompatible).

## Reuse and extension

The same harness extends to: comparing a framework upgrade (run the *same* model before/after),
A/B-testing a prompt or middleware change, and adding new models as they appear. It is the gate
referenced throughout [Phase 5](../phases/phase-5-production-deepagents/overview.md) and the
detection half of the [observability loop](observability.md).

**Worked A/B example**: the read-only GraphQL routing change was validated exactly this way — the
*same* models on the *same* `v4` dataset, once with the GraphQL tool available (`graphql` variant)
and once without (`baseline`), yielding distinct comparable experiments. The correctness axis showed
the win landing on the cross-domain query class (0.5 → 1.0 on the IP-allocation trap) — see
[GraphQL Read Path](../phases/phase-5-production-deepagents/graphql-read-path.md).

**The same harness gated a framework upgrade.** Re-running the identical A/B after the DeepAgents
0.7.5 bump ([0.7.5 Upgrade](../phases/phase-5-production-deepagents/0-7-5-upgrade.md)) showed the
GraphQL path's earlier ~2× tool-call cost had *vanished* — demonstrating that a measured "cost" can be
an artifact of the layer around the model, exposed only by running the *same* eval before and after.

**Replication is part of the method, not an afterthought.** Single runs on a 6-question set are noisy
(±0.15–0.25; two queries swing 0.0/0.5/1.0 with no code change). A single-run A/B with an open caveat
is a *hypothesis*: the GraphQL result only became quotable after **3× replication** (per-arm), which
averaged out the noisy queries, confirmed the routing fix held (device-detail 3/3 on one model, 2/3
on the other, trajectory-verified as MCP-routed), and stabilized the aggregate (≈0.82). Rule of thumb:
**per-question wins that repeat every run are trustworthy on one run; an aggregate ranking needs ≥3.**

## Dataset scale, stratification, and saturation

A six-question set can validate a per-question A/B; it cannot rank models. At n=6 the 95% CI on a
pass rate is ≈**±0.32–0.40**. Scaling to **90 questions stratified on four axes** (difficulty tier ×
answer type × domain × data partition) produced the reusable rules below — and one result worth
recording in its own right.

**Define difficulty by mechanism, never by topic.** simple = one object, one filter; medium = one
join or one aggregation; advanced = ≥2 hops or absence reasoning. If "advanced" means "questions
about circuits", the benchmark measures topic familiarity and calls it difficulty.

**Every group must appear in every tier.** If one tenant (or customer, or partition) appears only in
the hard tier, its *name* becomes a proxy for difficulty and any routing signal is an artefact. Check
the cross-tabulation explicitly rather than assuming.

**Budget tool calls per tier.** A single global efficiency threshold penalises the hard tier by
construction, since it has a higher floor by design. Calibrate budgets from measured percentiles —
ours were revised twice from data (simple ≤2→3, medium ≤4→7).

### Saturation is the number that matters — and it is not a property of the questions

Headline means hid the real finding. Across three model families the aggregates sat in a 2.7pp band
(0.906 / 0.911 / 0.933) with **every paired CI including zero**, and **69 of 90 items were solved by
all three**. The conclusion drawn was that the set had saturated.

**That conclusion was wrong, and the correction is the more useful lesson.** A fourth, more distant
family moved saturation to **59/90** and produced two significant differences. Ten items that looked
inert across three similar models were discriminating all along — the measurement lacked the model
spread to reveal it.

**A saturated item carries no information *against the models you happened to test*.** Report
solved-by-all and failed-by-all counts alongside the leaderboard — a mean cannot tell you whether a
set still discriminates. But read a high saturation figure as a statement about **your model
sample**, not about your questions, and test that reading with a model unlike the others before
concluding the set is exhausted.

The two levers are distinct: **a wider model spread** exposes discrimination that already exists;
**harder items** raise the ceiling for models that are genuinely close. Predicting that further runs
would be "uninformative by construction" was a forecast made from three closely-matched models, and
the next run falsified it.

**Use paired analysis on identical questions**, reporting the pairwise difference with its CI rather
than two independent means: roughly a third less variance from the same data.

### Authoring rules that survived contact with data

- **Every expected entity must be a literal substring of its own reference answer.** An audit of an
  existing set found **four of six examples violated this** — a *perfect* answer could not score 1.0,
  one self-scoring 0.500 — meaning some previously-quoted coverage numbers reflected an authoring bug,
  not model behaviour. This is a three-line check; run it before scoring anything.
- **Self-match is necessary, not sufficient.** It proves a perfect *reference* scores 1.0; it never
  proves a correct *answer* does. A `"<number> <noun>"` entity breaks when a correct answer inserts a
  qualifier ("13 sites" misses "13 **Dunder-Mifflin** sites"). Prefer identifiers — names, serials,
  addresses — over synthesised counts.
- **Absence items cannot carry entities at all.** The same model phrased the same "none" answer two
  incompatible ways across two runs. An entity on an absence item measures phrasing luck; score those
  with the reference-grounded judge alone.
- **Pin the scope, or the item is unanswerable.** Where a plausible second reading changes the answer,
  author *both* as a contrast pair differing only in the filter — the sharpest test of whether the
  agent applied it.
- **No bare superlatives without a verified unique maximum** (ours turned out to be a tie).

### A metric caveat: coverage is confounded with verbosity

`entity_coverage` rewards saying more. The **most correct** model in the sweep scored **lowest** on it
(0.831 vs 0.885) while writing answers **71% the length** of its rival's. It stays useful *within* a
model and is misleading *across* models — a caution that applies to any substring-coverage metric.

### What survives when correctness saturates

Efficiency. Tool calls separated the same three models cleanly (4.93 / 5.76 / 6.27 mean; 76/69/66 of
90 within budget) where correctness could not. For routing decisions, **cost per query is the signal
that survives a ceiling** — see
[ADR-0037](../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md).

**See also**: [Evaluation](evaluation.md) · [Observability](observability.md) ·
[Phase 5 → Evaluating for Correctness](../phases/phase-5-production-deepagents/evaluation-correctness.md) ·
[Phase 5 → GraphQL Read Path](../phases/phase-5-production-deepagents/graphql-read-path.md) ·
[Phase 5 → Stratified Benchmark v5](../phases/phase-5-production-deepagents/stratified-benchmark-v5.md) ·
[ADR-0030 — Model-Matrix Evaluation Harness](../adr/0030-model-matrix-evaluation-harness.md) ·
[ADR-0033 — Reference-Grounded Correctness Evaluator](../adr/0033-reference-grounded-correctness-evaluator.md) ·
[ADR-0037 — Stratified v5 Benchmark](../adr/0037-stratified-benchmark-v5-difficulty-not-capability.md)
