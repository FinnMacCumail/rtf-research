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
the win landing on the cross-domain query class (0.5 → 1.0 on the IP-allocation trap) at a
tool-call cost — see [GraphQL Read Path](../phases/phase-5-production-deepagents/graphql-read-path.md).

**See also**: [Evaluation](evaluation.md) · [Observability](observability.md) ·
[Phase 5 → Evaluating for Correctness](../phases/phase-5-production-deepagents/evaluation-correctness.md) ·
[ADR-0030 — Model-Matrix Evaluation Harness](../adr/0030-model-matrix-evaluation-harness.md) ·
[ADR-0033 — Reference-Grounded Correctness Evaluator](../adr/0033-reference-grounded-correctness-evaluator.md)
